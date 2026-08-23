# Preparation

**Caution**: many companies  require you create resources with proper tags. For each command here, you can add tags as you wish.



## Load OS variables everytime when you open a new console

### Using Environment Variables in Kubernetes
During the entire serials, it is assumed that you are under the working directory in the console. Please ensure to load the OS variables from the **.env** file every time.

On Linux/Mac:

```shell
. ./helper/load-envfile.sh
```
***Caution***: In the above command, it's a "**dot**", then a "**space**", and then "**./helper/load-envfile.sh**".

Or:

```shell
source ./helper/load-envfile.sh
```

Or:

```shell
export $(grep -v '^#' .env | xargs)
```



To validate:

```Shell
# Verify environment variables are set
printenv | grep -E 'RESOURCE_GROUP|LOCATION|VNET_NAME'
```



On Windows Powershell:

```Powershell
helper\Load-EnvFile.ps1
```

To validate:

```Powershell
# Verify environment variables are set
Get-ChildItem Env: | Where-Object { $_.Name -match 'RESOURCE_GROUP|LOCATION|VNET_NAME' }
```



### Make sure you have authenticated with Azure:

On Linux/Mac:

```shell
# Authenticate with Azure using tenant ID
az login --tenant ${TENANT_ID}
```

On Powershell:

```Powershell
# Authenticate with Azure using tenant ID
az login --tenant $env:TENANT_ID
```



Verify on Linux/Mac/Powershell:

```shell
az account show
```



## Create Resource Group

On Linux/Mac:

```Shell
az group create --name $RESOURCE_GROUP --location $LOCATION
```
The output looks like:

![image-20250205162211999](docs/02-resourcegroup_vnet_subnet/image-20250205162211999.png)

On Powershell:
```Powershell
az group create --name $env:RESOURCE_GROUP --location $env:LOCATION
```

You can search the resource group name and see it in the Azure Portal
![image-20250205162528575](docs/02-resourcegroup_vnet_subnet/image-20250205162528575.png)



## Create VNET and Subnets

### Create VNET

On Linux/Mac:

```shell
az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --name $VNET_NAME \
    --address-prefixes $VNET_ADDRESS_PREFIX \
    --location $LOCATION
```

On Powershell:

```Powershell
az network vnet create `
    --resource-group $env:RESOURCE_GROUP `
    --name $env:VNET_NAME `
    --address-prefixes $env:VNET_ADDRESS_PREFIX `
    --location $env:LOCATION
```

The command line output looks like:

<img src="docs/02-resourcegroup_vnet_subnet/image-20250206000536379.png" alt="image-20250206000536379" style="zoom:50%;" />

### Create Default Subnet

On Linux/Mac:

```Shell
az network vnet subnet create \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $DEFAULT_SUBNET_NAME \
    --address-prefixes $DEFAULT_SUBNET_PREFIX \
    --service-endpoints Microsoft.Storage Microsoft.Sql
```

On Powershell:

```Powershell
az network vnet subnet create `
    --resource-group $env:RESOURCE_GROUP `
    --vnet-name $env:VNET_NAME `
    --name $env:DEFAULT_SUBNET_NAME `
    --address-prefixes $env:DEFAULT_SUBNET_PREFIX `
    --service-endpoints Microsoft.Storage Microsoft.Sql
```

The output looks like:

<img src="docs/02-resourcegroup_vnet_subnet/image-20250207183943374.png" alt="image-20250207183943374" style="zoom:50%;" />

### Create Subnet for database

On Linux/Mac:

```Shell
az network vnet subnet create \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $FLEXIBLE_DB_SUBNET_NAME \
    --address-prefixes $FLEXIBLE_DB_SUBNET_PREFIX \
    --delegations "Microsoft.DBforPostgreSQL/flexibleServers"
```

On Powershell:
```Powershell
az network vnet subnet create `
    --resource-group $env:RESOURCE_GROUP `
    --vnet-name $env:VNET_NAME `
    --name $env:FLEXIBLE_DB_SUBNET_NAME `
    --address-prefixes $env:FLEXIBLE_DB_SUBNET_PREFIX `
    --delegations "Microsoft.DBforPostgreSQL/flexibleServers"
```



The output looks like:

<img src="docs/02-resourcegroup_vnet_subnet/image-20250207164508855.png" alt="image-20250207164508855" style="zoom:50%;" />

### Create subnet for nodes

On Linux/Mac:

```Shell
az network vnet subnet create \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $NODE_SUBNET_NAME \
    --address-prefixes $NODE_SUBNET_PREFIX \
    --delegations "Microsoft.ContainerInstance/containerGroups"
```

On Powershell:

```Powershell
az network vnet subnet create `
    --resource-group $env:RESOURCE_GROUP `
    --vnet-name $env:VNET_NAME `
    --name $env:NODE_SUBNET_NAME `
    --address-prefixes $env:NODE_SUBNET_PREFIX `
    --delegations "Microsoft.ContainerInstance/containerGroups"
```

The output looks like:

<img src="docs/02-resourcegroup_vnet_subnet/image-20250207164625822.png" alt="image-20250207164625822" style="zoom:50%;" />

### Validate

You can search the vnet name in the azure portal and check the subnets under the vnet. The UI looks like:

<img src="docs/02-resourcegroup_vnet_subnet/image-20250207164725189.png" alt="image-20250207164725189" style="zoom:50%;" />
