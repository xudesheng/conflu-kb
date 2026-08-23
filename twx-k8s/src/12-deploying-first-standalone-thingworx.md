# Deploying First Thingworx

During this test, it is assumed that you are in the `test0` folder.

## Understand the folder structure

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250218234542662.png" alt="image-20250218234542662" style="zoom:50%;" />

Under `test0` folder, there are two subfolders: `deployment` folder is where all of deployment files sit in. `thingworx-charts` folder is the helm-charts for all necessary components.

There are also some `.env` files and `.yaml` files. We will explain that later.

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250218234615241.png" alt="image-20250218234615241" style="zoom:50%;" />

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250218234641774.png" alt="image-20250218234641774" style="zoom:50%;" />

### 12.1.1 Which components should be involved in the deployment.

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250218235245063.png" alt="image-20250218235245063" style="zoom:50%;" />

In the `common.yaml` file, the `enabled` field defines whether the component will be deployed or not. In the above example, `twx` and `twx-keystore-setup` will be deployed, but `twx-ingress-tls-secrets-provisioning` will be ignored.

The `priority` defines the order of execution, the lower the earlier. So, the `twx-keystore-setup` will be executed before the deployment of `twx`.

### Where those varaibles are defined

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250218235639395.png" alt="image-20250218235639395" style="zoom:50%;" />

The variables used in the `common.yaml` file are defined in the `common.env` file.



### Where the dependency to the infrastructure are defined.

We know the Thingworx deployment has depedencies, like the storage, ingress and database. 

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250218235914909.png" alt="image-20250218235914909" style="zoom:50%;" />

The dependency to the `ingress` is defined in the `thingworx.yaml` file under the `deployment` folder.

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219000018795.png" alt="image-20250219000018795" style="zoom:50%;" />

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219000042186.png" alt="image-20250219000042186" style="zoom:50%;" />

The dependency to the `storage` is defined in the `thingworx-pvc.yaml` file and also the `common.env` file. From this example, you can see that the `storageClass` can be directly defined in the `thingworx-pvc.yaml` file, or it can be a variable in the `thingworx-pvc.yaml` file and with a value in the `common.env` file. 

You can use either way to handle the variables. If you prefer to using some script language to drive the deployment, puting the value in the `common.env` may be more convincing.

The dependency to the database will be discussed later.

### How the thingworx charts are located

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219002326021.png" alt="image-20250219002326021" style="zoom:50%;" />

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219002428629.png" alt="image-20250219002428629" style="zoom:50%;" />

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219002508469.png" alt="image-20250219002508469" style="zoom:50%;" />

In the `common.yaml` file, the `helmRepos` defines the name of the repo URL you can reference. If you look at the `postgres` section, the chart is `bitnami/postgresql`. The `helm` tool will go to use the url defined in the `helmRepos` section to retrive the helm chart.

On ther other hand, the `twx-pvc-provisioning` section is trying to access `thingworx-charts/provisioning`. But the `thingworx-charts` has been commented out in the `helmRepos` section. So, the `helm` tool will look for the `provisioning` chart in a local folder with name `thingworx-charts`.

There are other two ways to locate the helm-charts and I will explain it later.



## Prepare a license before your test

Please get a Thingworx license file and make sure it includes the support of HA if you want to test HA later.

Then please use the following command to convert it to a base64 encoded string.

On Linux:

```
base64 -w 0 license.bin
```

On Mac:

```
base64 license.bin | tr -d '\n'
```

On Powershell:

```
[Convert]::ToBase64String([IO.File]::ReadAllBytes("license.bin"))
```

Then please copy/paste the base64 encoded string in the `common.env` file.

<img src="docs/12-deploying-first-standalone-thingworx/image-20250221013033621.png" alt="image-20250221013033621" style="zoom:50%;" />

You have to do this for the `common.yaml` file in all test folders.



## Dry-run, Deploy and Destroy

### Dry-run

```
helmsman -subst-env-values -f common.yaml -e common.env --dry-run
```

The `helmsman` command receives both yaml files (`-f`) and enviromen files (-e) as the input. You can provide multiple files:

```
helmsman -subst-env-values -f yaml1.yaml -f yaml2.yaml -e env1.env -e env2.env -e env3.env --dry-run
```

When multiple files are provided, the later one will overwrite the ealier one for the same key.

At the end, the output looks like:

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219005214040.png" alt="image-20250219005214040" style="zoom:50%;" />

### Deploy

```
helmsman -subst-env-values -f common.yaml -e common.env --apply
```

The above command will start to deploy a Thingworx instance. The server name generated will be in the pattern `${DEPLOYMENT_NAME}-thingworx-${DEPLOYMENT_NAMESPACE}.${DOMAIN}`, which is defined in the `thingworx.yaml` file under the `deployment` folder.

 <img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219013458590.png" alt="image-20250219013458590" style="zoom:50%;" />



<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219011446070.png" alt="image-20250219011446070" style="zoom:50%;" />

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219011535164.png" alt="image-20250219011535164" style="zoom:50%;" />

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219011629564.png" alt="image-20250219011629564" style="zoom:50%;" />

### Detroy a deployment

```
helmsman -subst-env-values -f common.yaml -e common.env --destroy
```



## How to use a HTTP helmchart repository

The `common-legacy.env` and `common-legacy.yaml` demonstrate how to use a HTTP helmchart repository:

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219014612892.png" alt="image-20250219014612892" style="zoom:50%;" />

![image-20250325022516171](docs/12-deploying-first-standalone-thingworx/image-20250325022516171.png)

In the above two files, the `thingworx-charts` will refer to the HTTP repository from the URL: `https://ptcinc.github.io/thingworx-charts/` .

Before you can use this repository in deployment, you need to add the login info of the repository to the local helm repo (Please see how to add in the chapter #4.4.1 )

Now you can start to deploy:

```
helmsman -subst-env-values -f common-legacy.yaml -e common-legacy.env --apply
```



## How to use a OCI helmchart repository

We can use the OCI helmchart repository defined in chapter #4 in deployment. 

<img src="docs/12-test0-deploying-first-standalone-thingworx/image-20250219015117290.png" alt="image-20250219015117290" style="zoom:50%;" />

What you need to do is to replace the `thingworx-charts` in the `.env` file with the OCI full path, here it is: `oci://dxudemocr.azurecr.io/helm`, the yaml file doesn't need a change.

Before you use the OCI repository, you need to login after you load all system variables:

On Linux/Mac:

```
HELM_USERNAME="00000000-0000-0000-0000-000000000000"
login_json=$(az acr login -n "$ACR_NAME" -g "$RESOURCE_GROUP" --only-show-errors --expose-token --output json)
access_token=$(echo "$login_json" | jq -r '.accessToken')
echo "$access_token" | helm registry login "${ACR_NAME}.azurecr.io" --username "$HELM_USERNAME" --password-stdin
```

On Powershell:

```
$HELM_USERNAME = "00000000-0000-0000-0000-000000000000"
$login_json = az acr login -n $env:ACR_NAME -g $env:RESOURCE_GROUP --expose-token --only-show-errors --output json | ConvertFrom-Json
$access_token = $login_json.accessToken
$access_token | helm registry login "$env:ACR_NAME.azurecr.io" --username $HELM_USERNAME --password-stdin
```



Now you can use the following command to deploy:

```
helmsman -subst-env-values -f common.yaml -e common-oci.env --apply
```

