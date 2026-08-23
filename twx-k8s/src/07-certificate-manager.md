# Certificate Manager



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



## Introduction to the Role of Certificate Manager

A Certificate Manager is a tool that automates the management of SSL/TLS certificates, which are essential for securing communications over the internet. It handles the issuance, renewal, and revocation of certificates, ensuring that your applications and services remain secure and compliant with industry standards.

## Installing a Certificate Manager using Helm

Helm is a package manager for Kubernetes that simplifies the deployment of applications. To install a Certificate Manager, such as cert-manager, follow these steps:

1. **Add the Jetstack Helm Repository**:
   - Add the repository that contains the cert-manager Helm chart:
     ```bash
     helm repo add jetstack https://charts.jetstack.io
     helm repo update
     ```

2. Copy `templates/cert-manager-values.yaml` to the `cluster` subfolder

   On Linux/Mac:

   ```
   cp templates/cert-manager-values.yaml cluster/
   ```
   
3. **Install cert-manager**:

   - Use Helm to install cert-manager in your Kubernetes cluster:
     ```Shell
     helm install cert-manager jetstack/cert-manager --namespace cert-manager \
         --create-namespace \
         --version 1.5.4 \
         --set installCRDs=true \
         -f cluster/cert-manager-values.yaml
     ```

   <img src="docs/07-certificate-manager/image-20250208024402506.png" alt="image-20250208024402506" style="zoom:50%;" />

4. Validation

   ```
   kubectl get pods --namespace cert-manager
   ```

   <img src="docs/07-certificate-manager/image-20250208024446712.png" alt="image-20250208024446712" style="zoom:50%;" />

5. Create a local CA

   Copy the `templates/cert-manager-validation-ca.yaml` to the `cluster` subfolder.

   On Linux/Mac:

   ```
   cp templates/cert-manager-validation-ca.yaml cluster/
   ```

   ```Shell
   kubectl apply -n cert-manager-validation -f cluster/cert-manager-validation-ca.yaml
   ```

   <img src="docs/07-certificate-manager/image-20250208024751894.png" alt="image-20250208024751894" style="zoom:50%;" />

6. Create the **Issuer** that will use the local CA:

   Copy the `templates/cert-manager-validation-issuer.yaml` to the `cluster` subfolder.

   On Linux/Mac:

   ```
   cp templates/cert-manager-validation-issuer.yaml cluster/
   ```

   ```Shell
   kubectl apply -n cert-manager-validation -f cluster/cert-manager-validation-issuer.yaml
   ```

   <img src="docs/07-certificate-manager/image-20250208025009605.png" alt="image-20250208025009605" style="zoom:50%;" />

7. Create **Certificate** to be signed with our internal CA:

   Copy the `templates/cert-manager-validation-cert.yaml` to the `cluster` subfolder.

   On Linux/Mac:

   ```
   cp templates/cert-manager-validation-cert.yaml cluster/
   ```

   ```Shell
   kubectl apply -n cert-manager-validation -f cluster/cert-manager-validation-cert.yaml
   ```

   <img src="docs/07-certificate-manager/image-20250208025205439.png" alt="image-20250208025205439" style="zoom:50%;" />

8. Validate

   On Linux/Mac:

   ```
   kubectl get -A certs,certificaterequests | grep -C5 --color 'True\|False'
   ```

   On Powershell:

   ```Powershell
   kubectl get -A certs,certificaterequests | Select-String -Pattern 'True|False' -Context 5,5
   ```

   <img src="docs/07-certificate-manager/image-20250208025429121.png" alt="image-20250208025429121" style="zoom:50%;" />

9. Check the secret:

   ```Shell
   kubectl -n cert-manager-validation get secret
   ```

   <img src="docs/07-certificate-manager/image-20250208025545660.png" alt="image-20250208025545660" style="zoom:50%;" />

   

