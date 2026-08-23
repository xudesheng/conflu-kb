# AKS Cluster



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



## Creating an AKS Cluster via Azure Portal

Creating an Azure Kubernetes Service (AKS) cluster through the Azure Portal is a straightforward process. Follow these steps to set up your AKS cluster and link it to your Azure Container Registry (ACR):

1. **Navigate to Azure Kubernetes Service**:
   - Go to the [Azure Portal](https://portal.azure.com) and access the "overview" page of your resource group:

     <img src="docs/05-aks-cluster/image-20250208002855596.png" alt="image-20250208002855596" style="zoom:50%;" />

   - Search for "Kubernetes services" in the search bar and select it.

     <img src="docs/05-aks-cluster/image-20250208002948712.png" alt="image-20250208002948712" style="zoom:50%;" />

2. **Create a New AKS Cluster**:
   - Click on "Create" to create a new AKS cluster.

     <img src="docs/05-aks-cluster/image-20250208003034829.png" alt="image-20250208003034829" style="zoom:50%;" />

   - Fill in the basic details such as subscription, resource group, and cluster name.

     <img src="docs/05-aks-cluster/image-20250208003221222.png" alt="image-20250208003221222" style="zoom:50%;" />

     On this page:

     1. **Subscription**: please double check and confirm.
     2. **Resource Group**: please select the resource group you createed in the chapter #2
     3. **Cluster preset configuration**: choose "Dev/Test"
     4. **Kubernetes cluster name**: Please enter the name you defined, here: **dxudemo-aks**
     5. **Region**: Confirm the region selected.
     6. **Availability zones**: None
     7. **AKS pricing tier**: Free if possible. :-)
     8. **Kubernetes version**: 1.30.7 at the time this document is created.
     9. **Rest**: just use default.

     

3. **Configure Node Pools**:
   - Please click the **agentpool** to change the default node pool setting.

     <img src="docs/05-aks-cluster/image-20250208003716655.png" alt="image-20250208003716655" style="zoom:50%;" />

   - Change the default system node pool settings

     <img src="docs/05-aks-cluster/image-20250208003911672.png" alt="image-20250208003911672" style="zoom:50%;" />

     **Node pool name**: **default**

     **OS SKU**: Ubuntu Linux

     **Node size**: Standard D8s v3 (you can choose smaller or bigger, it's not critical)

     **Minimum node count**: 1

     **Maximum node count**: 20

   - Click **Update**

     **Enable virtual nodes**: false

     

4. **Networking**:

   - **Enable private cluster**: false

   - **Set authorized IP ranges**: false (We will add the allowed IP list later.)

   - **Network configuration**: Azure CNI Overlay

   - **Bring your own Azure virtual network**: false

   - **DNS name prefix**: just use the proposed name.

   - **Enable Cilium dataplane and network policy**: false

   - **Network policy**: **Azure**

     <img src="docs/05-aks-cluster/image-20250210020002487.png" alt="image-20250210020002487" style="zoom:50%;" />

5. **Integrate with Azure Container Registry**:

   - In the "Integrations" tab, please select the container registry you created in chapter #3

     <img src="docs/05-aks-cluster/image-20250208004839989.png" alt="image-20250208004839989" style="zoom:50%;" />
     
   - <img src="docs/05-aks-cluster/image-20250210020132816.png" alt="image-20250210020132816" style="zoom:50%;" />

6. **Monitoring**:

   - Unselect all. 

7. **Security**:

   - Only select **Enable Image Cleaner** (but it's optional)

   - You have to deselect **Enable Workload Identity** first, and then deselect **Enable OIDC** 

     <img src="docs/05-aks-cluster/image-20250208005115905.png" alt="image-20250208005115905" style="zoom:50%;" />

8. **Advanced**:

   - Just use the proposed **infrastructure resource group** name, **Caution**: please don't modify it.
   - Please remember this group name, in my case, it's **MC_dxudemo-rg_dxudemo-aks_eastus**. We have to use this name very often later. The pattern is: `MC_${RESOURCE_GROUP}_${CLUSTER_NAME}_${LOCATION}`

9. **Tags**:

   - You can enter any necessary tags.

   

10. **Review and Create**:
   - Review your settings and click "Create" to deploy the AKS cluster.
   - You can go to have two cups of coffee before this step finishes. :-)
   - **Go to resource** to see the new created cluster

11. **Setup public access IP ranges**:

    1. **Settings**
    2. **Networking**
    3. **Public access**
    4. **Manage**
    5. **Set authorized IP ranges**: true
    6. **Authorized IP ranges**: enter the IP ranges
    7. **Save**

12. Get credential to local:

    On Linux/Mac:

    ```Shell
    az aks get-credentials -n ${CLUSTER_NAME} -g ${RESOURCE_GROUP} --admin
    ```

    On Powershell:

    ```Powershell
    az aks get-credentials `
        --name $env:CLUSTER_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --admin
    ```

    

13. Use `k9s` to check

    If you have `k9s` installed, you can use it to check the cluster status now: (**PRESS *0*** to see all namespaces)

    <img src="docs/05-aks-cluster/image-20250208021918180.png" alt="image-20250208021918180" style="zoom:50%;" />


## Creating an AKS Cluster via Azure CLI

For those who prefer using the command line, the Azure CLI provides a powerful way to create and manage AKS clusters. Follow these steps to create an AKS cluster and link it to your ACR:

1. Create the cluster:
   
   On Linux/Mac:
   
   ```Shell
   export SUBNET_ID=$(az network vnet show -g $RESOURCE_GROUP -n $VNET_NAME --query "subnets[0].id" -o tsv)
   
   export MY_PUBLIC_IP=$(curl -s https://ipinfo.io/ip)
   
   az aks create \
     --resource-group ${RESOURCE_GROUP} \
     --name ${CLUSTER_NAME} \
     --location ${LOCATION} \
     --vnet-subnet-id $SUBNET_ID \
     #--docker-bridge-address 172.17.0.1/16 \
     --service-cidr 192.168.0.0/16 \
     --dns-service-ip 192.168.0.10 \
     --nodepool-name default \
     --node-count ${NODE_COUNT} \
     --vm-set-type VirtualMachineScaleSets \
     --enable-cluster-autoscaler \
     --min-count 1 \
     --max-count 20 \
     --kubernetes-version ${KUBERNETES_VERSION} \
     --node-vm-size ${NODE_VM_SIZE} \
     --network-plugin azure \
     --network-policy azure \
     --dns-name-prefix ${CLUSTER_NAME}-dns \
     --attach-acr ${ACR_NAME} \
     --no-ssh-key \
     --enable-managed-identity \
     --api-server-authorized-ip-ranges ${MY_PUBLIC_IP}/32 \
     --yes
   ```
   
   On Powershell:
   
   ```
   # Get subnet ID
   $SUBNET_ID = az network vnet show -g $env:RESOURCE_GROUP -n $env:VNET_NAME --query "subnets[0].id" -o tsv
   
   # Get public IP
   $MY_PUBLIC_IP = (Invoke-WebRequest -Uri "https://ipinfo.io/ip" -UseBasicParsing).Content.Trim()
   
   # Create AKS cluster
   az aks create `
       --resource-group $env:RESOURCE_GROUP `
       --name $env:CLUSTER_NAME `
       --location $env:LOCATION `
       --vnet-subnet-id $SUBNET_ID `
       --service-cidr "192.168.0.0/16" `
       --dns-service-ip "192.168.0.10" `
       --nodepool-name "default" `
       --node-count $env:NODE_COUNT `
       --vm-set-type "VirtualMachineScaleSets" `
       --enable-cluster-autoscaler `
       --min-count 1 `
       --max-count 20 `
       --kubernetes-version $env:KUBERNETES_VERSION `
       --node-vm-size $env:NODE_VM_SIZE `
       --network-plugin "azure" `
       --network-policy "azure" `
       --dns-name-prefix "$env:CLUSTER_NAME-dns" `
       --attach-acr $env:ACR_NAME `
       --no-ssh-key `
       --enable-managed-identity `
       --api-server-authorized-ip-ranges "$MY_PUBLIC_IP/32" `
       --yes
   ```
   
   
   
2. Integrate with Container Registry

   On Linux/Mac:
   ```
   az aks update \
     --resource-group ${RESOURCE_GROUP} \
     --name ${CLUSTER_NAME} \
     --attach-acr ${ACR_NAME}
   ```
   
   On Powershell:
   
   ```
   az aks update `
       --resource-group $RESOURCE_GROUP `
       --name $CLUSTER_NAME `
       --attach-acr $ACR_NAME
   ```

3. Use k9s to check (same as last chapter)
