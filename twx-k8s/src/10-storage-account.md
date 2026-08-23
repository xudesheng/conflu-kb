# Storage Account

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



## Create Premium Storage Account from Azure Portal

1. Please go to your managed resource group (aka infrastructure group) and select "create"

<img src="docs/10-storage-account/image-20250209013548908.png" alt="image-20250209013548908" style="zoom:40%;" />

2. Search and select "storage account"

<img src="docs/10-storage-account/image-20250209013713123.png" alt="image-20250209013713123" style="zoom:50%;" />

3. **Basics**

   Please make sure to the selected "Resource Group" is your infrastructure resource group and the primary service is `Azure Files`

<img src="docs/10-storage-account/image-20250210000238626.png" alt="image-20250210000238626" style="zoom:40%;" />

4. **Advanced**

   You have to unselect "Require secure transfer for REST API operations" if you want to use `nfs` protocol later.

<img src="docs/10-storage-account/image-20250209014006663.png" alt="image-20250209014006663" style="zoom:50%;" />



5. **Networking**

   For security purpose, I recommend to disable the public access and use private endpoint

<img src="docs/10-storage-account/image-20250209014409972.png" alt="image-20250209014409972" style="zoom:50%;" />

6. Create private endpoint

   **Caution**: please make sure the "Resource Group" is right, the `Storage sub-resource` is `file` and the `vnet/subnet` is correct.

<img src="docs/10-storage-account/image-20250209014440527.png" alt="image-20250209014440527" style="zoom:50%;" />



7. **Data protection**

<img src="docs/10-storage-account/image-20250209014532270.png" alt="image-20250209014532270" style="zoom:50%;" />



8. **Encryption**

<img src="docs/10-storage-account/image-20250209014625709.png" alt="image-20250209014625709" style="zoom:50%;" />



9. **Tags**

   You can add any tag/value here.



10. **Create**

<img src="docs/10-storage-account/image-20250209015125268.png" alt="image-20250209015125268" style="zoom:40%;" />

## Create Storage Class

We will create two storage classes here, one is for `nfs` protocol and one is for `smb` protocol.

```Shell
export MC_RESOURCE_GROUP=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query nodeResourceGroup  -o tsv)
envsubst < templates/storage-nfs.tpl.yaml>cluster/storage-nfs.yaml
envsubst < templates/storage-smb.tpl.yaml>cluster/storage-smb.yaml
```



```
kubectl apply -f cluster/storage-nfs.yaml
kubectl apply -f cluster/storage-smb.yaml
```

<img src="docs/10-storage-account/image-20250209015952933.png" alt="image-20250209015952933" style="zoom:50%;" />



<img src="docs/10-storage-account/image-20250209020218792.png" alt="image-20250209020218792" style="zoom:40%;" />



## Validate the endpoint:

```
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
```

```Shell
nslookup $STORAGE_ACCOUNT_NAME.privatelink.file.core.windows.net
```

<img src="docs/10-storage-account/image-20250210174614227.png" alt="image-20250210174614227" style="zoom:50%;" />



<img src="docs/10-storage-account/image-20250210174650326.png" alt="image-20250210174650326" style="zoom:40%;" />



<img src="docs/10-storage-account/image-20250210174734897.png" alt="image-20250210174734897" style="zoom:30%;" />

Please make sure that the two IP addresses are the same.
