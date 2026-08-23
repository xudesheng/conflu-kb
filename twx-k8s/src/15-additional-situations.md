# Additional Deployment Options



## Setup logback

You can define your logging logic in a new logback.xml file and deploy it with the helm-charts together.

1. define your logback.xml file

2. copy and paste the content into `deployment/logback_map.xml` file, please be careful to handle the ident carefully. The name will be used later.

   <img src="docs/15-additional-situations/image-20250221013639277.png" alt="image-20250221013639277" style="zoom:50%;" />

3. add or enable the `configmaps-provisioning` section in the `common.yaml` file

   <img src="docs/15-additional-situations/image-20250221013559323.png" alt="image-20250221013559323" style="zoom:50%;" />

4. enable the logback override config map in the `deployment/thingworx.yaml` file

   <img src="docs/15-additional-situations/image-20250221013759956.png" alt="image-20250221013759956" style="zoom:50%;" />

5. Redeploy Thingworx Group



## Adjust pod resource

1. Thingworx Pod Resource

   <img src="docs/15-additional-situations/image-20250221014811633.png" alt="image-20250221014811633" style="zoom:50%;" />

   The resource is defined in the `deployment/thingworx.yaml` file. You can add it if it's not there.

   The value of each varabile is defined in `common.env` file

   ```
   TWX_CPU_REQUEST: 4
   TWX_CPU_LIMIT: 5
   TWX_MEMORY_REQUEST: 16Gi
   TWX_MEMORY_LIMIT: 20Gi
   THINGWORX_CATALINA_OPTS: "-Xmx14G -Xms14G -Dsun.io.useCanonCaches=false -Dsun.io.useCanonPrefixCache=false"
   ```

   Please make sure to adjust the `THINGWORX_CATALINA_OPTS` and make sure the `-Xms` and `-Xmx` configuration aligned with the your memory  settings.

   

2. Ignite Pod Resource

   <img src="docs/15-additional-situations/image-20250221015110485.png" alt="image-20250221015110485" style="zoom:50%;" />

   The Ignite pod resource is defined in the `deployment/ignite.yaml` file.

   ```
   IGNITE_CPU_REQUEST: 4
   IGNITE_CPU_LIMIT: 4
   IGNITE_MEMORY_REQUEST: 16Gi
   IGNITE_MEMORY_LIMIT: 20Gi
   ```

   

3. ZooKeeper Pod Resource

   The ZooKeeper pod resource is defined in the `deployment/zookeeper.yaml` file

   <img src="docs/15-additional-situations/image-20250221015306441.png" alt="image-20250221015306441" style="zoom:50%;" />

   ```
   ZOOKEEPER_CPU_REQUEST: 1
   ZOOKEEPER_CPU_LIMIT: 1500m
   ZOOKEEPER_MEMORY_REQUEST: 2Gi
   ZOOKEEPER_MEMORY_LIMIT: 3Gi
   ```

   

4. Connection Server Pod Resource

   The Connection Server pod resource is defined in `deployment/alwayson/main.yaml` file

   <img src="docs/15-additional-situations/image-20250221015514061.png" alt="image-20250221015514061" style="zoom:50%;" />

   ```
   CXSERVER_CPU_LIMIT: 2
   CXSERVER_MEMORY_LIMIT: 8Gi
   CXSERVER_CPU_REQUEST: 1
   CXSERVER_MEMORY_REQUEST: 3Gi
   ```

   

5. eMC Pod Resource

   The eMC Pod Resource is defined in `deployment/emessage/main.yaml`

   <img src="docs/15-additional-situations/image-20250221015720308.png" alt="image-20250221015720308" style="zoom:50%;" />

   ```
   EMSG_CPU_LIMIT: 2
   EMSG_MEMORY_LIMIT: 4Gi
   EMSG_CPU_REQUEST: 2
   EMSG_MEMORY_REQUEST: 4Gi
   EMESSAGE_JAVA_OPTS: " -Xms512m -Xmx1536m"
   ```

   

## Setup secret for container registry access

If your container registry requires credential to access insider the cluster, you need to create a secret before you deploy.

For example, if you are going to deploy test4. In the demo, your namespace will be `test4`

```shell
# create namespace manually, replace test4 with your namespace
kubectl create namespace test4
kubectl create secret docker-registry acr-secret \
	--docker-server=$ACR_NAME.azurecr.io \
	--docker-username=$ACR_NAME \
	--docker-password=1QEKFtumw62FsKe4bxcgg5gmYJgK5+okr+ACRDizRDU \
	--docker-email=youremail@email.com \
	-n test4
```

The docker-password can be obtained from container registry portal page: Settings -> Access keys -->password

Then go to enable the following two lines in all of the files list below:

```
imagePullSecrets:
  - name: acr-secret
```



    deployment/ignite.yaml
    deployment/zookeeper-certificate-manager.yaml
    deployment/thingworx-certificate-manager.yaml
    deployment/alwayson.yaml
    deployment/alwayson/main.yaml
    deployment/emessage/certificate-manager.yaml
    deployment/alwayson-certificate-manager.yaml
    deployment/alwayson/certificate-manager.yaml
    deployment/emessage/main.yaml
    deployment/keystore.yaml
    deployment/akka-tls/certificate-manager.yaml
    deployment/certificate-manager/main.yaml
    deployment/thingworx.yaml
