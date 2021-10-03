# Velero on vSphere with Tanzu (Source Cluster) and Azure (Destination Cluster)

## Azure configuration (Destination Cluster)

1. Create Storage (optional)

```bash
# Create AZURE RESOURCE GROUP

AZURE_BACKUP_RESOURCE_GROUP=kubernetes
az group create -n $AZURE_BACKUP_RESOURCE_GROUP --location eastus
```

```bash
# Create STORAGE ACCOUNT (if not present already) 
AZURE_STORAGE_ACCOUNT_ID="velero$(uuidgen | cut -d '-' -f5 | tr '[A-Z]' '[a-z]')"
az storage account create \
--name $AZURE_STORAGE_ACCOUNT_ID \
--resource-group $AZURE_BACKUP_RESOURCE_GROUP \
--sku Standard_GRS \
--encryption-services blob \
--https-only true \
--kind BlobStorage \
--access-tier Hot

# Get the AZURE_STORAGE_ACCOUNT_ID (if already exits)
AZURE_STORAGE_ACCOUNT_ID=`az storage account list  --query '[0].name' -o tsv`
```

```bash
BLOB_CONTAINER=velero

# Create Blob container (if it does not exist already)
az storage container create -n $BLOB_CONTAINER\
--public-access off \
--account-name $AZURE_STORAGE_ACCOUNT_ID
```

2. Prepare secret file 

```bash
# Create secrets files

AZURE_SUBSCRIPTION_ID=`az account list --query '[?isDefault].id' -o tsv`
AZURE_TENANT_ID=`az account list --query '[?isDefault].tenantId' -o tsv`
AZURE_CLIENT_SECRET=`az ad sp create-for-rbac \
--name "velero-navneet" \
--role "Contributor" \
--query 'password' -o tsv \
--scopes /subscriptions/$AZURE_SUBSCRIPTION_ID`

### remember to save the secret as you cannot retrieve it back 

AZURE_CLIENT_ID=`az ad sp list --display-name "velero-navneet" --query '[0].appId' -o tsv`

cat << EOF  > ./credentials-velero
AZURE_SUBSCRIPTION_ID=${AZURE_SUBSCRIPTION_ID}
AZURE_TENANT_ID=${AZURE_TENANT_ID}
AZURE_CLIENT_ID=${AZURE_CLIENT_ID}
AZURE_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
AZURE_RESOURCE_GROUP=${AZURE_BACKUP_RESOURCE_GROUP}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF
```

3. Install Velero on the Azure cluster

```bash
velero install \
--provider azure \
--plugins velero/velero-plugin-for-microsoft-azure \
--bucket $BLOB_CONTAINER \
--secret-file ./credentials-velero \
--backup-location-config   resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID,storageAccount=$AZURE_STORAGE_ACCOUNT_ID \
--snapshot-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID \
--features=EnableAPIGroupVersions \
--default-volumes-to-restic \
--use-restic \
--wait
```

4. Validate that the install is successful and the backup location is available 
```bash
$ kubectl get pods -n velero    

NAMESPACE     NAME                                                              READY   STATUS    RESTARTS   AGE

velero        restic-6ft8c                                                      1/1     Running   0          30s
velero        velero-98f54bb54-lwxhm                                            1/1     Running   0          30s

$ velero backup-location get                                                                                                                                           
NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   azure      velero          Available   2021-10-03 20:06:44 +0000 UTC   ReadWrite     true

```
---
## vSphere with Tanzu configuration (Source cluster)

1. Copy the `credentials-velero` from the Azure install (see above) to the jump box where the cluster will be configured. 

2. Install velero on the source cluster using the same environment variables as above - 

```bash
velero install \
--provider azure \
--plugins harbor.navneetv.com/proxy_cache/velero/velero-plugin-for-microsoft-azure \
--bucket velero \
--image harbor.navneetv.com/proxy_cache/velero/velero:v1.7.0 \
--secret-file ./credentials-velero \
--backup-location-config   resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID,storageAccount=$AZURE_STORAGE_ACCOUNT_ID \
--snapshot-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID \
--features=EnableAPIGroupVersions \
--default-volumes-to-restic \
--use-restic
```
Note the difference in how the images are referenced using Harbor's Proxy cache feature (if you are encountering the Docker rate-limiting issue). Modify the value accordingly to use a private registry.

```
--plugins harbor.navneetv.com/proxy_cache/velero/velero-plugin-for-microsoft-azure
--image   harbor.navneetv.com/proxy_cache/velero/velero:v1.7.0 
```

3. Validate that the install is successful and the backup location is available 
```bash
$ kubectl get pods -n velero    

NAMESPACE     NAME                                                              READY   STATUS    RESTARTS   AGE

velero        restic-mfs9v                                                      1/1     Running     0          18s
velero        restic-qqgt8                                                      1/1     Running     0          18s
velero        velero-564f9f8b9b-gcbpz                                           1/1     Running     0          18s

$ velero backup-location get                                                                                                                                       

NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   azure      velero          Available   2021-10-03 16:45:47 -0400 EDT   ReadWrite     true
```

---
## Backup on the source cluster
1. Leveraging the example provided in the velero documentation, install the application with PVs. 

2. Create a backup 
```shell
$ velero backup create nginx-backup --include-namespaces nginx-example                                                                                                                                           ─╯
Backup request "nginx-backup" submitted successfully.
Run `velero backup describe nginx-backup` or `velero backup logs nginx-backup` for more details.

$ velero backup logs nginx-backup
...

$ velero backup get    

NAME           STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
nginx-backup   Completed   0        0          2021-10-03 17:14:44 -0400 EDT   29d       default            <none>

$ velero backup describe nginx-backup --details                                                                                                                           
Name:         nginx-backup
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  velero.io/source-cluster-k8s-gitversion=v1.19.7+vmware.1
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=19

Phase:  Completed

Errors:    0
Warnings:  0

Namespaces:
  Included:  nginx-example
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  <none>

Storage Location:  default

Velero-Native Snapshot PVs:  auto

TTL:  720h0m0s

Hooks:  <none>

Backup Format Version:  1.1.0

Started:    2021-10-03 17:14:44 -0400 EDT
Completed:  2021-10-03 17:15:08 -0400 EDT

Expiration:  2021-11-02 17:14:44 -0400 EDT

Total items to be backed up:  14
Items backed up:              14

Resource List:
  apps/v1/Deployment:
    - nginx-example/nginx-deployment
  apps/v1/ReplicaSet:
    - nginx-example/nginx-deployment-7676db6c4d
  discovery.k8s.io/v1beta1/EndpointSlice:
    - nginx-example/my-nginx-vfcqp
  rbac.authorization.k8s.io/v1/RoleBinding:
    - nginx-example/cs-rid-org-a0f60cc5-58dc-448a-8f17-a30fc58f7bb9-rbac.authorization.k8s.io-ClusterRole-view
  rbac.authorization.k8s.io/v1beta1/RoleBinding:
    - nginx-example/cs-rid-org-a0f60cc5-58dc-448a-8f17-a30fc58f7bb9-rbac.authorization.k8s.io-ClusterRole-view
  v1/ConfigMap:
    - nginx-example/istio-ca-root-cert
  v1/Endpoints:
    - nginx-example/my-nginx
  v1/Namespace:
    - nginx-example
  v1/PersistentVolume:
    - pvc-30b644d0-09da-4f12-bec0-06ab6070e014
  v1/PersistentVolumeClaim:
    - nginx-example/nginx-logs
  v1/Pod:
    - nginx-example/nginx-deployment-7676db6c4d-ghws5
  v1/Secret:
    - nginx-example/default-token-zlb6c
  v1/Service:
    - nginx-example/my-nginx
  v1/ServiceAccount:
    - nginx-example/default

Velero-Native Snapshots: <none included>

Restic Backups:
  Completed:
    nginx-example/nginx-deployment-7676db6c4d-ghws5: nginx-logs

```


---
## Restore on the destination cluster

To make sure that Restic volumes get mapped to the correct destination storage class, create a config map similar to the one below on the destination cluster before performing a restore. 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: change-storage-class-config
  namespace: velero
  labels:
    velero.io/plugin-config: ""
    velero.io/change-storage-class: RestoreItemAction
data:
  # add 1+ key-value pairs here, where the key is the old
  # storage class name and the value is the new storage
  # class name.
  tanzu: default
  
```

1. Restore from the backup that was cresated previosuly. 
```
$ velero restore create --from-backup nginx-backup

Restore request "nginx-backup-20211003212637" submitted successfully.
Run `velero restore describe nginx-backup-20211003212637` or `velero restore logs nginx-backup-20211003212637` for more details.
```

2. Validate the application is running successfully 

```
kubectl get pods -A 

NAMESPACE       NAME                                                              READY   STATUS    RESTARTS   AGE
...
nginx-example   nginx-deployment-7676db6c4d-ghws5                                 2/2     Running   0          110s
```

---

### additonal steps (if using Minio as a Azure blob gateway)

3. Install Minio as a gateway on the on-prem environment

```bash 
AZURE_STORAGE_ACCOUNT_KEY=`az storage account keys list --account-name ${AZURE_STORAGE_ACCOUNT_ID} |jq -r '.[0].value'`

export MINIO_ROOT_USER=${AZURE_STORAGE_ACCOUNT_ID}
export MINIO_ROOT_PASSWORD=${AZURE_STORAGE_ACCOUNT_KEY}
minio gateway azure --address 10.197.107.62:9000
```
