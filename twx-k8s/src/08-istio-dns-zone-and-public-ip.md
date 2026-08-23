# Migrating Ingress from Nginx to Istio — DNS Zone, Public IP and Gateway



## Prerequisites

This chapter assumes you have already completed the following from the original series:

- **Chapter 07**: cert-manager is installed and running in the `cert-manager` namespace.
- **Chapter 08**: The nginx-based ingress is fully operational, including:
  - A DNS Zone (e.g., `demo.dxu.edc.devops.ptc.io`)
  - A Public IP bound to the ingress-nginx controller
  - A wildcard DNS A record `*.CLUSTER_NAME` pointing to that Public IP
  - A ClusterIssuer (`letsencrypt-cluster-issuer`) configured with ACME + Azure DNS-01 challenge
  - A wildcard certificate (`letsencrypt-certificate`) in `kube-system` namespace
  - An ingress-nginx deployment with `ingressClass: nginx`

The goal of this chapter is to deploy an Istio-based ingress **alongside** the existing nginx ingress, so that both can coexist during the migration period. Existing Thingworx instances using nginx will not be affected.



## What Can Be Reused

Before creating new resources, it is important to understand which existing infrastructure components can be shared with the Istio-based ingress.

| Resource | Reuse? | Explanation |
|----------|--------|-------------|
| **cert-manager** | Yes | cert-manager is a cluster-level operator. It manages both external certificates (via Let's Encrypt) and internal certificates (Akka TLS, Ignite TLS, etc.). It has no dependency on the ingress controller type. |
| **ClusterIssuer** | Yes | The `letsencrypt-cluster-issuer` is a cluster-scoped resource. It can issue certificates for any domain within the same DNS Zone, regardless of whether the consuming workload uses nginx or Istio. |
| **DNS Zone** | Yes | The DNS Zone itself is just a container for DNS records. Multiple A records (for different IPs) can coexist within the same zone. |
| **Service Principal & Secret** | Yes | The Service Principal created in Chapter 08 (step 4-5) grants cert-manager the "DNS Zone Contributor" role for the DNS Zone. Since we reuse the same DNS Zone and ClusterIssuer, no new Service Principal is needed. |
| **Existing Public IP** | No | The current Public IP is bound to the nginx `LoadBalancer` Service. Istio needs its own `LoadBalancer` Service and therefore its own Public IP. |
| **Existing wildcard certificate** | No | The existing certificate covers `*.CLUSTER_NAME.DNS_ZONE_NAME`. For Istio, we will use a different subdomain pattern, which requires a new certificate. |
| **Existing DNS A record** | No | The existing `*.CLUSTER_NAME` record points to the nginx IP. We need a new record pointing to the Istio IP. |



## Step 1: Choose a DNS Naming Strategy

Since nginx and Istio will coexist, each needs its own domain pattern mapped to its own Public IP. We recommend adding a subdomain prefix for the Istio entry point.

**Naming Convention:**

| Ingress Controller | DNS Pattern | Example FQDN |
|---|---|---|
| Nginx (existing) | `*.CLUSTER_NAME.DNS_ZONE_NAME` | `myapp-twx101.dxudemo-aks.demo.dxu.edc.devops.ptc.io` |
| Istio (new) | `*.istio.CLUSTER_NAME.DNS_ZONE_NAME` | `myapp-twx101.istio.dxudemo-aks.demo.dxu.edc.devops.ptc.io` |

This ensures complete isolation: all traffic to `*.dxudemo-aks.*` continues to go through nginx, while traffic to `*.istio.dxudemo-aks.*` goes through Istio.

Define the following environment variables in your `.env` file (or export them in your shell). These are **in addition to** the existing variables from Chapter 08:

```shell
# Istio-specific variables (add to .env)
ISTIO_IP_NAME="dxudemo-istio-publicip"
ISTIO_CERTIFICATE_NAME="letsencrypt-istio-certificate"
```

The following variables from the original `.env` are **reused as-is** and do not need to change:

```shell
# These are REUSED from Chapter 08 — do NOT duplicate or change
DNS_ZONE_RESOURCE_GROUP="axeda-loadtest-rg"
DNS_ZONE_NAME="demo.dxu.edc.devops.ptc.io"
CLUSTER_NAME="dxudemo-aks"
CLUSTER_ISSUER_NAME="letsencrypt-cluster-issuer"
SECRET_NAME="cert-manager-azuredns-secret"
```



## Step 2: Create a New Public IP for Istio

Create a dedicated static Public IP for the Istio ingress gateway. This IP will be assigned to the Istio `LoadBalancer` Service.

```shell
export MC_RESOURCE_GROUP=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query nodeResourceGroup -o tsv)

az network public-ip create -g $MC_RESOURCE_GROUP \
    --name $ISTIO_IP_NAME \
    --sku Standard \
    --allocation-method static \
    --zone 1 2 3
```

Retrieve the IP address:

```shell
export ISTIO_IP_ADDRESS=$(az network public-ip show -g $MC_RESOURCE_GROUP \
    -n $ISTIO_IP_NAME \
    --query "ipAddress" -o tsv)

echo "Istio Public IP: $ISTIO_IP_ADDRESS"
```



## Step 3: Add DNS Record for the Istio Public IP

Add a wildcard A record under the **existing DNS Zone** that points to the new Istio Public IP. Note the record-set name uses `*.istio.CLUSTER_NAME` to distinguish it from the nginx record `*.CLUSTER_NAME`.

```shell
az network dns record-set a add-record \
    --resource-group $DNS_ZONE_RESOURCE_GROUP \
    --zone-name $DNS_ZONE_NAME \
    --record-set-name "*.istio.$CLUSTER_NAME" \
    --ipv4-address $ISTIO_IP_ADDRESS
```

**Caution**: On `zsh` or other shells that perform filename globbing, wrap the `*` in double quotes if needed.

### Validate DNS resolution

```shell
host test-app.istio.${CLUSTER_NAME}.${DNS_ZONE_NAME}
```

The response should resolve to your new Istio Public IP address. Compare with the existing nginx resolution:

```shell
host test-app.${CLUSTER_NAME}.${DNS_ZONE_NAME}
```

The two commands should return **different** IP addresses — the first is the Istio IP, the second is the nginx IP.



## Step 4: Create a New Wildcard Certificate for Istio

Since the Istio domain pattern (`*.istio.CLUSTER_NAME.DNS_ZONE_NAME`) differs from the nginx pattern, we need a separate TLS certificate. The existing **ClusterIssuer** is reused — no new issuer is needed.

Create a certificate template file `templates/istio-certificate.tpl.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: $ISTIO_CERTIFICATE_NAME
  namespace: istio-system
spec:
  commonName: "*.istio.$CLUSTER_NAME.$DNS_ZONE_NAME"
  dnsNames:
    - "*.istio.$CLUSTER_NAME.$DNS_ZONE_NAME"
  secretName: $ISTIO_CERTIFICATE_NAME
  duration: 2160h
  renewBefore: 1h
  issuerRef:
    kind: ClusterIssuer
    name: $CLUSTER_ISSUER_NAME
```

Key differences from the nginx certificate (Chapter 08 step 7):

| Aspect | Nginx Certificate | Istio Certificate |
|--------|------------------|-------------------|
| Name | `letsencrypt-certificate` | `letsencrypt-istio-certificate` |
| Namespace | `kube-system` | `istio-system` |
| Common Name | `*.CLUSTER_NAME.DNS_ZONE_NAME` | `*.istio.CLUSTER_NAME.DNS_ZONE_NAME` |
| ClusterIssuer | `letsencrypt-cluster-issuer` | `letsencrypt-cluster-issuer` (same) |

The certificate is placed in the `istio-system` namespace because the Istio ingress gateway runs there and needs direct access to the TLS secret.

Generate and apply:

```shell
envsubst < templates/istio-certificate.tpl.yaml > cluster/istio-certificate.yaml

kubectl create namespace istio-system --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f cluster/istio-certificate.yaml
```

Validate:

```shell
kubectl get certificate -n istio-system
```

It may take a few minutes for the `Ready` status to become `True`. cert-manager will use the same ACME DNS-01 challenge process (via the existing Service Principal) to prove domain ownership and obtain the certificate from Let's Encrypt.



## Step 5: Install Istio and Gateway API CRDs

### 5.1 Install the Kubernetes Gateway API CRDs

The Kubernetes Gateway API is a **separate standard** from Istio's own networking CRDs. Istio implements this standard, but the CRDs must be installed explicitly. Without them, any Helm chart that creates `Gateway` or `HTTPRoute` resources will fail with:

```
no matches for kind "Gateway" in version "gateway.networking.k8s.io/v1"
```

Install the Gateway API CRDs:

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

Validate:

```shell
kubectl get crd | grep gateway.networking.k8s.io
```

You should see CRDs including `gateways.gateway.networking.k8s.io`, `httproutes.gateway.networking.k8s.io`, and `gatewayclasses.gateway.networking.k8s.io`.

### 5.2 Install Istio CLI

Download and install `istioctl`:

On Linux/Mac:

```shell
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/istio-*/bin:$PATH
```

On Windows (PowerShell):

```powershell
# Download from https://github.com/istio/istio/releases and add istioctl to your PATH
```

### 5.3 Install Istio with Custom Configuration

Since we are using the Kubernetes Gateway API (shared Gateway model), Istio's classic `istio-ingressgateway` component is **not needed**. The Gateway API will create its own backing proxy and `LoadBalancer` Service automatically when we apply the shared Gateway resource (Step 6). If we were to leave the classic ingressgateway enabled, it would claim the static Public IP, preventing the Gateway API service from using it.

Create a file `cluster/istio-values.yaml`:

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

### 5.4 Validate Istio Installation

```shell
kubectl get pods -n istio-system
```

You should see `istiod` running. There should be **no** `istio-ingressgateway` pod — that is expected.

Confirm the `istio` GatewayClass is available (created automatically by istiod when it detects the Gateway API CRDs):

```shell
kubectl get gatewayclass
```

You should see a GatewayClass named `istio`.



## Step 6: Create the Shared Gateway

### Dedicated vs Shared Gateway Model

The thingworx-server Helm chart supports two Istio deployment models:

- **Dedicated model** (`gateway.enabled: true`): Each Thingworx deployment creates its own `Gateway` resource. Istio automatically provisions a dedicated proxy pod and `LoadBalancer` Service per deployment, each with its own Public IP. This provides full isolation but consumes more resources.

- **Shared model** (`gateway.enabled: false` + `parentRefs`): A single shared `Gateway` resource is created once at the cluster level. Each Thingworx deployment only creates `HTTPRoute`, `DestinationRule`, and `EnvoyFilter` resources that reference the shared Gateway. This results in a single Public IP, a single wildcard DNS record, and a single wildcard TLS certificate — similar to how the shared nginx ingress controller works.

**This guide uses the shared model**, which is analogous to the existing shared nginx architecture.

### 6.1 Create the shared Gateway resource

When we apply a `Gateway` resource with `gatewayClassName: istio`, Istio automatically creates a backing Deployment + `LoadBalancer` Service for it. We use two Gateway API fields to control this auto-created Service:

- `spec.addresses` — requests a specific static IP for the LoadBalancer (maps to `loadBalancerIP` on the Service).
- `spec.infrastructure.annotations` — propagates annotations to the auto-created Service (needed for Azure to find the static IP in the correct resource group).

Apply the following in the `istio-system` namespace. Replace `<MC_RESOURCE_GROUP>` with the actual node resource group name, and `<ISTIO_IP_ADDRESS>` with the static IP from Step 2:

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
      value: "<ISTIO_IP_ADDRESS>"
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
      hostname: "*.istio.dxudemo-aks.demo.dxu.edc.devops.ptc.io"
      tls:
        mode: Terminate
        certificateRefs:
          - name: letsencrypt-istio-certificate
      allowedRoutes:
        namespaces:
          from: All
```

Key points:

- `addresses` ensures the auto-created LoadBalancer Service binds to the pre-created static Public IP, rather than getting a random dynamic IP.
- `infrastructure.annotations` passes annotations to the auto-created Service:
  - The `azure-load-balancer-resource-group` annotation tells AKS where to find the static IP.
  - The health probe annotations (`port_443_health-probe_port`, `port_443_health-probe_protocol`, `health-probe-request-path`) redirect the Azure Load Balancer's health check for port 443 to Istio's HTTP health endpoint on port 15021. Without these, the LB probes port 443 directly with no TLS SNI, causing Envoy to reject the probe (`filter_chain_not_found`), which makes the LB mark the backend as unhealthy and stop forwarding external traffic.
- `hostname` uses a wildcard matching all Thingworx instances under the Istio domain pattern.
- `certificateRefs` points to the wildcard TLS certificate created in Step 4, which lives in the same `istio-system` namespace — no cross-namespace secret reference needed.
- `allowedRoutes.namespaces.from: All` permits `HTTPRoute` resources from **any** namespace to attach to this Gateway. This is the mechanism that enables cross-namespace routing — **no `ReferenceGrant` is needed**.

Save the above as `cluster/shared-istio-gateway.yaml` and apply:

```shell
kubectl apply -f cluster/shared-istio-gateway.yaml
```

**Important**: The classic `istio-ingressgateway` must be disabled (Step 5.3) before applying this Gateway. Otherwise, both the classic Service and the Gateway API's auto-created Service will compete for the same static IP.

### 6.2 Why ReferenceGrant is NOT needed

In the Kubernetes Gateway API, cross-namespace access is controlled by two separate mechanisms:

| Mechanism | Controls | Where it lives |
|-----------|----------|----------------|
| `allowedRoutes` (on Gateway listener) | Which namespaces can attach HTTPRoutes to this Gateway | On the Gateway resource itself |
| `ReferenceGrant` | Cross-namespace references to backend **Services** or **Secrets** | In the target namespace being referenced |

In our architecture:

1. **HTTPRoute (in `twx101`) → Gateway (in `istio-system`)**: Controlled by `allowedRoutes.namespaces.from: All`, not by ReferenceGrant.
2. **HTTPRoute (in `twx101`) → backend Services (in `twx101`)**: Same namespace — no grant needed.
3. **Gateway (in `istio-system`) → TLS Secret (in `istio-system`)**: Same namespace — no grant needed.

### 6.3 Validate the shared Gateway

```shell
kubectl get gateway -n istio-system
```

Verify that `PROGRAMMED` is `True` and `ADDRESS` matches your static IP.

Also check the auto-created backing Service:

```shell
kubectl get svc -n istio-system | grep shared-istio-gateway
```

You should see a `LoadBalancer` Service named `shared-istio-gateway-istio` with `EXTERNAL-IP` matching your static IP. This is the only Istio LoadBalancer Service — there should be no `istio-ingressgateway` Service.



## Step 7: Understand How the Thingworx Helm Chart Works with the Shared Gateway

The development version of the `thingworx-server` Helm chart (v1.3.0) introduces Istio support through the Kubernetes Gateway API. The chart's `values.yaml` has both `ingress` (nginx) and `gateway` (Istio) sections:

```yaml
ingress:
  enabled: false
  ingressClassName: nginx
  ...

gateway:
  enabled: false
  className: istio
  parentRefs:       # <-- NEW: reference to external shared Gateway
  ...
```

When using the **shared model**, you set `gateway.enabled: false` and provide `gateway.parentRefs` pointing to the shared Gateway. The chart will then create only:

| Resource | Purpose |
|----------|---------|
| `HTTPRoute` (gateway.networking.k8s.io/v1) | Routing rules: path-based routing to Thingworx, Connection Server, etc. References the shared Gateway via `parentRefs`. |
| `DestinationRule` (networking.istio.io/v1beta1) | Traffic policies: cookie-based session affinity (`consistentHash` with `httpCookie`) to ensure requests from the same user are routed to the same pod in HA deployments |
| `EnvoyFilter` (networking.istio.io/v1alpha3) | Edge cases: max request body size, cookie SameSite attribute |

The `Gateway` resource is **not** created by the chart — it already exists as the shared resource in `istio-system` (Step 6).



## Step 8: Configure Thingworx Deployment for Istio

To deploy a Thingworx instance using the shared Istio gateway, modify the deployment values file (e.g., `deployments/thingworx/main.yaml`) as follows:

### 8.1 Disable nginx ingress, configure shared gateway

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
    - "${DEPLOYMENT_NAME}-thingworx-${DEPLOYMENT_NAMESPACE}.istio.${CLUSTER_NAME}.${CLUSTER_DOMAIN}"
  enableAgentWebSockets: true
  cxserver:
    enabled: true
    serviceName: "${DEPLOYMENT_NAME}-cx"
    port: 8080
    healthPort: 8000
```

Key differences from the dedicated model:
- `gateway.enabled: false` — no `Gateway` resource is created by the chart.
- `gateway.parentRefs` — tells the `HTTPRoute` to attach to the shared Gateway in `istio-system`.
- No `tls` section needed — TLS termination is handled by the shared Gateway, which already references the wildcard certificate.

### 8.2 Update common.env (if needed)

In `common.env`, update the chart reference to point to the development chart:

```
THINGWORX_CHART=thingworx-charts-dev/thingworx-server
THINGWORX_CHART_VERSION=1.3.0
```

### 8.3 eMessage Connector and Connection Server

The eMessage Connector (`emessage-cxserver`) and Connection Server (`always-on-cxserver`) Helm charts currently only support nginx `Ingress` resources. To use them with Istio:

- **Option A**: Keep their `Ingress` resources using `ingressClassName: nginx` (requires nginx to remain running — suitable for the migration period).
- **Option B**: Create separate `HTTPRoute` and `Gateway` resources for these services (requires chart modifications similar to what was done for `thingworx-server`).

During the initial migration, **Option A** is recommended: use Istio for Thingworx Foundation only, while Connection Server and eMessage Connector continue to use the existing nginx ingress.



## Step 9: Validate the Istio-based Deployment

### 9.1 Verify the HTTPRoute and related resources

```shell
kubectl get httproute -n $DEPLOYMENT_NAMESPACE
kubectl get destinationrule -n $DEPLOYMENT_NAMESPACE
kubectl get envoyfilter -n istio-system
```

Note: the shared Gateway is in `istio-system`, but the HTTPRoute and DestinationRule are in the deployment namespace.

### 9.2 Test HTTPS Access

Open a browser and navigate to:

```
https://<DEPLOYMENT_NAME>-thingworx-<NAMESPACE>.istio.<CLUSTER_NAME>.<DNS_ZONE_NAME>/Thingworx
```

For example:

```
https://newco-thingworx-twx101.istio.dxudemo-aks.demo.dxu.edc.devops.ptc.io/Thingworx
```

Verify that:
1. The TLS certificate is valid (issued by Let's Encrypt).
2. The Thingworx login page loads correctly.
3. WebSocket connections work (test with a Connection Server agent if applicable).

### 9.3 Verify nginx is Unaffected

Confirm that the existing nginx-based Thingworx instances are still accessible at their original URLs:

```
https://<DEPLOYMENT_NAME>-thingworx-<NAMESPACE>.<CLUSTER_NAME>.<DNS_ZONE_NAME>/Thingworx
```



## Summary: Resource Inventory

The following table summarizes all resources involved in the side-by-side nginx + Istio setup:

| Resource | Nginx (Chapter 08) | Istio (this chapter) | Shared? |
|----------|--------------------|-----------------------|---------|
| cert-manager | `cert-manager` namespace | Same | Yes |
| ClusterIssuer | `letsencrypt-cluster-issuer` | Same | Yes |
| Service Principal | `dxudemo-aks-cert-manager` | Same | Yes |
| DNS Zone | `demo.dxu.edc.devops.ptc.io` | Same | Yes |
| Gateway API CRDs | N/A | `gateway.networking.k8s.io` CRDs (Step 5.1) | N/A |
| Public IP | `dxudemo-publicip` | `dxudemo-istio-publicip` | No |
| DNS A Record | `*.dxudemo-aks` | `*.istio.dxudemo-aks` | No |
| TLS Certificate | `letsencrypt-certificate` in `kube-system` | `letsencrypt-istio-certificate` in `istio-system` | No |
| Ingress Controller | `ingress-nginx` namespace | `istio-system` namespace (istiod + Gateway API auto-created proxy) | No |
| IngressClass / GatewayClass | `nginx` | `istio` (auto-registered by istiod) | No |
| Classic `istio-ingressgateway` | N/A | **Disabled** — not needed with Gateway API; IP is managed by the shared Gateway instead | N/A |
| Shared Gateway | N/A (nginx uses IngressClass) | `shared-istio-gateway` in `istio-system` — owns the static IP via `spec.addresses` (Step 6) | Shared across all Thingworx deployments |

This architecture ensures zero-downtime migration: you can switch individual Thingworx instances from nginx to Istio one at a time, and roll back by simply toggling `ingress.enabled` / `gateway.parentRefs` in the deployment values.

### Per-deployment vs Cluster-level resources

| Scope | Resources |
|-------|-----------|
| **Cluster-level (one-time setup)** | Gateway API CRDs, Istio, Public IP, DNS record, TLS Certificate, shared Gateway |
| **Per-deployment (per Thingworx instance)** | HTTPRoute, DestinationRule, EnvoyFilter |
