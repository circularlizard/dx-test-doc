---
id: k8s-next-migrate-detail-dam
title: Operator migration - DAM
---

This section will documented a overview how it is possible to migrate a dam persistence and binaries from a operator based deployment to a HELM based deployment. Also a output of this design is a first draft for the raw documentation. This document may change after DBHA implementation.

## Migration-Mode for the HELM deployment

Before we can start to document the backup and restore of the dam persistence and binaries we should implemented a migration mode. The details of implementing migration mode will be available [here](https://pages.git.cwp.pnp-hcl.com/Team-Q/development-doc/docs/architecture/kube/k8s-next-migrate-detail-core).

## Backup and restore process
This section documents the backup and restore process of the dam persistence and binaries.

### Backup form the operator based deployment
#### 1. Ensure persistence(read write) and dam pod are running

The following command will help to find pods current status.
```
kubectl -n <namespace> get all
```

Ensure pods are up and running. If more than one dam and persistence pods are running, then scale down the pods to one.
#### 2. Database backup from persistence(read write)

a. Connect to persistence pod. The following command is used to connect with persistence container.
```
kubectl exec --stdin --tty pod/<pod-name> -n <namespace> -- /bin/bash
```
Example:
```
kubectl exec --stdin --tty pod/dx-deployment-persistence-0 -n dx-ns -- /bin/bash
```

b. Dump the current database using pg_dump.
```
[dx_user@dx-deployment-persistence-0 /]$ pg_dump dxmediadb > /tmp/dxmediadb.dmp
```

c. Download the dumped database to local system by using the following command.
```
kubectl cp <namespace>/<pod-name>:<source-file> <target-file>
```

Example
```
kubectl cp dx-ns/dx-deployment-persistence-0:/tmp/dxmediadb.dmp /backup/dxmediadb.dmp
```

#### 3. DAM binaries and profile backup

a. Connect to DAM pod. The following command is used to connect with container.
```
kubectl exec --stdin --tty pod/dx-deployment-dam-0 -n dx-ns -- /bin/bash
```

b. Compress the DAM binaries which are located under /opt/app/upload directory.
```
tar -cvpzf backupml.tar.gz --exclude=/backupml.tar.gz --one-file-system /opt/app/upload
```

c. Compress the DAM profiles which are located under /etc/config directory.
```
tar -cvpzf backupmlcfg.tar.gz --exclude=/backupmlcfg.tar.gz --one-file-system /etc/config
```

d. Download the compressed binaries and profiles to the local system.
```
kubectl cp dx-ns/dx-deployment-dam-0:/opt/app/server-v1/backupml.tar.gz /backup/backupml.tar.gz
kubectl cp dx-ns/dx-deployment-dam-0:/opt/app/server-v1/backupmlcfg.tar.gz /backup/backupmlcfg.tar.gz
```

### Restore to the HELM based deployment

#### 1. Start the HELM deployment

Before we can start with the restore we must ensure that the HELM based deployment is in correct state. Some adjustments are necessary.

- The extraction of kubernetes DX configuration from the operator based deployment to a valid `values.yaml` file is done.
- Enable the `migration` mode.

Now the HELM deployment can be started.
```
helm install -n <namespace> --create-namespace -f <values.yaml> <prefix> <chart>
```

Example:
```
helm install -n dxns-helm --create-namespace -f hcl-dx-deployment/value-sample.yaml dx-deployment hcl-dx-deployment
```

At the moment, the dam and persistence pods are alive. But application is not ready since migration mode is enabled.

#### 2. Restore the database

a. Upload the backup database to persistence pod.
```
kubectl cp /backup/dxmediadb.dmp dxns-helm/pod/dx-deployment-persistence-rw-0:/tmp/dxmediadb.dmp
```

b. Connect to persistence read write pod and perform the restore process.
```
kubectl exec --stdin --tty pod/dx-deployment-persistence-rw-0 -n dxns-helm -- /bin/bash
[dx_user@dx-deployment-persistence-rw-0 tmp]$ dropdb dxmediadb
[dx_user@dx-deployment-persistence-rw-0 tmp]$ createdb -O dxuser dxmediadb
[dx_user@dx-deployment-persistence-rw-0 tmp]$ psql dxmediadb < dxmediadb.dmp
```

#### 3. Restore the binaries and profiles

a. Upload the backup binary and profiles to dam pod.
```
kubectl cp /backup/backupml.tar.gz dxns-helm/pod/dx-deployment-digital-asset-management-0:/opt/app/server-v1/backupml.tar.gz
kubectl cp /backup/backupmlcfg.tar.gz dxns-helm/pod/dx-deployment-digital-asset-management-0:/opt/app/server-v1/backupmlcfg.tar.gz
```

b. Connect to dam pod and perform the restore process.
```
kubectl exec --stdin --tty pod/dx-deployment-digital-asset-management-0 -n dxns-helm -- /bin/bash
[dx_user@dx-deployment-digital-asset-management-0 server-v1]$ tar -xf /backupml.tar.gz --directory /opt/app/upload
[dx_user@dx-deployment-digital-asset-management-0 server-v1]$ tar -xf /backupmlcfg.tar.gz --directory /etc/config
[dx_user@dx-deployment-digital-asset-management-0 server-v1]$ rm /backupml.tar.gz
[dx_user@dx-deployment-digital-asset-management-0 server-v1]$ rm /backupmlcfg.tar.gz
```

#### 4. Disable the migration mode and the deployment

Before we can start with the final upgrade of the HELM deployment some adjustments are necessary.

- Disable the `migration` mode.
- Enable all relevant applications.

Now the HELM deployment can be upgraded.
```
helm upgrade -n <namespace> --create-namespace -f <values.yaml> <prefix> <chart>
```

Example:
```
helm upgrade -n dxns-helm --create-namespace -f hcl-dx-deployment/value-sample.yaml dx-deployment hcl-dx-deployment
```