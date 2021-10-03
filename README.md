# Installing Velero on Azure (and using Minio to forward on-prem backup to the AZ storage)

1. Create Storage (optional)

```bash
# Create AZURE RESOURCE GROUP

AZURE_BACKUP_RESOURCE_GROUP=kubernetes
az group create -n $AZURE_BACKUP_RESOURCE_GROUP --location eastus
```

```bash
# Create STORAGE ACCOUNT

AZURE_STORAGE_ACCOUNT_ID="velero$(uuidgen | cut -d '-' -f5 | tr '[A-Z]' '[a-z]')"
az storage account create \
--name $AZURE_STORAGE_ACCOUNT_ID \
--resource-group $AZURE_BACKUP_RESOURCE_GROUP \
--sku Standard_GRS \
--encryption-services blob \
--https-only true \
--kind BlobStorage \
--access-tier Hot

ZURE_STORAGE_ACCOUNT_ID=`az storage account list  --query '[0].name' -o tsv`
```

```bash
# Create Blob container

BLOB_CONTAINER=velero
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

2. Install Velero on the Azure cluster

```bash
velero install \
--provider azure \
--plugins velero/velero-plugin-for-microsoft-azure \
--bucket $BLOB_CONTAINER \
--secret-file ./credentials-velero \
--backup-location-config   resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID,storageAccount=$AZURE_STORAGE_ACCOUNT_ID \
--snapshot-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID \
--use-restic \ 
--wait
```

3. Install Minio as gateway on the on-prem enviornment

```bash 
AZURE_STORAGE_ACCOUNT_KEY=`az storage account keys list --account-name ${AZURE_STORAGE_ACCOUNT_ID} |jq -r '.[0].value'`

export MINIO_ROOT_USER=${AZURE_STORAGE_ACCOUNT_ID}
export MINIO_ROOT_PASSWORD=${AZURE_STORAGE_ACCOUNT_KEY}
minio gateway azure --address 10.197.107.62:9000
```
