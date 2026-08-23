# Upgrading Ingress from Nginx to Istio (In-Place Cutover)



## Overview

This document describes how to replace the existing nginx-based ingress with Istio for Thingworx deployments on an already-running cluster.

**Constraints:**

- **Same hostnames**: All existing FQDNs (e.g., `myapp-twx101.dxudemo-aks.demo.dxu.edc.devops.ptc.io`) are preserved. No DNS changes, no new subdomains.
- **Same Public IP**: The static Public IP currently bound to nginx is reused by Istio.
- **Same DNS record**: The existing wildcard A record `*.CLUSTER_NAME` continues to point to the same IP.
- **Downtime is acceptable**: There will be a planned outage during the cutover window while traffic shifts from nginx to Istio.

**High-level flow:**

```
Phase 1: Prepare (nginx still running, no downtime)
  ├── Install Gateway API CRDs
  ├── Install Istio (no ingressgateway)
  ├── Create wildcard certificate in istio-system
  └── Prepare updated deployment values

Phase 2: Cutover (downtime starts)
  ├── Uninstall nginx ingress controller (releases the Public IP)
  ├── Apply shared Istio Gateway (claims the same Public IP)
  └── Redeploy Thingworx with Istio gateway settings

Phase 3: Validate (downtime ends)
  └── Verify HTTPS access, session affinity, WebSocket
```



## Prerequisites

This document assumes the following from the original series are in place:

- **Chapter 07**: cert-manager is installed and running in the `cert-manager` namespace.
- **Chapter 08**: The nginx-based ingress is fully operational:
  - DNS Zone: `demo.dxu.edc.devops.ptc.io`
  - Static Public IP bound to `ingress-nginx` LoadBalancer Service
  - Wildcard DNS A record: `*.CLUSTER_NAME` → Public IP
  - ClusterIssuer: `letsencrypt-cluster-issuer` (ACME + Azure DNS-01)
  - Wildcard certificate: `letsencrypt-certificate` in `kube-system`
  - Ingress controller: `ingress-nginx` with `ingressClass: nginx`
- **Thingworx deployments**: One or more Thingworx instances running with `ingress.enabled: true` and `ingressClassName: nginx`.



## What Can Be Reused

| Resource | Reuse? | Notes |
|----------|--------|-------|
| **cert-manager** | Yes | Cluster-level operator, no dependency on ingress type |
| **ClusterIssuer** | Yes | Cluster-scoped, issues certs for any domain in the same DNS Zone |
| **Service Principal & Secret** | Yes | Same DNS Zone, same ClusterIssuer |
| **DNS Zone** | Yes | Container for DNS records, unchanged |
| **DNS A Record** | Yes | Same `*.CLUSTER_NAME` pointing to same IP — no change needed |
| **Public IP** | Yes | Same static IP, transferred from nginx Service to Istio Gateway Service |
| **Certificate domain** | Yes | Same `*.CLUSTER_NAME.DNS_ZONE_NAME` wildcard — but a **new Certificate CR** is created in `istio-system` because the Istio Gateway needs the TLS secret in its own namespace |



## Phase 1: Prepare (No Downtime)

Everything in this phase is done while nginx is still serving traffic. No disruption occurs.

### 1.1 Install the Kubernetes Gateway API CRDs

The Gateway API is a separate Kubernetes standard that Istio implements. The CRDs must be present before any `Gateway` or `HTTPRoute` resource can be created.

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

Validate:

```shell
kubectl get crd | grep gateway.networking.k8s.io
```

You should see `gateways.gateway.networking.k8s.io`, `httproutes.gateway.networking.k8s.io`, and `gatewayclasses.gateway.networking.k8s.io`.

### 1.2 Install Istio

Since we are using the Gateway API (shared Gateway model), Istio's classic `istio-ingressgateway` component must be **disabled**. If left enabled, it would create its own `LoadBalancer` Service that competes with the Gateway API's auto-created Service for the same Public IP.

Create `cluster/istio-values.yaml`:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: false
  meshConfig:
    accessLogFile: /dev/stdout
```

Install:

```shell
istioctl install -f cluster/istio-values.yaml -y
```

Validate:

```shell
kubectl get pods -n istio-system
```

You should see `istiod` running. There should be **no** `istio-ingressgateway` pod.

Confirm the `istio` GatewayClass exists:

```shell
kubectl get gatewayclass
```

### 1.3 Create Wildcard Certificate in `istio-system`

The existing nginx certificate (`letsencrypt-certificate`) lives in `kube-system`. The Istio Gateway needs the TLS secret in the `istio-system` namespace (where the Gateway resource is created). Rather than copying secrets, we create a new Certificate CR that covers the **same wildcard domain** — cert-manager will issue it independently.

Create `templates/istio-certificate.tpl.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: $CERTIFICATE_NAME-istio
  namespace: istio-system
spec:
  commonName: "*.$CLUSTER_NAME.$DNS_ZONE_NAME"
  dnsNames:
    - "*.$CLUSTER_NAME.$DNS_ZONE_NAME"
  secretName: $CERTIFICATE_NAME-istio
  duration: 2160h
  renewBefore: 1h
  issuerRef:
    kind: ClusterIssuer
    name: $CLUSTER_ISSUER_NAME
```

Note that `commonName` and `dnsNames` are **identical** to the nginx certificate — the only differences are the name and namespace.

Generate and apply:

```shell
kubectl create namespace istio-system --dry-run=client -o yaml | kubectl apply -f -

envsubst < templates/istio-certificate.tpl.yaml > cluster/istio-certificate.yaml

kubectl apply -f cluster/istio-certificate.yaml
```

Validate:

```shell
kubectl get certificate -n istio-system
```

Wait for `Ready` to become `True`. cert-manager uses the same ACME DNS-01 challenge via the existing Service Principal. This typically takes 1–3 minutes.

### 1.4 Prepare Updated Deployment Values

Before the cutover, prepare the modified deployment values files. For each Thingworx instance (e.g., `deployments/thingworx/main.yaml`):

**Before (nginx):**

```yaml
ingress:
  enabled: true
  ingressClassName: "${TWX_URL_INGRESS_ID}"
  hosts:
    - ${DEPLOYMENT_NAME}-thingworx-${DEPLOYMENT_NAMESPACE}.${CLUSTER_NAME}.${CLUSTER_DOMAIN}
  tls:
    - hosts:
        - ${DEPLOYMENT_NAME}-thingworx-${DEPLOYMENT_NAMESPACE}.${CLUSTER_NAME}.${CLUSTER_DOMAIN}
  # ... nginx annotations ...

gateway:
  enabled: false
```

**After (Istio):**

```yaml
ingress:
  enabled: false

gateway:
  enabled: false
  parentRefs:
    - name: shared-istio-gateway
      namespace: istio-system
  session_cookie_name: SERVER
  proxy:
    bodySize: 150m
    readTimeout: '1200'
    sendTimeout: '1200'
  hosts:
    - "${DEPLOYMENT_NAME}-thingworx-${DEPLOYMENT_NAMESPACE}.${CLUSTER_NAME}.${CLUSTER_DOMAIN}"
  enableAgentWebSockets: true
  cxserver:
    enabled: true
    serviceName: "${DEPLOYMENT_NAME}-cx"
    port: 8080
    healthPort: 8000
```

Key points:
- `ingress.enabled: false` — disables nginx `Ingress` resource creation.
- `gateway.enabled: false` + `gateway.parentRefs` — uses the shared Istio Gateway (no per-deployment Gateway).
- `gateway.hosts` uses the **same hostname pattern** as the nginx ingress — no `istio.` subdomain.
- No `tls` section — TLS termination is handled at the shared Gateway level.
- `session_cookie_name: SERVER` — enables cookie-based session affinity for HA deployments.

**Do not apply these changes yet.** Save them and proceed to Phase 2.

### 1.5 Prepare the Shared Gateway YAML

Create `cluster/shared-istio-gateway.yaml`. The `addresses` field must reference the **same static Public IP** currently used by nginx. The `hostname` must match the **same wildcard pattern** used by nginx.

Retrieve the current nginx Public IP:

```shell
export MC_RESOURCE_GROUP=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query nodeResourceGroup -o tsv)

export IP_ADDRESS=$(az network public-ip show -g $MC_RESOURCE_GROUP -n $IP_NAME --query "ipAddress" -o tsv)

echo "Public IP to reuse: $IP_ADDRESS"
```

Create the Gateway manifest:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-istio-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  addresses:
    - type: IPAddress
      value: "<IP_ADDRESS>"
  infrastructure:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-resource-group: "<MC_RESOURCE_GROUP>"
      service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/healthz/ready"
      service.beta.kubernetes.io/port_443_health-probe_port: "15021"
      service.beta.kubernetes.io/port_443_health-probe_protocol: "http"
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.CLUSTER_NAME.DNS_ZONE_NAME"
      tls:
        mode: Terminate
        certificateRefs:
          - name: letsencrypt-certificate-istio
      allowedRoutes:
        namespaces:
          from: All
```

Replace placeholders with actual values. The `certificateRefs` name must match the `secretName` from Step 1.3.

The health probe annotations are critical on Azure:
- Without them, the Azure Load Balancer probes port 443 directly with a plain TCP/HTTPS check (no TLS SNI).
- Envoy rejects the probe (`filter_chain_not_found`), the LB marks the backend unhealthy, and all external traffic is dropped.
- The annotations redirect the health check to Istio's internal HTTP health endpoint on port 15021.

**Do not apply this yet** — the static IP is still held by nginx.



## Phase 2: Cutover (Downtime Starts)

> **⚠️ Downtime begins here.** All Thingworx instances served by nginx will become inaccessible until Istio is fully operational.

### 2.1 Verify Phase 1 Readiness

Before proceeding, confirm:

```shell
# Gateway API CRDs installed
kubectl get crd gateways.gateway.networking.k8s.io

# Istio running
kubectl get pods -n istio-system | grep istiod

# GatewayClass available
kubectl get gatewayclass istio

# Certificate ready
kubectl get certificate -n istio-system
```

All must show healthy / ready status.

### 2.2 Uninstall Nginx Ingress Controller

This releases the static Public IP from the nginx `LoadBalancer` Service, making it available for Istio.

```shell
helm uninstall ingress-nginx -n ingress-nginx
```

Verify the Public IP is no longer bound:

```shell
kubectl get svc -n ingress-nginx
```

The namespace should be empty (or the service should be gone). The static Public IP resource in Azure is **not deleted** — it simply becomes unassigned.

> **Note**: If you have other services using the nginx ingress (e.g., Grafana), they will also lose external access at this point. Plan accordingly.

### 2.3 Apply the Shared Istio Gateway

Now that the IP is free, apply the shared Gateway to claim it:

```shell
kubectl apply -f cluster/shared-istio-gateway.yaml
```

Verify the Gateway is programmed and has the correct IP:

```shell
kubectl get gateway -n istio-system
```

Check that `PROGRAMMED` is `True` and `ADDRESS` matches your Public IP.

Verify the auto-created LoadBalancer Service:

```shell
kubectl get svc -n istio-system | grep shared-istio-gateway
```

You should see a `LoadBalancer` Service named `shared-istio-gateway-istio` with `EXTERNAL-IP` matching your static IP. This may take 1–2 minutes to provision.

### 2.4 Redeploy Thingworx with Istio Settings

Apply the updated deployment values prepared in Step 1.4. If using helmsman:

```shell
helmsman --apply -f common.yaml
```

Or if deploying manually with Helm:

```shell
helm upgrade <release-name> <chart-path> \
    --namespace <namespace> \
    -f <updated-values-file>
```

Verify the Istio resources are created:

```shell
kubectl get httproute -n <DEPLOYMENT_NAMESPACE>
kubectl get destinationrule -n <DEPLOYMENT_NAMESPACE>
kubectl get envoyfilter -n istio-system
```



## Phase 3: Validate (Downtime Ends)

### 3.1 Test HTTPS Access

Open a browser and navigate to the **same URL** as before:

```
https://<DEPLOYMENT_NAME>-thingworx-<NAMESPACE>.<CLUSTER_NAME>.<DNS_ZONE_NAME>/Thingworx
```

Verify:
1. The TLS certificate is valid (issued by Let's Encrypt, same wildcard domain).
2. The Thingworx login page loads correctly.
3. You can log in and the session persists (no repeated login prompts — session affinity is working).

### 3.2 Verify Session Affinity

The Istio `DestinationRule` uses `consistentHash` with `httpCookie` (cookie name: `SERVER` by default) to route requests from the same user to the same pod. If you experience repeated login prompts:

1. Clear browser cookies and retry.
2. Verify the DestinationRule has the correct `loadBalancer` configuration:

```shell
kubectl get destinationrule -n <DEPLOYMENT_NAMESPACE> -o yaml | grep -A 5 consistentHash
```

### 3.3 Verify WebSocket (if applicable)

If Connection Server agents use WebSocket (`/Thingworx/WS` or `/Thingworx/WSTunnelServer`), verify they can connect. The HTTPRoute includes specific rules for these paths when `enableAgentWebSockets: true`.

### 3.4 Verify Other Services

If other services previously used nginx (e.g., Grafana, eMessage), they now need an alternative:
- **Option A**: Create additional `HTTPRoute` resources pointing to the shared Istio Gateway.
- **Option B**: Reinstall nginx alongside Istio with a **different** IP (side-by-side model, described in `08-istio-dns-zone-and-public-ip.md`).



## Rollback Plan

If the Istio deployment fails and you need to restore nginx:

### Quick Rollback

```shell
# 1. Delete the shared Gateway (releases the IP)
kubectl delete gateway shared-istio-gateway -n istio-system

# 2. Reinstall nginx ingress controller (reclaims the IP)
envsubst < templates/ingress-values.tpl.yaml > cluster/ingress-values.yaml

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --version 4.7.1 \
    -f cluster/ingress-values.yaml

# 3. Re-enable nginx in Thingworx deployment values
#    Set ingress.enabled: true, remove gateway.parentRefs
#    Then redeploy
```

The static IP will be reclaimed by nginx, and DNS resolution is unchanged, so external access is restored once nginx is healthy.



## Summary: What Changed vs. What Stayed

| Aspect | Before (Nginx) | After (Istio) | Changed? |
|--------|----------------|---------------|----------|
| Hostname pattern | `*.CLUSTER_NAME.DNS_ZONE_NAME` | `*.CLUSTER_NAME.DNS_ZONE_NAME` | No |
| Public IP | `dxudemo-publicip` | `dxudemo-publicip` (same) | No |
| DNS A record | `*.dxudemo-aks` → IP | `*.dxudemo-aks` → IP (same) | No |
| Certificate domain | `*.dxudemo-aks.demo.dxu.edc.devops.ptc.io` | Same domain, new CR in `istio-system` | Namespace only |
| Ingress controller | `ingress-nginx` | `istiod` + Gateway API proxy | Yes |
| Traffic routing | `Ingress` (nginx) | `HTTPRoute` (Gateway API) | Yes |
| Session affinity | nginx `upstream-hash-by` | Istio `DestinationRule` + `consistentHash` | Mechanism changed |
| TLS termination | nginx ingress controller | Istio Gateway (Envoy proxy) | Mechanism changed |
| Load Balancer Service | `ingress-nginx-controller` in `ingress-nginx` | `shared-istio-gateway-istio` in `istio-system` | Yes |

### Estimated Downtime

The downtime window is from Step 2.2 (nginx uninstall) to Step 3.1 (first successful HTTPS access). Typical duration:

| Step | Estimated Time |
|------|---------------|
| 2.2 Uninstall nginx | ~30 seconds |
| 2.3 Apply Gateway + IP assignment | 1–2 minutes |
| 2.4 Redeploy Thingworx | 2–5 minutes (depends on chart and pod startup) |
| **Total** | **~3–8 minutes** |
