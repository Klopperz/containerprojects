**Initialize variables**
```
$resourceGroupName = "owncloud"
$uniqueID = (new-guid).Guid.Substring(0,8)
$cfTunnelToken = "xxx"
$containerGroup = $uniqueID + $resourceGroupName +"cg"
$registryName = $uniqueID + $resourceGroupName +"acr"
$keyvaultName = $uniqueID + $resourceGroupName +"kv"
$storageName = $uniqueID + $resourceGroupName +"storage"
$logAnalyticsName = $uniqueID + $resourceGroupName +"logs"
$containerName = "owncloud"
$imageName = "owncloud/server"
$imageTag = "latest"
$location = "West Europe"
$azLoggedinUserID = (az ad signed-in-user show | convertfrom-json).id
$azSubscription = (az account show | convertfrom-json).id
```

**Create Resourcegroup**
```
az group create --name $resourceGroupName --location $location
```

**Create Keyvault**
```
az keyvault create --name $keyvaultName `
  --location $location `
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
  --sku Standard `
  --location $location

az acr identity assign `
  --name $registryName `
  --identities [system]

az acr update `
  --name $registryName `
  --admin-enabled true

$registryPassword = (az acr credential show `
  --name $registryName | ConvertFrom-Json).passwords[0].value

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
  --target-repo "$containerName" `
  --cred-set docker

az acr cache create --registry $registryName `
  --resource-group $resourceGroupName `
  --name cloudflared `
  --source-repo "docker.io/cloudflare/cloudflared" `
  --target-repo "cloudflared" `
  --cred-set docker
```

**Create storage account**
```
az storage account create `
  --name $storageName `
  --resource-group $resourceGroupName `
  --kind StorageV2 `
  --sku Standard_LRS `
  --location $location `
  --min-tls-version TLS1_2

az storage share create `
  --account-name $storageName `
  --name owncloud

$storageKey = (az storage account keys list `
  --resource-group $resourceGroupName `
  --account-name $storageName | ConvertFrom-Json)[1].value
```

**Create workspace**
```
az monitor log-analytics workspace create `
  --name $logAnalyticsName `
  --resource-group $resourceGroupName `
  --location $location

$workspaceKey = (az monitor log-analytics workspace get-shared-keys `
  --resource-group $resourceGroupName `
  --workspace-name $logAnalyticsName | convertfrom-json).primarySharedKey

$workspaceId = (az monitor log-analytics workspace show `
  --name ba518070owncloudlogs `
  --resource-group owncloud | ConvertFrom-Json).customerId
```


**Create containers**
```
$containerDefinitionFile = "owncloud.yml"
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<containerGroupName>",$containerGroup))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<imgRegistryHostName>",$registryName))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<imgRegistryUserName>",$registryName))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<imgRegistryPassword>",$registryPassword))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<imageName>",$imageName))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<CfTunnelToken>",$cfTunnelToken))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<storagename>",$storageName))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<storagekey>",$storageKey))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<workspaceId>",$workspaceId))
set-content -path $containerDefinitionFile -value ((get-content -path $containerDefinitionFile).replace("<workspaceKey>",$workspaceKey))

az container create `
  --resource-group $resourceGroupName `
  --file owncloud.yml
```

**Destroy containers**
```
az container delete `
  --name $containerGroup `
  --resource-group $resourceGroupName `
  --yes
```