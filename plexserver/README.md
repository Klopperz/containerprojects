**Get account key**
```
ACCOUNT_KEY=$(az storage account keys list \
  --resource-group <resourcegroup> \
  --account-name persistantstorage \
  --query "[0].value" -o tsv)
  ```

**Install Plex server**
```
az container create --resource-group <resourcegroup> --file plexserver.yml
```

**Destroy Plex server**
```
az container delete --name plexserver --resource-group Plex
```

**Get workspace keys**
```
az monitor log-analytics workspace get-shared-keys \
  --resource-group Plex \
  --workspace-name plexserver \
  --query primarySharedKey -o tsv
```
