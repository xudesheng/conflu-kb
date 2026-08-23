# From Standalone to High Availability

In the `test` deployment, it's a `standalone` Thingworx deployment + `Connection Server`. In the `test2` deployment, it's a `standalone` Thingworx deployment  + `eMC`.

We will discuss how to have a HA deployment for Thingworx. This document will high-light the major changes from **standalone** to **HA**, but it may not exhaust all options.



## Adding ZooKeeper

ZooKeeper is required in Thingworx HA deployment and the replica count should be even: 1, or 3 or 5. Typically, we choose 3 for non-production deployment and 5 for production deployment.

As usual, we need to add the related components in the `common.yaml` file.

```

  ${DEPLOYMENT_NAME}-zookeeper-tls:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: $CERTIFICATE_MANAGER_CHART
    version: "$CERTIFICATE_MANAGER_CHART_VERSION"
    description: "Zookeeper Certificates"
    group: zk
    priority: -950
    wait: true
    timeout: 600
    valuesFiles:
      - "deployment/zookeeper-certificate-manager.yaml"

  ${DEPLOYMENT_NAME}-zookeeper-tls-provisioning:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: $PROVISION_CHART
    version: "$PROVISION_CHART_VERSION"
    description: "Provision Zookeeper Secrets"
    group: zk
    priority: -1050
    wait: true
    valuesFiles:
      - "deployment/zookeeper-secrets.yaml"

  ${DEPLOYMENT_NAME}-zookeeper:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: "$ZOOKEEPER_CHART"
    version: "$ZOOKEEPER_CHART_VERSION"
    group: zk
    description: "Zookeeper"
    priority: -900
    wait: true
    timeout: 600
    valuesFiles:
      - "deployment/zookeeper.yaml"
```



## Adding Ignite

Ignite is the component for cache management in the Thingworx HA architecture. The replica count normally is 3

We need to enable all related components in the `common.yaml` file:

```yaml
  ${DEPLOYMENT_NAME}-ignite-tls:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: $CERTIFICATE_MANAGER_CHART
    version: "$CERTIFICATE_MANAGER_CHART_VERSION"
    description: "Ignite Certificates"
    group: ignite
    timeout: 1200
    priority: -900
    wait: true
    valuesFiles:
      - "deployment/ignite-certificate-manager.yaml"

  ${DEPLOYMENT_NAME}-ignite:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: $IGNITE_CHART
    version: "$IGNITE_CHART_VERSION"
    group: ignite
    description: "Apache Ignite"
    priority: -700
    wait: true
    timeout: 1200
    valuesFiles:
      - "deployment/ignite.yaml"
```



## Enable cluster mode in Thingworx

<img src="docs/14-from-standalone-to-ha/image-20250220020049416.png" alt="image-20250220020049416" style="zoom:50%;" />

## 14.4 Enable cluster mode in eMC and Connection Server

<img src="docs/14-from-standalone-to-ha/image-20250220020401003.png" alt="image-20250220020401003" style="zoom:50%;" />

<img src="docs/14-from-standalone-to-ha/image-20250220020448533.png" alt="image-20250220020448533" style="zoom:50%;" />

## Now you can start to deploy

```
helmsman -subst-env-values -f common.yaml -e common.env --apply
```



## Scale up and Scale down nodes

The "scale" here is to adjust the number of the Thingworx nodes, or replica count.

When you first time to scale up the replica count of the Thingworx node, you need to include the `tls` setup too.

For example, when you change the relica count of Thingworx from 2 to 3:

```
THINGWORX_REPLICA_COUNT: 3
```

You can use this command to redeploy:

```
helmsman -subst-env-values -f common.yaml -e common.env --apply  --group twx-tls --group twx --migrate-context
```

The above command tells you that:

1. it will execute the deployment file defined in the `twx-tls` group in the `common.yaml` file
2. it will execute the deployment file defined in the `twx` group too.
3. it will only change the thingworx deployment but not zookeeper, Ignite, eMC or connection server.



if you want to change from 3 to 2, or from 2 to 3 later, you ondly need to include the `twx` group since the `tls` has been provisioned already.

```shell
helmsman -subst-env-values -f common.yaml -e common.env --apply  --group twx --migrate-context
```

