# How Helm Chart Helps Thingworx Deployment

## Overview of Thingworx Platform Deployment

Thingworx platform deployment on Azure Kubernetes Service (AKS) involves multiple components that work together to ensure a scalable, reliable, and high-performance system. The deployment can be either Standalone (single node) or High Availability (HA) with multiple nodes. Let's explore the architecture and relationships between these components.

### High-Level Architecture

Below is a comprehensive architecture diagram showing the relationships between all components in a Thingworx HA deployment:

<img src="docs/11-helm-chart-and-twx-deployment/image-20250217181038482.png" alt="image-20250217181038482" style="zoom:50%;" />



For a Standalone deployment, the architecture would be simpler as it doesn't require ZooKeeper and Ignite:

<img src="docs/11-helm-chart-and-twx-deployment/image-20250224001630219.png" alt="image-20250224001630219" style="zoom:50%;" />


### 11.1.2 Deployment Dependencies and Relationships

The deployment of Thingworx involves several layers of dependencies:

<img src="docs/11-helm-chart-and-twx-deployment/image-20250224003212676.png" alt="image-20250224003212676" style="zoom:50%;" />



### Key Components and Their Roles

1. **Azure (Cloud Platform)**: The underlying cloud infrastructure that provides compute, storage, and networking resources.
2. **Kubernetes**: The container orchestration platform that manages the deployment, scaling, and operation of application containers.
3. **AKS (Azure Kubernetes Service)**: A managed Kubernetes service provided by Azure, simplifying cluster management.
4. **Azure Storage**: Provides persistent storage for Thingworx shared file repositories.
5. **Azure PostgreSQL Flexible Server**: The database service used by Thingworx Foundation for storing application data.
6. **Thingworx HA (High Availability)**: A deployment configuration that ensures the Thingworx platform remains available even if one or more nodes fail.
   - **ZooKeeper**: Coordinates and manages the state of the Thingworx HA cluster.
   - **Ignite**: Provides in-memory data grid capabilities for caching and distributed computing. Maintains bi-directional communication with Thingworx Foundation for data synchronization and state management.
   - **Connection Server**: Manages AlwaysOn device connections to the Thingworx platform.
   - **eMessage Connector**: Facilitates Axeda device communication to the Thingworx platform..
   - **Thingworx Foundation**: The core platform that provides the runtime environment for Thingworx applications.
   - **PostgreSQL Database for Thingworx Application**: The database used by Thingworx Foundation.
   - **Shared File Repo based on Azure Storage**: A shared file repository for storing platform-related files.
7. **Helm Chart**: A package manager for Kubernetes that simplifies the deployment and management of applications.
8. **Deployment Files**: YAML files that define the configuration and resources required for deploying each component.
9. **common.env, common.yaml**: Configuration files used by Helmsman to manage shared variables and deployment order.
10. **Cert Manager**: Automates the management and issuance of TLS certificates.
11. **Ingress Nginx**: Acts as a load balancer and manages external access to the services within the cluster.
12. **Monitoring Stack**: Provides monitoring and observability for the deployed applications.
13. **Public IP**: An Azure resource that provides the entry point for external users and devices to access the Thingworx platform through the Ingress controller.

### Helm Chart in Thingworx Deployment

#### Component Deployment Order

The deployment order is crucial, especially in HA deployments. Here's a visualization of the deployment sequence:

<img src="docs/11-helm-chart-and-twx-deployment/image-20250224003512705.png" alt="image-20250224003512705" style="zoom:50%;" />



### Managing Component Dependencies

The relationships between components are managed at multiple levels:

1. **Infrastructure Level**: Through Azure resource management
2. **Kubernetes Level**: Through Kubernetes services and networking
3. **Application Level**: Through Helm charts and their configurations
4. **Configuration Level**: Through shared variables in common.env and common.yaml

<img src="docs/11-helm-chart-and-twx-deployment/image-20250224003638850.png" alt="image-20250224003638850" style="zoom:50%;" />



### Deployment Files and Configuration

Each Helm Chart requires one or more deployment files to specify the exact configuration for the component. These files define parameters such as resource limits, environment variables, and service types.

#### Example: Thingworx Foundation Deployment File

```yaml
# deployment/thingworx.yaml
replicaCount: 1

image:
  registry: ${THINGWORX_IMAGE_REGISTRY}
  repository: ${THINGWORX_IMAGE}
  tag: ${THINGWORX_IMAGE_TAG}
  pullPolicy: ${IMAGE_PULL_POLICY}

dbinit:
  enabled: true
  registry: ${THINGWORX_IMAGE_REGISTRY}
  repository: ${THINGWORX_INIT_IMAGE}
  tag: ${THINGWORX_IMAGE_TAG}
  pullPolicy: ${IMAGE_PULL_POLICY}
  schema: postgres

scripts:
  registry: docker.io
  repository: alpine
  tag: "3.13.2"
  pullPolicy: ${IMAGE_PULL_POLICY}

# enable service monitor to collect metrics in prometheus
serviceMonitor:
  enabled: true
  certificateSecretRef: ${DEPLOYMENT_NAME}-twx-tls-cm-ca-certificate
```

### Managing Dependencies with Helmsman

Helmsman is a tool used to manage the deployment of multiple Helm Charts. It uses `common.env` and `common.yaml` files to manage shared configuration variables and define the order in which components should be deployed.

#### Example: common.env

```env
THINGWORX_CHART: thingworx-charts/thingworx-server
THINGWORX_CHART_VERSION: 1.0.118
THINGWORX_IMAGE: thingworx/platform-postgres
THINGWORX_INIT_IMAGE: thingworx/postgresql-init-twx
THINGWORX_IMAGE_TAG: java21.0.5-tomcat9.0.95-platform9.7.0-b34
```

#### Example: common.yaml

```yaml
apps:

  ${DEPLOYMENT_NAME}-twx-database-secrets-provisioning:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: $PROVISION_CHART
    version: "$PROVISION_CHART_VERSION"
    description: "Provision Thingworx secrets"
    group: twx-setup
    priority: -850
    wait: true
    valuesFiles:
      - "deployment/database-secrets.yaml"

  ${DEPLOYMENT_NAME}-twx-pvc-provisioning:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: $PROVISION_CHART
    version: "$PROVISION_CHART_VERSION"
    description: "Provision Thingworx PVC"
    group: twx-setup
    priority: -850
    wait: true
    valuesFiles:
      - "deployment/thingworx-pvc.yaml"

  ${DEPLOYMENT_NAME}-postgres:
    namespace: $DEPLOYMENT_NAMESPACE
    enabled: true
    chart: bitnami/postgresql
    version: "10.3.17"
    group: db
    description: "Postgresql"
    priority: -800
    wait: true
    timeout: 600
    valuesFiles:
      - "deployment/postgres.yaml"
```

### Summary:

Deploying Thingworx on AKS involves a combination of Helm Charts, deployment files, and configuration management tools like Helmsman. Understanding the relationships between these components and how they interact is crucial for a successful deployment. Whether you're deploying a standalone Thingworx instance or a high-availability cluster, Helm Charts provide a consistent and repeatable way to manage the deployment process.

## 11.2 Next Steps

In the following chapters, we will explore different deployment configurations in detail:
- Pod-based Database vs Azure PostgreSQL Flexible Server
- Thingworx Standalone vs HA Deployment
- Connection Server vs eMessage Connector

Each configuration will build upon the architectural foundations discussed in this chapter while maintaining the same basic relationships between components.

| SN    | Name                                                         | Note                                                         |
| ----- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| test0 | thingworx standalone with pod based Postgresql               | suitable for simple development.                             |
| test1 | thingworx standalone with Postgresql Flexible Server+ Connection Server | suitable for development and standalone production (AlwaysOn device) |
| Test2 | thingworx HA with Postgresql Flexible Server + Connection Server | suitable for HA QA and HA production (AlwaysOn device)       |
| Test3 | thingworx standalone with Postgresql Flexible Server + eMessage Connector | suitable for development and standalone production (Axeda device) |
| test4 | thingworx HA with Postgresql Flexible Server + eMessage Connector | suitable for HA QA and HA production (Axeda device)          |



