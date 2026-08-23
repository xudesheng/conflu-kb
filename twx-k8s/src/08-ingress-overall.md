# Istio Ingress Deployment Modes — Overall Analysis (Draft)



## Key Constraint

Each Kubernetes Gateway API `Gateway` resource triggers Istio to auto-create a dedicated Deployment (proxy pod) and `LoadBalancer` Service. Therefore:

> **One Gateway = One LoadBalancer = One Public IP**

This means the number of Gateways directly determines the number of Public IPs consumed.



## Deployment Dimensions

Every Thingworx + eMessage deployment involves decisions across four dimensions:

| Dimension | Options |
|-----------|---------|
| **Gateway scope** | Per-app, Per-namespace, Cluster-wide |
| **Public IP** | Per-app, Per-namespace, Cluster-wide (follows Gateway scope) |
| **Certificate** | Per-app (specific), Per-namespace (wildcard), Cluster-wide (wildcard) |
| **Hostname** | Always unique per app (this is a given — TWX and EMC have different FQDNs) |

Since IP follows Gateway scope, the independent variables are really: **Gateway scope** and **Certificate scope**.



## Enumerated Modes

### Mode A: Fully Dedicated — One Gateway Per App

```
Namespace: twx101
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌─── TWX Gateway ───┐    ┌─── EMC Gateway ───┐    │
│  │  IP: 1.2.3.4      │    │  IP: 1.2.3.5      │    │
│  │  Host: twx.a.com  │    │  Host: emc.a.com  │    │
│  │  Cert: cert-twx   │    │  Cert: cert-emc   │    │
│  └────────┬──────────┘    └────────┬──────────┘    │
│           │                        │                │
│    ┌──────┴──────┐          ┌──────┴──────┐        │
│    │ HTTPRoute   │          │ HTTPRoute   │        │
│    │ DestRule    │          │ DestRule    │        │
│    │ EnvoyFilter │          │ EnvoyFilter │        │
│    └─────────────┘          └─────────────┘        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

- **Gateways**: 2 per namespace (TWX + EMC)
- **Public IPs**: 2 per namespace
- **Certificates**: Can be individual or wildcard — each Gateway references its own
- **DNS**: One A record per app hostname
- **Chart behavior**: Both charts use `gateway.enabled: true` independently

| Property | Value |
|----------|-------|
| Isolation | Maximum — each app is fully independent |
| Resource cost | Highest — 2 proxy pods, 2 LBs, 2 IPs per namespace |
| Operational complexity | Moderate — each app is self-contained |
| Suitable for | High-security environments, multi-tenant with strict isolation |


### Mode B: Namespace-Shared Gateway — One Gateway Per Namespace

```
Namespace: twx101
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌────────── Namespace Gateway ──────────┐          │
│  │  IP: 1.2.3.4                         │          │
│  │  Host: *.twx101.a.com  (wildcard)    │          │
│  │  Cert: cert-twx101-wildcard          │          │
│  └──────────────┬───────────────────────┘          │
│                 │                                   │
│       ┌─────────┴─────────┐                        │
│       │                   │                        │
│  ┌────┴─────┐       ┌────┴─────┐                  │
│  │ TWX      │       │ EMC      │                  │
│  │ HTTPRoute│       │ HTTPRoute│                  │
│  │ DestRule │       │ DestRule │                  │
│  │ EnvFilter│       │ EnvFilter│                  │
│  └──────────┘       └──────────┘                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

- **Gateways**: 1 per namespace
- **Public IPs**: 1 per namespace
- **Certificate**: 1 wildcard per namespace (e.g., `*.twx101.cluster.domain`)
- **DNS**: One wildcard A record per namespace
- **Chart behavior**: Neither chart creates the Gateway — it is created externally (e.g., by a shared namespace setup script or a separate Helm chart). Both charts use `gateway.parentRefs` to reference it.

| Property | Value |
|----------|-------|
| Isolation | Per-namespace — apps in same namespace share ingress |
| Resource cost | Medium — 1 proxy pod, 1 LB, 1 IP per namespace |
| Operational complexity | Medium — namespace Gateway must be created before deploying apps |
| Suitable for | Standard deployments with namespace-level isolation |

**Important**: The namespace-level Gateway must use a wildcard hostname (e.g., `*.twx101.cluster.domain`) or multiple explicit listeners so that HTTPRoutes from both TWX and EMC can attach. If the Gateway only lists the TWX hostname, the EMC HTTPRoute cannot match.

**Who creates the namespace Gateway?**

| Option | Pros | Cons |
|--------|------|------|
| Separate manifest (kubectl) | Clean separation, no chart coupling | Extra manual step per namespace |
| TWX chart creates it (with wildcard) | Automated, one fewer step | Coupling: EMC depends on TWX being deployed first; TWX Gateway must know about EMC's hostname pattern |
| Dedicated "namespace-gateway" Helm chart | Reusable, parameterized | One more chart to maintain |

Recommendation: Separate manifest or a lightweight "namespace-gateway" chart. This avoids coupling between TWX and EMC charts.


### Mode C: Cluster-Wide Shared Gateway

```
Namespace: istio-system
┌───────────────────────────────────────────────────────┐
│  ┌────────── Shared Gateway ───────────────────┐      │
│  │  IP: 1.2.3.4                                │      │
│  │  Host: *.istio.cluster.domain  (wildcard)   │      │
│  │  Cert: cluster-wide-wildcard                │      │
│  │  allowedRoutes: from All namespaces         │      │
│  └─────────────────┬──────────────────────────┘      │
│                    │                                   │
└────────────────────┼───────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
  Namespace: twx101  │     Namespace: twx102
  ┌─────┴────┐       │     ┌─────┴────┐
  │TWX Route │       │     │TWX Route │
  │EMC Route │       │     │EMC Route │
  │DestRules │       │     │DestRules │
  │EnvFilter │       │     │EnvFilter │
  └──────────┘       │     └──────────┘
                     │
               Namespace: twx103
               ┌─────┴────┐
               │TWX Route │
               │EMC Route │
               │DestRules │
               │EnvFilter │
               └──────────┘
```

- **Gateways**: 1 for the entire cluster (or a small fixed number)
- **Public IPs**: 1 (or a small fixed number)
- **Certificate**: 1 cluster-wide wildcard (e.g., `*.istio.cluster.domain`)
- **DNS**: 1 wildcard A record
- **Chart behavior**: No chart creates a Gateway. Both charts use `gateway.parentRefs` pointing to the cluster-wide Gateway in `istio-system`.

| Property | Value |
|----------|-------|
| Isolation | Minimal — all apps share one ingress point |
| Resource cost | Lowest — 1 proxy pod, 1 LB, 1 IP for all |
| Operational complexity | Low after initial setup — apps just add routes |
| Suitable for | Dev/test environments, cost-sensitive, similar to the shared nginx model |

This is the mode we have currently deployed and tested.


### Mode D: Hybrid — Dedicated Gateway for TWX, Shared for EMC

```
Namespace: twx101
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌─── TWX Gateway ───┐                             │
│  │  IP: 1.2.3.4      │                             │
│  │  Host: twx.a.com  │                             │
│  │  Cert: cert-twx   │                             │
│  └────────┬──────────┘                             │
│           │                                         │
│    ┌──────┴──────┐       ┌──────────────┐          │
│    │ TWX         │       │ EMC          │          │
│    │ HTTPRoute   │       │ HTTPRoute ───┼─── parentRefs ──→ TWX Gateway
│    │ DestRule    │       │ DestRule     │          │
│    │ EnvoyFilter │       │ EnvoyFilter  │          │
│    └─────────────┘       └──────────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

- **Gateways**: 1 per namespace (created by TWX chart)
- **Public IPs**: 1 per namespace
- **Certificate**: Wildcard or multi-SAN covering both TWX and EMC hostnames
- **Chart behavior**: TWX uses `gateway.enabled: true` with wildcard/multi-host listeners. EMC uses `gateway.parentRefs` pointing to the TWX-created Gateway.

| Property | Value |
|----------|-------|
| Isolation | Per-namespace (like Mode B) |
| Resource cost | Medium — 1 proxy, 1 LB, 1 IP per namespace |
| Operational complexity | High — EMC depends on TWX release name to form the parentRef; TWX Gateway must include EMC's hostname |
| Suitable for | Teams that want per-namespace isolation without a separate Gateway manifest |

**Challenge**: The TWX Gateway's listeners must cover the EMC hostname. This means either:
- Using a wildcard hostname in the TWX Gateway listener (e.g., `*.twx101.cluster.domain`)
- Adding EMC's hostname as an additional listener in the TWX Gateway's `tls` list

Both approaches create coupling between the TWX chart configuration and the EMC deployment.



## Summary: Mode Comparison

| Aspect | Mode A (Fully Dedicated) | Mode B (NS-Shared) | Mode C (Cluster-Shared) | Mode D (Hybrid) |
|--------|--------------------------|---------------------|-------------------------|-----------------|
| Gateways per NS | 2 (TWX + EMC) | 1 (external) | 0 (cluster-level) | 1 (TWX-created) |
| IPs per NS | 2 | 1 | 0 (shared) | 1 |
| Total IPs (N namespaces) | 2N | N | 1 | N |
| Certificate | Per-app or wildcard | Per-NS wildcard | Cluster wildcard | Per-NS wildcard |
| DNS records | Per-app | Per-NS wildcard | 1 cluster wildcard | Per-NS wildcard |
| Gateway creator | Each chart | External | External (cluster-admin) | TWX chart |
| Chart coupling | None | None (both use parentRefs) | None (both use parentRefs) | High (EMC depends on TWX) |
| Tested | No | No | **Yes** (current setup) | No |



## Minimum Chart Parameters Per Mode

### thingworx-server chart

| Parameter | Mode A | Mode B | Mode C | Mode D |
|-----------|--------|--------|--------|--------|
| `gateway.enabled` | `true` | `false` | `false` | `true` |
| `gateway.className` | Required | — | — | Required |
| `gateway.hosts` | Required | Required | Required | Required |
| `gateway.tls` | Required | — | — | Required (wildcard covering EMC too) |
| `gateway.addresses` | Recommended | — | — | Recommended |
| `gateway.infrastructure` | Required (Azure) | — | — | Required (Azure) |
| `gateway.parentRefs` | — | Required | Required | — |
| `gateway.session_cookie_name` | Optional | Optional | Optional | Optional |
| `gateway.proxy.bodySize` | Optional | Optional | Optional | Optional |

### emessage-cxserver chart

| Parameter | Mode A | Mode B | Mode C | Mode D |
|-----------|--------|--------|--------|--------|
| `gateway.enabled` | `true` | `false` | `false` | `false` |
| `gateway.className` | Required | — | — | — |
| `gateway.hosts` | Required | Required | Required | Required |
| `gateway.tls` | Required | — | — | — |
| `gateway.addresses` | Recommended | — | — | — |
| `gateway.infrastructure` | Required (Azure) | — | — | — |
| `gateway.parentRefs` | — | Required | Required | Required (→ TWX Gateway) |
| `gateway.session_cookie_name` | Optional | Optional | Optional | Optional |
| `gateway.proxy.bodySize` | Optional | Optional | Optional | Optional |

### Summary of what each chart must support

Both charts need exactly the same parameter surface:

```yaml
gateway:
  enabled: false/true        # create a Gateway resource?
  className: istio            # GatewayClass (when enabled)
  hosts: []                   # hostnames for HTTPRoute
  tls: []                     # TLS listeners (when enabled)
  addresses: []               # static IP (when enabled)
  infrastructure: {}          # Azure annotations (when enabled)
  parentRefs: []              # reference to external Gateway
  session_cookie_name: ""     # session affinity cookie
  proxy:
    bodySize: 150m            # max request body size
```

The only difference is the **default cookie name** (`SERVER` for TWX, `EMESSAGE_SERVER` for EMC) to avoid cookie collision when both apps share the same Gateway proxy.

**Current chart status**: Both charts already support this full parameter surface.



## Critical Implementation Detail: EnvoyFilter Namespace

### The Problem

`EnvoyFilter` is a **namespace-scoped** Istio resource. Its `workloadSelector` only matches proxy pods **in the same namespace** as the EnvoyFilter itself. When Istio creates a Gateway proxy pod, that pod runs in the **Gateway's namespace**.

This creates a mode-dependent namespace requirement:

| Mode | Gateway location | Proxy pod location | EnvoyFilter must be in |
|------|-----------------|-------------------|----------------------|
| A (dedicated) | Release namespace (e.g., `twx101`) | `twx101` | `twx101` |
| B (NS-shared) | App namespace (e.g., `twx101`) | `twx101` | `twx101` |
| C (cluster-shared) | `istio-system` | `istio-system` | `istio-system` |

An earlier version of both charts hardcoded `namespace: istio-system` on all EnvoyFilter resources. This worked for Mode C but was **broken for Mode A and Mode B** — the EnvoyFilter would be deployed to `istio-system` while the proxy pod runs in the app namespace, so the `workloadSelector` would never match.

### The Fix

Both charts now compute the EnvoyFilter namespace dynamically:

```
{{- $envoyFilterNS := .Release.Namespace }}
{{- if .Values.gateway.parentRefs }}
  {{- $firstRef := index .Values.gateway.parentRefs 0 }}
  {{- if $firstRef.namespace }}
    {{- $envoyFilterNS = $firstRef.namespace }}
  {{- end }}
{{- end }}
```

Logic:
- If `parentRefs[0].namespace` is explicitly set (Mode C) → use that namespace (e.g., `istio-system`)
- If `parentRefs` is set but without an explicit namespace (Mode B) → use the release namespace
- If no `parentRefs` (dedicated Mode A) → use the release namespace

### Verification

Helm template rendering confirms correct behavior for all modes:

| Mode | `--namespace` | parentRefs config | EnvoyFilter namespace |
|------|---------------|-------------------|----------------------|
| A | `twx101` | (none, `gateway.enabled: true`) | `twx101` |
| B | `twx101` | `[{name: ns-gateway}]` | `twx101` |
| C | `twx101` | `[{name: shared-gw, namespace: istio-system}]` | `istio-system` |

This applies identically to both the thingworx-server and emessage-cxserver charts.

### Key Insight for Mode Comparison

This analysis reveals that **Mode B and Mode C are structurally identical from the chart's perspective**. The only difference is whether the `parentRefs` entry includes an explicit `namespace` field:

```yaml
# Mode B — Gateway in same namespace, namespace field omitted
parentRefs:
  - name: ns-gateway

# Mode C — Gateway in different namespace, namespace field required
parentRefs:
  - name: shared-istio-gateway
    namespace: istio-system
```

All downstream behavior (EnvoyFilter namespace, workloadSelector, HTTPRoute attachment) is automatically derived from this single field. No other chart parameter differs between Mode B and Mode C.



## What Each Mode Requires Beyond the Charts

| Pre-requisite | Mode A | Mode B | Mode C | Mode D |
|---------------|--------|--------|--------|--------|
| Gateway API CRDs | Yes | Yes | Yes | Yes |
| Istio installed | Yes | Yes | Yes | Yes |
| Public IP(s) | 2 per NS | 1 per NS | 1 total | 1 per NS |
| DNS record(s) | Per-app A records | Per-NS wildcard | 1 cluster wildcard | Per-NS wildcard |
| Certificate(s) | Per-app or wildcard | Per-NS wildcard | 1 cluster wildcard | Per-NS wildcard (covers both apps) |
| External Gateway manifest | — | Yes (per NS) | Yes (one in istio-system) | — |
| Hostname pattern | `app.cluster.domain` | `app.ns.cluster.domain` | `app-ns.istio.cluster.domain` | `app.ns.cluster.domain` |



## Hostname Pattern Considerations

The choice of mode influences the DNS/hostname pattern:

| Mode | Hostname example (TWX) | Hostname example (EMC) | Wildcard cert covers |
|------|------------------------|------------------------|---------------------|
| A | `twx.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | `emc.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | `*.dxudemo-aks.demo.dxu.edc.devops.ptc.io` |
| B | `twx.twx101.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | `emc.twx101.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | `*.twx101.dxudemo-aks.demo.dxu.edc.devops.ptc.io` (per-NS) |
| C | `newco-twx-twx101.istio.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | `newco-emc-twx101.istio.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | `*.istio.dxudemo-aks.demo.dxu.edc.devops.ptc.io` |
| D | Same as B | Same as B | Same as B |

**Note**: Wildcard certificates only cover one subdomain level. `*.a.com` matches `foo.a.com` but NOT `foo.bar.a.com`. This constrains how deep the hostname hierarchy can go.



## Recommendation

For the installation guide, focus on **Mode B** and **Mode C** as the primary documented paths:

- **Mode C** (cluster-shared) is already tested and proven. It's the simplest to set up and the cheapest to operate. Recommended for dev/test and single-team clusters.

- **Mode B** (namespace-shared) is the natural choice for production multi-tenant clusters where teams want per-namespace isolation without per-app IP overhead. It requires a namespace-level Gateway manifest but no chart changes.

- **Mode A** (fully dedicated) is supported by the charts but should be documented as an advanced option — it's expensive and rarely needed.

- **Mode D** (hybrid) should be **discouraged** due to the tight coupling between TWX and EMC configurations. Mode B achieves the same result with cleaner separation.



## Next Steps

1. Finalize this analysis and confirm which modes to cover in the installation guide.
2. Create a namespace-level Gateway template for Mode B (if needed).
3. Write detailed installation guides for the selected modes.
