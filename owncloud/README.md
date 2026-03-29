**Initialize variables**
```
$resourceGroupName = "owncloud"
$registryName = "$resourceGroupName"+"reg"
$keyvaultName = "$resourceGroupName"+"keyv"
$containerName = "owncloud"
$imageName = "owncloud/ocis"
$imageTag = "latest"
$azLoggedinUserID = (az ad signed-in-user show | convertfrom-json).id
$azSubscription = (az account show | convertfrom-json).id
```

**Create Resourcegroup**
```
az group create --name $resourceGroupName --location "west europe"
```

**Create Keyvault**
```
az keyvault create --name $keyvaultName `
  --location "West Europe" `
  --resource-group "$resourceGroupName"
az role assignment create --role "Key Vault Secrets Officer" `
  --assignee $azLoggedinUserID `
  --scope "/subscriptions/$azSubscription/resourceGroups/$resourceGroupName/providers/Microsoft.KeyVault/vaults/$keyvaultName"
```

**Set dockerhub credentials**
```
$name = Read-Host "Please enter your dockerhub accountname"
$password = Read-Host "Please enter your dockerhub password"
az keyvault secret set --vault-name $keyvaultName `
  --name "docker-username" `
  --value $name
az keyvault secret set --vault-name $keyvaultName `
  --name "docker-password" `
  --value $password
```

**Create container registry**
```
az acr create --name $registryName `
  --resource-group $resourceGroupName `
  --sku Standard

az acr identity assign --name $registryName `
  --identities [system]

$kvUsernameId="https://$keyvaultName.vault.azure.net/secrets/docker-username"
$kvPasswordId="https://$keyvaultName.vault.azure.net/secrets/docker-password"

az acr credential-set create `
  --name docker `
  --resource-group $resourceGroupName `
  --registry $registryName `
  --login-server docker.io `
  --username-id $kvUsernameId `
  --password-id $kvPasswordId

$acrCacheCredIdentity = (az acr credential-set show `
  --name docker `
  --registry $registryName | ConvertFrom-Json).identity.principalId

az role assignment create --role "Key Vault Secrets User" `
  --assignee $acrCacheCredIdentity `
  --scope "/subscriptions/$azSubscription/resourceGroups/$resourceGroupName/providers/Microsoft.KeyVault/vaults/$keyvaultName"

az acr cache create --registry $registryName `
  --resource-group $resourceGroupName `
  --name $containerName `
  --source-repo "docker.io/$imageName" `
  --target-repo "$imageName" `
  --cred-set docker
```