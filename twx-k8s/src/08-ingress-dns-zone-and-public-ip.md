# DNS Zone and Public IP



## Load OS variables every time when you open a new console

### Using Environment Variables in Kubernetes

During the entire serials, it is assumed that you are under the working directory in the console. Please ensure to load the OS variables from the **.env** file every time.

On Linux/Mac:

```shell
. ./helper/load-envfile.sh
```

On Windows Powershell:

```Powershell
helper\Load-EnvFile.ps1
```




## Introduction to the Role of DNS Zone

A DNS Zone is a distinct part of the domain namespace that is managed by a particular organization or administrator. It contains DNS records for a domain, which map domain names to IP addresses, enabling users to access resources using human-readable names instead of numeric IP addresses.

### DNS Configuration in Thingworx Deployments

In each Thingworx deployment, two or three unique access endpoints are created:

1. **User Access:** A dedicated Thingworx URL for end-user interactions.
2. **AlwaysOn Devices:** An endpoint for devices that communicate using the AlwaysOn protocol (via WebSocket).
3. **Axeda Devices:** An endpoint for devices using the Axeda protocol.

For security reasons, every one of these endpoints must be accessed over HTTPS. This requirement means that each endpoint needs a fully qualified domain name (FQDN) to uniquely identify it. Thus, you must set up appropriate DNS records to resolve these FQDNs to the correct public IP addresses. Additionally, each domain should have a valid SSL/TLS certificate installed to secure traffic.

In this series, it is assumed that you have the authority to create and manage a DNS Zone and that you hold the "DNS Zone Contributor" role. This permission is essential for enabling automated certificate issuance via Let's Encrypt.

#### Using the Pre-configured Demo DNS Zone

For colleagues within PTC who have access to the subscription used in this demonstration, a DNS Zone named **demo.dxu.edc.devops.ptc.io** has already been set up for immediate use. Within this zone, domain names follow the pattern:

```
*.CLUSTER_NAME.demo.dxu.edc.devops.ptc.io
```

With this configuration, Cert Manager can automatically request and obtain the necessary certificates.

#### Setting Up Your Own DNS Zone

For those outside of PTC or if you do not have access to the pre-provisioned subscription, it is advisable to create a DNS Zone in the subscription where your cluster is deployed. Ensure you are granted "DNS Zone Contributor" rights. For example, if your organization owns the domain **demotest.io**, you might set up a new DNS Zone such as **demo.demotest.io**. You would then create an NS record in **demotest.io** to delegate DNS management to the new DNS Zone. (The detailed process for subdomain delegation is beyond the scope of this document.)

#### Alternative Approaches

- **Wildcard DNS Record in an Existing Domain:**  
  If you lack the permissions to create a new DNS Zone, consider asking your organization to add a wildcard DNS record. The recommended naming pattern is:

  ```
  *.CLUSTER_NAME.yourdomain.com
  ```

  For instance, if your cluster is named `demo-aks` and your domain is **demotest.io**, you can create a wildcard A record or CNAME record such as `*.demo-aks.demotest.io` that maps to your public IP address. Note that in this scenario, you must manually obtain and manage your SSL/TLS certificates—uploading them (for example, via a ConfigMap) to your cluster during deployment.

- **No DNS Modification Access:**  
  In the event that you do not have permission to modify any DNS records, you must resort to alternative methods to obtain valid certificates. The specifics of these alternative approaches are beyond the scope of this article.

For the purposes of this guide, we assume that you have access to and can manage the pre-configured DNS Zone **demo.dxu.edc.devops.ptc.io**.





## Create Public IP

### 1. Create a public IP

```Shell
export MC_RESOURCE_GROUP=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query nodeResourceGroup  -o tsv)

az network public-ip create -g $MC_RESOURCE_GROUP --name $IP_NAME --sku Standard --allocation-method static
```

<img src="docs/08-dns-zone-public-ip/image-20250208181935129.png" alt="image-20250208181935129" style="zoom:50%;" />



### 2 Add DNS record for the public IP

```Shell
export IP_ADDRESS=$(az network public-ip show -g $MC_RESOURCE_GROUP -n $IP_NAME --query "{address: ipAddress}" -o tsv)

az network dns record-set a add-record --resource-group $DNS_ZONE_RESOURCE_GROUP \
    --zone-name $DNS_ZONE_NAME \
    --record-set-name "*.$CLUSTER_NAME" \
    --ipv4-address $IP_ADDRESS
```

**Caution**: without the double quotes, you may encounter an error on some shell, for example, `zsh`. Some shell may try to perform filename globbing on the unquoted * in the command line.

<img src="docs/08-dns-zone-public-ip/image-20250208182735591.png" alt="image-20250208182735591" style="zoom:50%;" />

### 3. validate

The `test-app` is not a real name yet. But we can use this approach to validate the DNS resolver: 

```Shell
host test-app.${CLUSTER_NAME}.${DNS_ZONE_NAME}
```

<img src="docs/08-dns-zone-public-ip/image-20250208183020871.png" alt="image-20250208183020871" style="zoom:50%;" />



### 4. Create RBAC service principal

```
export ZONE_ID=$(az network dns zone show --name $DNS_ZONE_NAME --resource-group $DNS_ZONE_RESOURCE_GROUP  --query id -o tsv)

export SP_NAME=${CLUSTER_NAME}-cert-manager

az ad sp create-for-rbac --name $SP_NAME \
    --role "DNS Zone Contributor" \
    --scopes $ZONE_ID \
    --query "[appId,password]" \
    --only-show-errors \
    -o tsv > cluster/sp_credentials.txt
```



Verify Service Principal creation:

```shell
az ad sp list --display-name $SP_NAME
```

<img src="docs/08-dns-zone-public-ip/image-20250208183501589.png" alt="image-20250208183501589" style="zoom:50%;" />

### 5. Create Client Secret

```Shell
export SP_ID=$(awk 'NR==1{print $1}' cluster/sp_credentials.txt)
export SP_PASSWORD=$(awk 'NR==2{print $1}' cluster/sp_credentials.txt)

export SUBSCRIPTION_ID=$(az account show --name $SUBSCRIPTION_NAME --query id -o tsv)

#export SECRET_NAME=cert-manager-azuredns-secret
#export CLUSTER_ISSUER_NAME=letsencrypt-cluster-issuer
export EMAIL_ID=xudesheng@gmail.com

kubectl create secret generic $SECRET_NAME --namespace cert-manager --from-literal=CLIENT_SECRET=$SP_PASSWORD
```



### 6. Create Cluster Issuer

```
envsubst < templates/cluster-issuer.tpl.yaml > cluster/cluster-issuer.yaml
```



```
kubectl apply -f cluster/cluster-issuer.yaml
```



```
kubectl get clusterissuers
```

<img src="docs/08-ingress-dns-zone-and-public-ip/image-20250210224629811.png" alt="image-20250210224629811" style="zoom:50%;" />

### 7. Create certificate

```Shell
envsubst < templates/certificate.tpl.yaml > cluster/certificate.yaml
```



```
kubectl apply -f cluster/certificate.yaml
```



```
kubectl get certificate -n kube-system
```

<img src="docs/08-ingress-dns-zone-and-public-ip/image-20250210225136756.png" alt="image-20250210225136756" style="zoom:50%;" />

It may take few minutes for the `Ready` status to show **true**



<img src="docs/08-ingress-dns-zone-and-public-ip/image-20250210022816734.png" alt="image-20250210022816734" style="zoom:50%;" />

## Create Ingress

The `ingressClass` name is `nginx`.

```
envsubst < templates/ingress-values.tpl.yaml > cluster/ingress-values.yaml
```

Please double check the generated `cluster/ingress-values.yaml` file and make sure the field `loadBalancerIP` has the right public IP you created and the field `default-ssl-certificate` has the right name as you configured in the `.env` file

<img src="docs/08-ingress-dns-zone-and-public-ip/image-20250210235945303.png" alt="image-20250210235945303" style="zoom:50%;" />

```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --version 4.7.1 \
    -f cluster/ingress-values.yaml
```

<img src="docs/08-dns-zone-public-ip/image-20250208190529667.png" alt="image-20250208190529667" style="zoom:50%;" />

## Validate the ingress and certificate issuer



```
envsubst < templates/sample-deployment.tpl.yaml > cluster/sample-deployment.yaml
```

```
kubectl apply -f cluster/sample-deployment.yaml 
```

<img src="docs/08-dns-zone-public-ip/image-20250209000910527.png" alt="image-20250209000910527" style="zoom:50%;" />

<img src="docs/08-dns-zone-public-ip/image-20250209000930999.png" alt="image-20250209000930999" style="zoom:50%;" />



## Update the public entry point to Grafana

The Grafana deployed in chapter 6 doesn't have a public accessible entry point. Since we have Ingress and public IP now, we can configure it to have a public accessible entry point.

Go to edit `cluster/prometheus-operator-values.yaml` file and uncomment the lines for `ingress`:

<img src="docs/08-ingress-dns-zone-and-public-ip/image-20250211235359943.png" alt="image-20250211235359943" style="zoom:50%;" />

**Caution**: You'd better use the `Ctrl + / ` on Windows or `cmd + /` on Mac to uncomment. If you manually remove the leading `#`, please make sure don't mess up the indent.

Then, run the following command (again):

On Linux/Mac:
```Shell
helm upgrade --install stable-prometheus-operator \
     -f cluster/prometheus-operator-values.yaml \
     prometheus-community/kube-prometheus-stack \
     --version 34.10.0 \
     --namespace monitoring \
     --create-namespace
```


On Powershell:
```Shell
helm upgrade --install stable-prometheus-operator `
     -f cluster/prometheus-operator-values.yaml `
     prometheus-community/kube-prometheus-stack `
     --version 34.10.0 `
     --namespace monitoring `
     --create-namespace
```



Few minutes later, you should be able to access: https://grafana.cluster_name.dns_zone_name, in my case: it's https://grafana.dxudemo-aks.demo.dxu.edc.devops.ptc.io



<img src="docs/08-ingress-dns-zone-and-public-ip/image-20250211235751626.png" alt="image-20250211235751626" style="zoom:50%;" />
