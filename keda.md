# Serverless in Kubernetes with KEDA and Azure Functions

## Installing KEDA and creating Azure Function

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

## Deploying Azure Function to Kubernetes
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

## Serverless HTTP scaling
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
