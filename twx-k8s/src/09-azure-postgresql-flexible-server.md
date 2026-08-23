# Azure PostgreSQL Flexible Server

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



## Create a Postgresql Flexible Server

1. Go to your resource group in Azure Portal
    <img src="docs/09-azure-postgresql-flexible-server/image-20250209011452569.png" alt="image-20250209011452569" style="zoom:50%;" />

2. Search for `Azure Database for PostgreSQL - Flexible Server`

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209011728462.png" alt="image-20250209011728462" style="zoom:50%;" />

3. **Basics**

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209011848831.png" alt="image-20250209011848831" style="zoom:40%;" />

4. **High Availability and password**

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209012019611.png" alt="image-20250209012019611" style="zoom:40%;" />

5. **Networking**

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209012207465.png" alt="image-20250209012207465" style="zoom:40%;" />

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209012255583.png" alt="image-20250209012255583" style="zoom:40%;" />

6. **Security**

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209012322473.png" alt="image-20250209012322473" style="zoom:50%;" />

7. **Tags**

   You can enter any tag and value here.

8. **Final Result**

   Please take a note of the "Server Name", it will be used in the later test.

   <img src="docs/09-azure-postgresql-flexible-server/image-20250209013202748.png" alt="image-20250209013202748" style="zoom:30%;" />

9. **Change Database Spec if needed**

   You can change the **Compute** setting to get more database power

<img src="docs/09-azure-postgresql-flexible-server/image-20250209013335564.png" alt="image-20250209013335564" style="zoom:40%;" />

## validate the endpoint connectivity

How to validate the endpoint connectivity:

1. Start a "busybox" pod:

```Shell
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
```

2. try to use `nslookup` command to resolve the database full name to IP address


```Shell
nslookup $DB_SERVER_NAME.postgres.database.azure.com
```

<img src="docs/09-azure-postgresql-flexible-server/image-20250210222214055.png" alt="image-20250210222214055" style="zoom:30%;" />



<img src="docs/09-azure-postgresql-flexible-server/image-20250210222312574.png" alt="image-20250210222312574" style="zoom:50%;" />

3. Make sure the IP address you got from the above step is the same in this step.

<img src="docs/09-azure-postgresql-flexible-server/image-20250210222551334.png" alt="image-20250210222551334" style="zoom:50%;" />



## How to validate the database connectivity:

This is my example:

```Shell
kubectl run -it --rm pg-client --image=postgres --restart=Never --env='PGPASSWORD=PleaseChangeMeNow2025!' -- psql -h dxudemoflex.postgres.database.azure.com -U postgres -d postgres
```


You can use your server name and password to validate.

```
kubectl run -it --rm pg-client --image=postgres --restart=Never --env="PGPASSWORD=${DB_ADMIN_PASSWORD}" -- psql -h ${DB_SERVER_NAME}.postgres.database.azure.com -U ${DB_ADMIN_USER} -d postgres
```

<img src="docs/09-azure-postgresql-flexible-server/image-20250210223553879.png" alt="image-20250210223553879" style="zoom:50%;" />

Type `\l` to list the current databases.

Type `\q` to quit.

<img src="docs/09-azure-postgresql-flexible-server/image-20250210223103342.png" alt="image-20250210223103342" style="zoom:50%;" />

Done!

