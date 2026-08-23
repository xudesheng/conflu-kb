# Upgrading ThingWorx

## Step 1: Preparing docker images for the Upgrade

Before you begin the upgrade process, ensure that you have the necessary Docker image for the targed version of ThingWorx. Normally, you need to upgrade Thingworx and Ignite together if you are running on HA mode. The `Connection Server` and `eMC` has less dependencies on Thingworx version but it may need to be upgraded together, please check the compatability matrix.



## Step 2: Updating Configuration Files

The next step involves updating the configuration files to reflect the new version of ThingWorx.

- **Modify `common.env` File**:
  - Locate the `common.env` file in your configuration directory.
  
  - Update the version number to 9.7. This ensures that all services use the correct version of ThingWorx.
  
  - Example:
    ```plaintext
    THINGWORX_VERSION=9.7
    ```
    
    

## Step 3: Scale down stateful set if necessary 

If you only upgrade minor version of Thingworx, then you don't need this step.

If you need to upgrade non-minor version, or you have to upgrade multiple componnents together, especially Thingworx and Ignite, you'd better to scale down the stateful set of the following components together:

- Thingworx statefulset:
  
- Ignite statefulset
  
- ZooKeeper statefulset
  
- Connection Server statefulset if you have
  
- eMC statefulset if you have
  
  <img src="docs/16-upgrading-thingworx/image-20250221111704665.png" alt="image-20250221111704665" style="zoom:50%;" />
  
  You may have a different value in each statefulset. But you need to change all of them to 0
  
  <img src="docs/16-upgrading-thingworx/image-20250221112040241.png" alt="image-20250221112040241" style="zoom:50%;" />
  
  If you don't use `k9s` tool, you can also use the following two commands to archive the same goal:
  
  * list the statefulset in a namespace
  
    ```
    kubectl get statefulset -n <namespace>
    ```
  
  * scale down the stateful set to 0
  
    ```
    kubectl scale statefulset <statefulset-name> --replicas=0 -n <namespace>
    ```
  
    You have to repeat for all statefulset in that namespace

## Step 4: Upgrade 

You can use the `apply` command, just like a regular deployment, to perform the upgrade:

```Shell
helmsman -subst-env-values -f common.yaml -e common.env --apply
```



## Step 5: Verifying the Upgrade

After the upgrade, it's crucial to verify that the new version is running correctly and that all functionalities are intact.

- **Post-Upgrade Verification**:
  - Access the ThingWorx platform through the web interface.
  
  - Check the version number in the platform's About section to confirm the upgrade.
  
  - Test critical functionalities to ensure they are working as expected.
  
    

## Additional Considerations

- **Backup**: Always create a backup of your current ThingWorx data and configurations before starting the upgrade. This ensures that you can restore your system in case of any issues.
- **Compatibility**: Verify that all custom extensions and integrations are compatible with ThingWorx 9.7. You may need to update or replace incompatible components.

By following these steps, you can successfully upgrade your ThingWorx platform. Ensure that you have a rollback plan in place in case any issues arise during the upgrade process.
