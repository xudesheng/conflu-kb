# Change Database, Add eMC and more

In this chapter, I will explain how to change the database from a pod to a Azure Postgresql Flexible Server and how to add the `eMessage Connector` to the deployment.

**Caution**: from now on, all helm-charts used in the deployment is from the OCI repository. You have to login to the repository by following the command in the previous chapters before you use the `helmsman` command.

## Change Database

### The current database setting

The first setting is in the `common.yaml`, the postgres deployment is enabled and the relevant deployment file is `deployment/progres.yaml`.

<img src="docs/13-change-database-and-add-emc/image-20250219112310078.png" alt="image-20250219112310078" style="zoom:50%;" />

The concrete setting for a Postgresql pod deployment is defined in the `deployment/postgres.yaml` file. The demanded postgresql version and credentials are defined here.

<img src="docs/13-change-database-and-add-emc/image-20250219112402312.png" alt="image-20250219112402312" style="zoom:50%;" />

Finally, in the `deployment/thingworx.yaml` file, the host of the database is the service name of the postgresql pod.

<img src="docs/13-change-database-and-add-emc/image-20250219112546851.png" alt="image-20250219112546851" style="zoom:50%;" />

So, if we want to change the database, we need to change all of the above settings.

### How the new settings should be

Let's move to the `project/test1` folder.

In order to change the database from a pod deployed Postgresql to a Azure Postgresql Flexible server, we need to make the following change:

1. make sure you have created the database based on the instructions in chapter #9

2. disable the postgres deployment in the `common.yaml` file

   <img src="docs/13-change-database-and-add-emc/image-20250219131439872.png" alt="image-20250219131439872" style="zoom:50%;" />

3. Change the DB host in the `thingworx.yaml` and let it point to your Flexible server

   **Caution**: due to historical reason, the `zure` should be disabled but the `rds` should be enabled when choosing Flexible server. If your database is not the Flexible server on Azure but just Azure Postgresql, please enable the `azure` and disable the `rds`.

   **Caution**: if you are using a AWS RDS Postgresql, the `azure` option should be disabled and the `rds` should be enabled, same as a Flexible server on Azure.

   <img src="docs/13-change-database-and-add-emc/image-20250219141056516.png" alt="image-20250219141056516" style="zoom:50%;" />

   <img src="docs/13-change-database-and-add-emc/image-20250219141141824.png" alt="image-20250219141141824" style="zoom:50%;" />

4. Now you can start to deploy a new one.

   Please make sure you are under the `project/test` folder:

   ```
   helmsman -subst-env-values -f common.yaml -e common.env --apply
   ```

   

5. Destroy

   when you use the destroy command to detroy a deployment, the database inside the Azure Postgresql Flexible Server will not be deleted. You need to delete it manually from Azure Portal or from the Azure CLI. (https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server/db?view=azure-cli-latest#az-postgres-flexible-server-db-delete)

   ```shell
   helmsman -subst-env-values -f common.yaml -e common.env --destroy
   ```
   
   ```shell
   az postgres flexible-server db delete --database-name test1 -g $RESOURCE_GROUP --server-name $DB_SERVER_NAME
   ```
   
   



## Add InfluxDB

In the `test` case, the `InfluxDB` has been added for demo purposes. The `influxDB` deployment is only a pod based, but you can change it to an enterprise `InfluxDB` deployment or a `InfluxDB` SaaS offering.

1. enable `InfluxDB` in the `common.yaml` file

   <img src="docs/13-change-database-and-add-emc/image-20250219152032361.png" alt="image-20250219152032361" style="zoom:50%;" />

2. Setup persistent provider in Thingworx for the InfluxDB

   <img src="docs/13-change-database-and-add-emc/image-20250219174325065.png" alt="image-20250219174325065" style="zoom:50%;" />

   

## Add Connection Server

**Caution**: You can configue to have more than 1 connection server even the Thingwox is configured as `standalone`.

1. Enable multiple components in the `common.yaml` file:

   1. Connection server needs certificates when talking with Thingworx.

      <img src="docs/13-change-database-and-add-emc/image-20250219174722615.png" alt="image-20250219174722615" style="zoom:50%;" />

   2. Connection Server needs a `AppKey` to talk with Thingworx, so, we need to provision it as Kubernetes secret.

      <img src="docs/13-change-database-and-add-emc/image-20250219175053112.png" alt="image-20250219175053112" style="zoom:50%;" />

   3. We need a `Graphite Exporter` service to convert Connection Server metrics into `Prometheus` format.

      <img src="docs/13-change-database-and-add-emc/image-20250219174835007.png" alt="image-20250219174835007" style="zoom:50%;" />

   4. We need to provision the connection server entities inside Thingworx

      <img src="docs/13-change-database-and-add-emc/image-20250219174935011.png" alt="image-20250219174935011" style="zoom:50%;" />

   5. We need to deploy connection server at the end

      <img src="docs/13-change-database-and-add-emc/image-20250219175152824.png" alt="image-20250219175152824" style="zoom:50%;" />

2. We need to tell Connection Server that the `Thingworx` node is running in `standalone` mode

   <img src="docs/13-change-database-and-add-emc/image-20250219175308155.png" alt="image-20250219175308155" style="zoom:50%;" />

3. Finally, how many Connection Server instances we want to have

   <img src="docs/13-change-database-and-add-emc/image-20250219175429682.png" alt="image-20250219175429682" style="zoom:50%;" />

   <img src="docs/13-change-database-and-add-emc/image-20250219175517084.png" alt="image-20250219175517084" style="zoom:50%;" />

   

4. Under the `deployment/alwayson` folder, you can find many configuration options.



## Add eMemssage Connector (eMC)

Let's go to the folder `project/test2`

Adding eMC will be similar to adding the Connection Server. However, eMC needs 3 extra extensions to be loaded on the Thingworx side to work correctly. In the meantime, eMC customers normally will use the `Software Content Management`, or SCM functionality. The `SCM` is an extension and it requires the `Thingworx Utility Core` to be installed first.

So, besides the steps to add eMC, we will also talk about how to add extensions to the deployment and how to control the order when loading the extensions.

1. Make sure you have built docker images for the 5 extensions. The 5 extensions are:

   1. (**TUC**)Thingworx Utility Core
   2. (**SCM**)Software Content Management
   3. (**CSE**)Connection Service Extension
   4. (**RAE**)Remote Access Extension
   5. (**ACE**)Axeda Compatibility Extension

2. Define the name and tag of each image in the `common.env` file

   ```
   EMESSAGE_CSE_EXTENSIONS_IMAGE: thingworx/emc-cse
   EMESSAGE_CSE_EXTENSIONS_IMAGE_TAG: 2.5.0
   EMESSAGE_RAE_EXTENSIONS_IMAGE: thingworx/emc-rae
   EMESSAGE_RAE_EXTENSIONS_IMAGE_TAG: 3.5.0
   EMESSAGE_ACE_EXTENSIONS_IMAGE: thingworx/emc-ace
   EMESSAGE_ACE_EXTENSIONS_IMAGE_TAG: 6.2.3
   EMESSAGE_TUC_EXTENSIONS_IMAGE: thingworx/emc-tuc
   EMESSAGE_TUC_EXTENSIONS_IMAGE_TAG: 9.7.0-b8
   EMESSAGE_SCM_EXTENSIONS_IMAGE: thingworx/emc-scm
   EMESSAGE_SCM_EXTENSIONS_IMAGE_TAG: 9.7.0-b8
   ```

3. Define the import order of all extensions in the `deployment/thingworx.yaml` file

   <img src="docs/13-change-database-and-add-emc/image-20250220012344449.png" alt="image-20250220012344449" style="zoom:50%;" />

4. Enable the required components for eMC in the `common.yaml` file

   Please take a look at the `common.yaml` file for the following sections:

   ```
   ${DEPLOYMENT_NAME}-emsg-tls
   ${DEPLOYMENT_NAME}-emsg-secrets-provisioning
   ${DEPLOYMENT_NAME}-emsg-metrics
   ${DEPLOYMENT_NAME}-emsg-entity-provisioning
   ${DEPLOYMENT_NAME}-emsg
   ```

   It's similar to what we have done when adding the Connection Server

5. Disable the cluster mode in the `deployment/emessage/main.yaml` file, same as adding the Connection Server

6. Define how many replica count needed for the eMC in `common.env` and `deployment/emessage/main.yaml`. Please check variable **EMESSAGE_REPLICA_COUNT**

7. Define the ingress for eMC

   <img src="docs/13-change-database-and-add-emc/image-20250220013217659.png" alt="image-20250220013217659" style="zoom:50%;" />

8. Now you can start to deploy:

   ```
   helmsman -subst-env-values -f common.yaml -e common.env --apply
   ```

   <img src="docs/13-change-database-and-add-emc/image-20250220013404129.png" alt="image-20250220013404129" style="zoom:33%;" />



