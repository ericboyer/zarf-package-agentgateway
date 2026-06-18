# UDS Package — agentgateway

A UDS-integrated Zarf package for [agentgateway](https://agentgateway.dev), an AI-native proxy and Kubernetes Gateway API implementation that handles MCP, A2A, LLM, HTTP, and gRPC traffic.

Wraps the upstream Helm charts `oci://cr.agentgateway.dev/charts/agentgateway` and `oci://cr.agentgateway.dev/charts/agentgateway-crds` (both pinned to `v1.2.1`) and adds:

- A UDS `packages.uds.dev` Custom Resource (network policies + tenant/admin gateway expose rules)
- A default `Gateway` resource so the controller actually spins up a data plane
- An `AgentgatewayParameters` resource that rebinds the admin UI from `localhost:15000` to `0.0.0.0:15000` so it's reachable through Istio
- A `Service` for the admin UI port (the controller doesn't create one by itself)

## What gets deployed

Per component:

| Component | Source | Purpose |
|---|---|---|
| `agentgateway-crds` chart | `oci://cr.agentgateway.dev/charts/agentgateway-crds:1.2.1` | `AgentgatewayBackend`, `AgentgatewayParameters`, `AgentgatewayPolicy` CRDs |
| `agentgateway` chart | `oci://cr.agentgateway.dev/charts/agentgateway:1.2.1` | Control plane `Deployment` + `ServiceAccount` + RBAC + control-plane `Service` (`grpc-xds-agw:9978`, `health:9093`, `metrics:9092`) |
| `uds-agentgateway-config` chart | `chart/` (local) | UDS `Package` CR, `Gateway`, `AgentgatewayParameters`, admin-UI `Service` |

All resources land in the `agentgateway` namespace.

Images pinned:

| Image | Used by |
|---|---|
| `cr.agentgateway.dev/controller:v1.2.1` | Control-plane `Deployment` |
| `cr.agentgateway.dev/agentgateway:v1.2.1` | Data-plane proxy pods created dynamically by the controller |

The data-plane image is passed to the controller via `AGW_PROXY_IMAGE_*` env vars (not referenced from any PodSpec), so Zarf will pull and store it but won't auto-rewrite it for airgap. For true airgap deploys, override `image.registry` to point at the internal registry.

## Prerequisites

- A cluster with the standard **Gateway API CRDs** installed (`gateway.networking.k8s.io`). UDS Core's Istio install provides these; vanilla k3d does not.
- UDS Core deployed (the `Package` CR is reconciled by the Pepr operator that ships with Core).

## Layout

```
.
├── README.md
├── zarf.yaml                              # root, flavor-gated, declares images
├── common/zarf.yaml                       # config chart → crds → controller, in order
├── chart/                                 # uds-agentgateway-config Helm chart
│   ├── Chart.yaml
│   ├── values.yaml                        # all user-facing tunables
│   └── templates/
│       ├── uds-package.yaml               # packages.uds.dev (network + expose)
│       ├── gateway.yaml                   # default Gateway + parametersRef
│       ├── agentgateway-parameters.yaml   # adminAddr: 0.0.0.0:15000
│       ├── admin-service.yaml             # ClusterIP for proxy:15000
│       └── jwt-policy.yaml                # AgentgatewayPolicy (Keycloak, optional)
├── values/
│   ├── common-values.yaml                 # shared chart values (monitoring off, etc.)
│   └── upstream-values.yaml               # pinned controller + proxy images
├── bundle/
│   ├── uds-bundle.yaml                    # dev test bundle
│   └── uds-config.yaml
├── tasks.yaml                             # delegates standard ops to uds-common
└── tasks/test.yaml                        # health-check
```

## Build & deploy

```bash
# Build the Zarf package and bundle
uds zarf package create --flavor upstream --skip-sbom --confirm
uds create bundle/ --confirm

# Full setup (k3d + UDS Core + this bundle)
uds run default

# Iterate on an existing cluster that already has UDS Core
uds run dev
```

Available tasks: `default`, `dev`, `test-install`, `test-upgrade`, `create-dev-package`, `create-deploy-test-bundle`.

## Using the gateway

The package gives you a running controller and an empty Gateway. To actually route traffic you apply three things per backend: a `Service` (telling agentgateway what protocol it speaks), an `AgentgatewayBackend` (how the proxy should treat it), and an `HTTPRoute` (which paths get sent there).

### 1. Sanity-check the install

```bash
# Controller up
kubectl get deploy -n agentgateway agentgateway

# Gateway accepted + programmed by the controller
kubectl get gateway -n agentgateway agentgateway-proxy

# Data-plane proxy pods (created lazily by the controller from the Gateway)
kubectl get pods -n agentgateway -l gateway.networking.k8s.io/gateway-name=agentgateway-proxy
```

If the Gateway shows `PROGRAMMED=True` and you see one or more proxy pods, the data plane is live and your expose URLs will resolve.

### 2. Route an MCP server through the gateway

Example: route `https://agentgateway-mcp.uds.dev/mcp` to an MCP server running in the same namespace. The `appProtocol: agentgateway.dev/mcp` annotation on the `Service` is what flips agentgateway into MCP mode for that backend.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-website-fetcher
  namespace: agentgateway
spec:
  selector:
    app: mcp-website-fetcher
  ports:
    - port: 80
      targetPort: 8000
      appProtocol: agentgateway.dev/mcp   # required
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: agentgateway
spec:
  mcp:
    targets:
      - name: mcp-target
        static:
          backendRef:
            name: mcp-website-fetcher
          port: 80
          protocol: SSE        # or StreamableHTTP, stdio, etc.
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp
  namespace: agentgateway
spec:
  parentRefs:
    - name: agentgateway-proxy
      namespace: agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mcp
      backendRefs:
        - name: mcp-backend
          group: agentgateway.dev
          kind: AgentgatewayBackend
```

`parentRefs` must match the `name` + `namespace` of the Gateway resource this chart created (`agentgateway-proxy` / `agentgateway`). If you renamed it via `gateway.name`, update accordingly.

### 3. Connect an MCP client

Use the public hostname from the `mcp` expose rule (default `https://agentgateway-mcp.<domain>`) plus the path you set on the `HTTPRoute`:

```bash
# Inspect with the upstream MCP Inspector tool
npx @modelcontextprotocol/inspector

# Then in the inspector UI:
#   Transport:  Streamable HTTP   (or SSE if your backend uses it)
#   URL:        https://agentgateway-mcp.uds.dev/mcp
```

For Claude Desktop or another MCP client, point its config at the same URL. If `keycloak.enabled=true` you also need a bearer token — see below.

### 4. Route an LLM (optional)

agentgateway can act as an LLM gateway in front of providers (OpenAI, Anthropic, Bedrock, Gemini, etc.). Example for OpenAI:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: openai-secret
  namespace: agentgateway
type: Opaque
stringData:
  Authorization: Bearer sk-...            # your API key
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway
spec:
  ai:
    provider:
      openai:
        model: gpt-4o-mini
  policies:
    auth:
      secretRef:
        name: openai-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway
spec:
  parentRefs:
    - name: agentgateway-proxy
      namespace: agentgateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1/chat/completions
      backendRefs:
        - name: openai
          group: agentgateway.dev
          kind: AgentgatewayBackend
```

Test through the public URL (replace `<token>` if `keycloak.enabled=true`):

```bash
curl https://agentgateway-mcp.uds.dev/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer <token>" \
  -d '{"messages":[{"role":"user","content":"hi"}]}'
```

`agentgateway-mcp.<domain>` is just the hostname for the data-plane HTTP listener — LLM and MCP routes co-exist on the same listener, distinguished by path.

### 5. With `keycloak.enabled=true`: getting a token

The Gateway-level `AgentgatewayPolicy` rejects requests without a valid Keycloak JWT. Two quick ways to get one:

**Client credentials (machine-to-machine)** — using the provisioned `agentgateway-mcp` client:

```bash
# Pull the client secret UDS provisioned
CLIENT_SECRET=$(kubectl get secret -n agentgateway agentgateway-mcp-sso \
  -o jsonpath='{.data.clientSecret}' | base64 -d)

# Get a token
TOKEN=$(curl -s -X POST \
  "https://sso.uds.dev/realms/uds/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=agentgateway-mcp" \
  -d "client_secret=$CLIENT_SECRET" \
  | jq -r .access_token)

# Call the gateway
curl https://agentgateway-mcp.uds.dev/mcp -H "Authorization: Bearer $TOKEN"
```

**User token (interactive testing)** — use the `password` grant on a test user, or grab a token from the browser dev tools after logging into another UDS app. The token's `iss` must equal `https://sso.uds.dev/realms/uds` and its `aud` claim must include one of `keycloak.audiences` (default: `agentgateway-mcp`).

**If you see 401**: most often `aud` doesn't match. Either:
- Decode the token at `jwt.io` and add the actual `aud` value to `keycloak.audiences` in chart values, then redeploy, **or**
- Add an audience-mapper to your calling client in the Keycloak admin UI so its tokens carry `aud: agentgateway-mcp`.

### 6. Inspect the running config (admin UI)

Open `https://agentgateway.<domain>` (default: `https://agentgateway.uds.dev`). The admin UI is read-only in Kubernetes mode — it shows the live listeners, backends, routes, and policies the controller has pushed to the proxy. Useful for confirming an `HTTPRoute` actually attached.

### 7. Add more policies

Beyond JWT auth, attach more `AgentgatewayPolicy` resources to either the Gateway (everything) or an `HTTPRoute` (per-route). Common patterns: API-key auth, rate limiting, CEL-based RBAC, request transformations, LLM guardrails. See the upstream [policy overview](https://agentgateway.dev/docs/kubernetes/latest/about/policies/overview/).

## Configuration reference

All values live in [chart/values.yaml](chart/values.yaml).

| Path | Default | What it does |
|---|---|---|
| `domain` | `###ZARF_VAR_DOMAIN###` | Base domain (set via the `DOMAIN` Zarf variable, default `uds.dev`) |
| `monitoring.enabled` | `false` | Emit `ServiceMonitor` for the controller's `metrics:9092` port |
| `gateway.enabled` | `true` | Whether to render the default `Gateway` resource |
| `gateway.name` | `agentgateway-proxy` | Gateway resource name. Controller creates a `Service` with this name |
| `gateway.className` | `agentgateway` | GatewayClass to attach to |
| `gateway.podLabelKey` | `gateway.networking.k8s.io/gateway-name` | Label the controller stamps on data-plane pods. Used by admin Service + netpol selectors |
| `gateway.http.enabled` | `true` | Render the HTTP listener (the MCP entry point) |
| `gateway.http.name` | `http` | Listener name |
| `gateway.http.port` | `80` | Listener port |
| `gateway.http.allowedRoutesFrom` | `Same` | `Same`/`All`/`Selector` — which namespaces may attach Routes |
| `admin.enabled` | `true` | Create the admin-UI `Service` + expose rule + `AgentgatewayParameters` bind override |
| `admin.host` | `agentgateway` | Hostname (becomes `agentgateway.<domain>`) |
| `admin.port` | `15000` | Port the admin UI listens on inside the proxy pod |
| `mcp.enabled` | `true` | Create the MCP expose rule |
| `mcp.host` | `agentgateway-mcp` | Hostname (becomes `agentgateway-mcp.<domain>`) |
| `mcp.port` | `80` | Service port to route to. Must match a Gateway listener |
| `keycloak.enabled` | `false` | Provision a Keycloak client + attach JWT validation to the data-plane Gateway |
| `keycloak.realm` | `uds` | Keycloak realm (UDS Core default) |
| `keycloak.ssoHost` | `sso` | Keycloak hostname (forms `https://sso.<domain>`) |
| `keycloak.clientId` | `agentgateway-mcp` | Client ID provisioned in Keycloak and default expected `aud` claim |
| `keycloak.secretName` | `agentgateway-mcp-sso` | Secret UDS creates with the client credentials |
| `keycloak.audiences` | `[agentgateway-mcp]` | List of accepted JWT `aud` values |
| `keycloak.service.{name,namespace,port}` | `keycloak / keycloak / 8080` | In-cluster Keycloak Service for JWKS fetches |
| `keycloak.jwksPath` | `/protocol/openid-connect/certs` | JWKS path appended to `/realms/<realm>` |
| `keycloak.mcp.resource` | `""` (defaults to `https://<mcp.host>.<domain>`) | `resource` advertised in OAuth-protected-resource metadata |
| `keycloak.mcp.scopesSupported` | `[openid, email, profile]` | Scopes advertised to MCP clients |
| `keycloak.mcp.bearerMethodsSupported` | `[header]` | Where MCP clients may carry the bearer token |
| `additionalNetworkAllow` | `[]` | Extra `network.allow` entries appended to the Package CR |

## How endpoints are exposed

| Endpoint | URL | Behind | Routes through |
|---|---|---|---|
| Admin UI | `https://agentgateway.<domain>` | UDS **admin** gateway | `agentgateway-admin` Service (created here) → proxy pod port `15000` |
| MCP | `https://agentgateway-mcp.<domain>` | UDS **tenant** gateway | `agentgateway-proxy` Service (auto-created by controller) → proxy pod port `80` |

The MCP route just gets you to the Gateway's HTTP listener. To actually route `/mcp` (or any path) to a real backend, the consumer applies their own `HTTPRoute` / `AgentgatewayPolicy` against the `agentgateway-proxy` Gateway. The agentgateway docs cover this in [Quickstart → MCP servers](https://agentgateway.dev/docs/kubernetes/latest/quickstart/mcp/).

## Architecture notes

agentgateway is split into a control plane and a data plane:

1. The **controller** (this chart's `Deployment`) watches `Gateway`, `HTTPRoute`, and agentgateway CRDs.
2. When you apply a `Gateway` resource referencing the `agentgateway` GatewayClass, the controller **dynamically creates** a `Deployment` + `Service` named after the Gateway.
3. Those proxy pods serve:
   - The HTTP listeners declared on the Gateway (default port `80`)
   - The admin UI on port `15000` (read-only in K8s mode)
4. `AgentgatewayParameters` referenced via `Gateway.spec.infrastructure.parametersRef` customizes the proxy deployment (env vars, raw config, resources, image, etc.).

Because the admin UI isn't a default Service, this chart creates `agentgateway-admin` selecting the proxy pods by their `gateway.networking.k8s.io/gateway-name` label. Because the admin UI binds to `localhost:15000` by default and Istio's ambient mesh delivers to the pod IP, this chart also sets `rawConfig.config.adminAddr: "0.0.0.0:15000"` via `AgentgatewayParameters` — without that, you'd see `503 UC upstream_reset_before_response_started{connection_termination}` in the Istio gateway logs.

## Keycloak / JWT auth on the MCP gateway

Flip `keycloak.enabled=true` (in `chart/values.yaml`, or via a bundle override) to put OIDC enforcement in front of the MCP listener. When on:

- A UDS-managed Keycloak client is provisioned (`clientId: agentgateway-mcp`, M2M / service-accounts client, no redirect URIs). Credentials land in a `Secret` named `agentgateway-mcp-sso` in the `agentgateway` namespace — available to any caller that needs them.
- An `AgentgatewayPolicy` (`<gateway-name>-jwt`) is created with `traffic.jwtAuthentication.mode: Strict` targeting the Gateway resource. Requests through the data-plane HTTP listener must carry a valid JWT issued by `https://sso.<domain>/realms/uds`.
- The `mcp.resourceMetadata` block is included so MCP clients can discover the auth server via the standard `.well-known/oauth-protected-resource` endpoint.
- Two egress rules are added to the Package CR: data-plane proxy → `keycloak.keycloak:8080` (for JWKS) and proxy → tenant gateway:443 (so the proxy can reach the public issuer URL if needed).
- The admin UI is **not** affected — `targetRefs` is the Gateway resource only, and the admin UI is served on a separate listener that's not part of the Gateway's listeners.

**Mapping client tokens to the `aud` claim**: Keycloak by default doesn't put the resource client's ID into tokens minted for other clients. Either:
1. Add an audience-mapper to the calling client(s) in Keycloak so their tokens carry `aud: agentgateway-mcp`, or
2. Extend `keycloak.audiences` to list the client IDs of every caller you want to accept.

**Bypassing for trusted callers**: set `keycloak.audiences` to include those client IDs; there's no allowlist-by-path knob in this chart. If you need path-based exemptions, replace the Gateway-level policy with HTTPRoute-level policies (attach the same `AgentgatewayPolicy` body to specific HTTPRoutes via `targetRefs.kind: HTTPRoute` instead).

## Network policy

The Package CR generates:

- `Ingress` + `Egress` for IntraNamespace
- `Egress` from the controller (`app.kubernetes.io/name: agentgateway`) to the Kube API server
- The two `expose` rules above (which Pepr expands into per-gateway ingress allows + VirtualServices)

Anything else — egress to LLM providers, downstream MCP backends, etc. — should be added via `additionalNetworkAllow` in values, or by overriding `additionalNetworkAllow` in the bundle.

## Troubleshooting

**`503 UC upstream_reset_before_response_started` on the admin UI** — proxy pods predate the `AgentgatewayParameters` change. Delete them so the controller recreates with the new bind:
```bash
kubectl delete pods -n agentgateway -l gateway.networking.k8s.io/gateway-name=agentgateway-proxy
```

**Admin / MCP routes return 503 with no upstream** — no data-plane pods exist yet. Verify the Gateway is `Accepted` + `Programmed`:
```bash
kubectl get gateway -n agentgateway agentgateway-proxy -o yaml
kubectl get pods -n agentgateway -l gateway.networking.k8s.io/gateway-name=agentgateway-proxy
```
If the Gateway never programs, check the controller logs (`kubectl logs -n agentgateway deploy/agentgateway`) and that the standard Gateway API CRDs are present.

**Controller `CrashLoopBackOff` mentioning `gatewayclasses`** — standard Gateway API CRDs are missing. Install them (UDS Core's Istio bundle does this) or apply the standard channel manifests from `kubernetes-sigs/gateway-api`.

**MCP route 404s on `/mcp`** — you've reached the data plane but no `HTTPRoute` attaches `/mcp` to a backend. Apply one (see upstream MCP quickstart).

**MCP returns 401 after enabling `keycloak.enabled`** — expected, the JWT policy is in `Strict` mode. Check:
- Caller is presenting a `Bearer` token in the `Authorization` header.
- Token's `iss` claim equals `https://sso.<domain>/realms/uds`.
- Token's `aud` claim is in `keycloak.audiences` (default `[agentgateway-mcp]`). Use `jwt.io` to inspect — if the audience doesn't match, add the caller's client ID to `keycloak.audiences` or configure a Keycloak audience-mapper to inject `agentgateway-mcp`.
- Data-plane pod can reach Keycloak: `kubectl logs -n agentgateway -l gateway.networking.k8s.io/gateway-name=agentgateway-proxy` and look for JWKS-fetch errors.

## Versions

- agentgateway: `v1.2.1`
- agentgateway-crds: `v1.2.1`
- uds-common tasks: `v1.24.11`
