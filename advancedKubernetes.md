## Advanced Kubernetes (optional section for attendies using Kubernetes already)

### Azure Application Gateway as Ingress
Azure Application Gateway is advanced reverse proxy solution with Web Application Firewall capabilities and enterprise-grade configuration and certificate store (eg. via Azure Key Vault). You can use Application Gateway as your Ingress implementation in AKS so rules are deployed as part of your Kubernetes environment (eg. with Helm).

Ingress controller will need access to configure Azure Application Gateway. As this runs in Pod, you currently cannot use service principal from AKS itself. You may either use AAD Pod Identity (solution to securely access AAD Identity from your Pods) or provide controller with secrets for service principal. Recommended is first option, but for simplicity we will try second option.

Let's prepare app gateway.

```powershell
$aksResourceGroup = az aks show -n yourname-aks -g cp-aks --query nodeResourceGroup -o tsv
$aksVnet = az network vnet list -g $aksResourceGroup --query [0].name -o tsv
az network vnet subnet create -g $aksResourceGroup `
    --vnet-name $aksVnet `
    -n appGwSubnet `
    --address-prefixes 10.100.0.0/24

az network public-ip create -n appgwip `
    -g $aksResourceGroup `
    --allocation-method Static `
    --sku Standard

az network application-gateway create -n appgw `
    -g $aksResourceGroup `
    --sku Standard_v2 `
    --public-ip-address appgwip `
    --vnet-name $aksVnet `
    --subnet appGwSubnet
```

Create service principal and assign RBAC for app gateway as Contributor and to resource group as Reader.

```powershell
$principalSecret = az ad sp create-for-rbac -n "http://appGwForAKS" --sdk-auth --skip-assignment
$encodedPrincipalSecret = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($principalSecret))

az role assignment create --role Contributor `
    --assignee $(az ad sp list --spn "http://appGwForAKS" --query [0].appId -o tsv) `
    --scope $(az network application-gateway show -n appgw -g $aksResourceGroup --query id -o tsv)

az role assignment create --role Reader `
    --assignee $(az ad sp list --spn "http://appGwForAKS" --query [0].appId -o tsv) `
    --scope $(az group show -n $aksResourceGroup --query id -o tsv)
```

Use Helm to deploy and configure Ingress controller for Application gateway

```powershell
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/

helm repo update

$aksAPI = az aks show -n yourname-aks -g cp-aks --query fqdn -o tsv
$subscriptionId = az account show --query id -o tsv

helm install application-gateway-kubernetes-ingress/ingress-azure `
     --name ingress-azure `
     --namespace default `
     --set appgw.name=appgw `
     --set appgw.resourceGroup=$aksResourceGroup  `
     --set appgw.subscriptionId=$subscriptionId `
     --set appgw.shared=false `
     --set armAuth.type=servicePrincipal `
     --set armAuth.secretJSON=$encodedPrincipalSecret `
     --set rbac.enabled=true `
     --set kubernetes.watchNamespace=default `
     --set aksClusterConfiguration.apiServerAddress=$aksAPI 
```

We are now ready to expose todo application via Ingress object using Application Gateway.

```powershell
kubectl apply -f ingressAppGw.yaml
kubectl get ingress
```

Test access to todo app via Application Gateway IP that you found with previous kubectl command or in GUI of Application Gateway. Also walk throw Application Gatewat configuration to see how Ingress controller configured policies there.

### Serverless in Kubernetes with KEDA and Azure Functions

#### Installing KEDA and creating Azure Function

Install Azure Functions local environment according to guide here: [https://github.com/azure/azure-functions-core-tools#installing](https://github.com/azure/azure-functions-core-tools#installing)

```powershell
mkdir keda-test
cd keda-test

func init . --docker --worker-runtime node --language javascript

# Create new Azure Function and select Azure Queue Storage Trigger
func new
```

Create storage account and queue

```powershell
$storageName = "myuniquekedastorage"
az storage account create --sku Standard_LRS -g cp-aks -n $storageName
$connectionString = az storage account show-connection-string --n $storageName --query connectionString -o tsv
az storage queue create -n myqueue --connection-string $connectionString
```

Let's configure our local Azure Function environment. Open local.settings.json and put storage connection string as value for AzureWebJobsStorage. Then go to function floder (QueueTrigger was default) and modify function.json to have "AzureWebJobsStorage" as value for connection and myqueue as queu name.

Modify host.json file to look like this:

```json
{
    "version": "2.0",
    "extensionBundle": {
        "id": "Microsoft.Azure.Functions.ExtensionBundle",
        "version": "[1.*, 2.0.0)"
    }
}
```
Let's test our Function locally. Start it, wait for build and then got to Storage Explorer (app or in portal) and create new message. Check that Function processing it.

```powershell
func start
```

You should see your Function is working locally.

#### Deploying Azure Function to Kubernetes
 Now let's install KEDA in AKS and push our Function there. Make sure you are authenticated to our Azure Container Registry.

```powershell
func kubernetes install --namespace keda

cd keda-test/
$registryName = "cpacrbn7n3upudvjcq"
az acr login -n $registryName

func kubernetes deploy --name keda-test `
    --registry $registryName".azurecr.io" `
    --max-replicas 10 `
    --cooldown-period 30

# If you prefer to build container manualy you can then use: func kubernetes deploy --name keda-test --image-name $registryName".azurecr.io"/func:v1
```

Watch Pods in defautl namespace and use portal or Storage Explorer to create message in queue. You will see Pod comming up, processing request and after defined period of time get deleted.

```powershell
kubectl get pods -w
```

#### Serverless HTTP scaling
Standard scaling with Kubernetes Horizontal Pod Autoscaler is great for services that are always running and need to scale based on load. Nevertheless some services are access rarely and you might want to use serverless solution by scaling number of instances to 0 when there is no request and start Pod only when needed. This can be achieved with Osiris and combine with AKS cluster autoscaler can help you save costs.

Func keda installer has also privisioned Osiris project for us in your AKS cluster. Let's create new Azure Function with HTTP Trigger.

```powershell
mkdir keda-webapi
cd keda-webapi

func init . --docker --worker-runtime node --language javascript

# Create new Azure Function and select HTTP Trigger
func new
```

We will keep default source code so let's just build container and deploy function to AKS.

```powershell
$registryName = "cpacrbn7n3upudvjcq"
az acr login -n $registryName

func kubernetes deploy --name keda-webapi `
    --registry $registryName".azurecr.io" `
    --max-replicas 10 `
    --cooldown-period 30
```

Get public IP address of service.

```powershell
kubectl get service
```

Wait for some time (about 5 minutes) and you will see there are no serverless-http Pods running in your cluster.

```powershell
kubectl get pods -w
```

Open public IP in browser and watch Pod being created and request completed. Wait 5 minutes (default) and Pod will get deleted.

### Distributed Application Runtime

#### Install DAPR CLI
First install DAPR CLI

```
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
```

#### Install DAPR in AKS
From DAPR CLI deploy solution to AKS

```
dapr init --kubernetes 
```

#### Prepare external components
First we will prepare external PaaS components to be used as backend for DAPR services.

```
export resourceGroup=akstemp
```

##### Provision CosmosDB account, database and container
```
export cosmosdbAccount=mujdaprcosmosdb
az cosmosdb create -n $cosmosdbAccount -g $resourceGroup
az cosmosdb sql database create -a $cosmosdbAccount -n daprdb -g $resourceGroup
az cosmosdb sql container create -g $resourceGroup -a $cosmosdbAccount -d daprdb -n statecont -p "/id"
```

##### Provision Azure Service Bus
```
export servicebus=mujdaprservicebus
az servicebus namespace create -n $servicebus -g $resourceGroup
az servicebus topic create -n orders --namespace-name $servicebus -g $resourceGroup
az servicebus namespace authorization-rule create --namespace-name $servicebus \
  -g $resourceGroup \
  --name daprauth \
  --rights Send Listen Manage
```

##### Provision blob storage
```
export storageaccount=mujdaprstorageaccount
az storage account create -n $storageaccount -g $resourceGroup --sku Standard_LRS --kind StorageV2
export storageConnection=$(az storage account show-connection-string -n $storageaccount -g $resourceGroup --query connectionString -o tsv)
az storage container create -n daprcontainer --connection-string $storageConnection
```

##### Privision Event Hub
```
export eventhub=mujdapreventhub
az eventhubs namespace create -g $resourceGroup -n $eventhub --sku Basic
az eventhubs eventhub create -g $resourceGroup --namespace-name $eventhub -n dapreventhub --message-retention 1
az eventhubs eventhub authorization-rule create \
  -g $resourceGroup \
  --namespace-name $eventhub \
  --eventhub-name dapreventhub \
  -n daprauth \
  --rights Listen Send
```

#### Provision DAPR components
DAPR uses custom resource Component to configure backend implementation for its various services. In order to easily pass connection strings and other details without manually modifying YAMLs we will use Helm 3 to install it. Please note we are not following best practices here - secret values should be passed via Secret object, but we are not doing that for simplicity reasons.

```
cd dapr
helm upgrade dapr-components ./dapr-components --install \
  --set cosmosdb.url=$(az cosmosdb show -n $cosmosdbAccount -g $resourceGroup --query documentEndpoint -o tsv) \
  --set cosmosdb.masterKey=$(az cosmosdb keys list -n $cosmosdbAccount -g $resourceGroup --type keys --query primaryMasterKey -o tsv) \
  --set cosmosdb.database=daprdb \
  --set cosmosdb.collection=statecont \
  --set serviceBus.connectionString=$(az servicebus namespace authorization-rule keys list --namespace-name $servicebus -g $resourceGroup --name daprauth --query primaryConnectionString -o tsv) \
  --set blob.storageAccount=$storageaccount \
  --set blob.key=$(az storage account keys list -n $storageaccount -g $resourceGroup --query [0].value -o tsv) \
  --set blob.container=daprcontainer \
  --set eventHub.connectionString=$(az eventhubs eventhub authorization-rule keys list --namespace-name $eventhub -g $resourceGroup --eventhub-name dapreventhub --name daprauth --query primaryConnectionString -o tsv)
```

#### State store example
To demo state management deploy pod1.yaml and check DAPR has injected side-car container. Then jump to container and use curl to call DAPR on loopback to store and read key.

```
kubectl apply -f pod1.yaml

kubectl describe pod pod1 | grep Image:
  Image:         tkubica/mybox
  Image:         docker.io/daprio/dapr:latest

kubectl exec -ti pod1 -- bash

curl -X POST http://localhost:3500/v1.0/state \
  -H "Content-Type: application/json" \
  -d '[
        {
          "key": "00-11-22",
          "value": "Tomas"
        }
      ]'

curl http://localhost:3500/v1.0/state/00-11-22
"Tomas"

kubectl delete -f pod1.yaml
```

#### Pub/Sub example
For publish/subscribe demo first deploy pod1.yaml and keep window open. In new window deploy python1.yaml and jump to it using python process. Copy and paste application sub.py. That is explosing endpoint into which DAPR will send messages should they arrive. In pod1.yaml use curl to send message.

```
kubectl apply -f pod1.yaml
kubectl exec -ti pod1 -- bash

curl -X POST http://localhost:3500/v1.0/publish/orders \
	-H "Content-Type: application/json" \
	-d '{
       	     "orderCreated": "ABC01"
      }'
exit

kubectl apply -f python.yaml
kubectl exec -ti python1 -- python

kubectl delete -f python.yaml
kubectl delete -f pod.yaml
```

#### Output binding to blob storage
In this example we will test output binding with Azure Blob Storage. Use DAPR API to create file in Blob storage.

```
kubectl apply -f pod1.yaml
kubectl exec -ti pod1 -- bash

curl -X POST http://localhost:3500/v1.0/bindings/binding-blob \
	-H "Content-Type: application/json" \
	-d '{ "metadata": {"blobName" : "myfile.json"}, 
      "data": {"mykey": "This is my value"}}'
exit

export storageConnection=$(az storage account show-connection-string -n $storageaccount -g $resourceGroup --query connectionString -o tsv)
az storage blob list -c daprcontainer -o table --connection-string $storageConnection

kubectl delete -f pod1.yaml
```

#### Input binding from Event Hub
In this example we will have input binding from Azure Event Hub. Run Python container, jump to python process and copy and paste code in binding.py. Generate some event in Event Hub and watch your code being called by DAPR and message content passed.

```
kubectl apply -f python.yaml
kubectl exec -ti python1 -- python

# Paste Python app binding.py and generate some event in Event Hub
```

### Service Mesh Interface
TBD

## Contacts

### Tomas Kubica - Cloud Solutions Architect
- https://www.linkedin.com/in/tkubica
- https://github.com/tkubica12
- https://twitter.com/tkubica

### Jaroslav Jindrich - Cloud Solutions Architect
- https://www.linkedin.com/in/jjindrich
- https://github.com/jjindrich
- https://twitter.com/jjindrich_cz
