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
│       └── admin-service.yaml             # ClusterIP for proxy:15000
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

## Versions

- agentgateway: `v1.2.1`
- agentgateway-crds: `v1.2.1`
- uds-common tasks: `v1.24.11`
