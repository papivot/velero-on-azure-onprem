# Installing Velero on Azure (and using Minio to forward on-prem backup to the AZ storage)

```bash 
export MINIO_ROOT_USER=storage-name
export MINIO_ROOT_PASSWORD=wuiU....Z7w==
minio gateway azure --address 10.197.107.62:9000
```
