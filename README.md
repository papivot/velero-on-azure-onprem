# Installing Velero on Azure (and using Minio to forward on-prem backup to the AZ storage)

1. Install Velero on the Azure cluster

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

```bash 
export MINIO_ROOT_USER=storage-name
export MINIO_ROOT_PASSWORD=wuiU....Z7w==
minio gateway azure --address 10.197.107.62:9000
```
