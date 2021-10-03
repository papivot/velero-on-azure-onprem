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

### remember to save the secret as you cannot retrive it back 

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

4. Validate that the Install is successfull and the backup location is available 
```bash
$ kubectl get pods -n velero    

NAMESPACE     NAME                                                              READY   STATUS    RESTARTS   AGE

velero        restic-6ft8c                                                      1/1     Running   0          30s
velero        velero-98f54bb54-lwxhm                                            1/1     Running   0          30s

$ velero backup-location get                                                                                                                                           
NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   azure      velero          Available   2021-10-03 20:06:44 +0000 UTC   ReadWrite     true

```

## vSphere with Tanzu configuration (Source cluster)

1. Copy the `credentials-velero` from the Azure install (see above) to the jumpbox where the cluster will be configured. 

2. Install velero on the source cluster using the same enviornmental variables as above - 

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
Note the difference in how the images are referenced using Harbor's Proxy cache feature (if you are encountering the Docker rate limiting issue). Modify the value accordingly to use a private registry.

```
--plugins harbor.navneetv.com/proxy_cache/velero/velero-plugin-for-microsoft-azure
--image   harbor.navneetv.com/proxy_cache/velero/velero:v1.7.0 
```

3. Validate that the Install is successfull and the backup location is available 
```bash
$ kubectl get pods -n velero    

NAMESPACE     NAME                                                              READY   STATUS    RESTARTS   AGE

velero                         restic-mfs9v                                                          1/1     Running     0          18s
velero                         restic-qqgt8                                                          1/1     Running     0          18s
velero                         velero-564f9f8b9b-gcbpz                                               1/1     Running     0          18s

$ velero backup-location get                                                                                                                                       

NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   azure      velero          Available   2021-10-03 16:45:47 -0400 EDT   ReadWrite     true
```



---


3. Install Minio as gateway on the on-prem enviornment

```bash 
AZURE_STORAGE_ACCOUNT_KEY=`az storage account keys list --account-name ${AZURE_STORAGE_ACCOUNT_ID} |jq -r '.[0].value'`

export MINIO_ROOT_USER=${AZURE_STORAGE_ACCOUNT_ID}
export MINIO_ROOT_PASSWORD=${AZURE_STORAGE_ACCOUNT_KEY}
minio gateway azure --address 10.197.107.62:9000
```
