# Azure Technical Academy - Hybrid IT Day 3: containers

## Prerequisities
- All day1 and day2 labs completed and knowledge of all topics covered
- Homework from day2 completed and shared with instructors via private message on Teams
- Access to own Azure subscription as Owner (access to single Resource Group is not enough for this lab)
- Rights to create Service Principal in AAD or precreated Service Principal with credentials
- Sufficient quota in subscription
  - 10 or more total vCPUs in region West Europe
  - 10 or more B-series VM vCPUs in region West Europe
- Precreated Azure DevOps organization with full rights for purpose of this Lab, instructions in [documentation](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops)

## Prepare Azure services for application

### Deploy Azure services manually

Check scripts in [arm-scripts](arm-scripts) and update parameter values.

```powershell
#$uniqueId = "-" + [system.environment]::MachineName
$uniqueId = ""
cd .\arm-scripts\
```

Run this script to deploy Azure Container Registry

```powershell
$rgArtifacts="cp-deployment-artifacts"+$uniqueId
az group create -l westeurope -n $rgArtifacts
az group deployment create -g $rgArtifacts `
    --template-file deploy-acr.json `
    --parameters deploy-acr.parameters.json
```

Run this script to deploy Azure Sql database

```powershell
$rgSql="cp-sql"+$uniqueId
az group create -l westeurope -n $rgSql
az group deployment create -g $rgSql `
    --template-file deploy-sql.json `
    --parameters deploy-sql.parameters.json --parameters administratorLoginPassword=Azure123
```

Run this script to deploy Azure WebApp for Containers with Windows Containers with reference to Azure Container Registry.
Check parameters file for correct ACR name and how ACR referenced in ARM template.

```powershell
$rgWebWin="cp-web-win"+$uniqueId
az group create -l westeurope -n $rgWebWin
az group deployment create -g $rgWebWin `
    --template-file deploy-webwin.json `
    --parameters deploy-webwin.parameters.json
```

### Deploy Azure services with Azure DevOps

Utilize your knowledge from Day2 and do following steps:

1. Create new DevOps Project AcademyDay3
2. Clone this Github repo into Azure Repos https://github.com/azurecz/azuretechacademy-hybridit-labs-day3.git and name it _academy_source
3. Create new Release pipeline CPWEB-Services-CD
4. Configure pipeline to deploy Azure services with ARM deployment - artifacts, Sql (-administratorLoginPassword Azure123), Web for Windows

**Sample of one task**
![DevOps task 1](media/devops-services.png)

**Sample of definition for Sql deployment task**

```yaml
steps:
- task: AzureResourceGroupDeployment@2
  displayName: 'ARM sql deploy'
  inputs:
    azureSubscription: 'YOUR-SUBSCRIPTION'
    resourceGroupName: 'cp-sql'
    location: 'West Europe'
    csmFile: '$(System.DefaultWorkingDirectory)/_academy_source/arm-scripts/deploy-sql.json'
    csmParametersFile: '$(System.DefaultWorkingDirectory)/_academy_source/arm-scripts/deploy-sql.parameters.json'
    overrideParameters: '-administratorLoginPassword Azure123'
```

## Building and storing container images with Azure Container Registry

We will deploy sample Todo web application from https://github.com/tkubica12/dotnetcore-sqldb-tutorial

### Build docker image with Azure DevOps

We will push Linux and Windows image into Azure Container Registry using Azure DevOps.

First clone source code to Azure Repos

1. Import git https://github.com/tkubica12/dotnetcore-sqldb-tutorial.git into new Azure Repos repository

Steps to create new release pipeline for Windows deployment

1. Create new Release pipeline CPWEBWINDOWS-CD
2. Create DEV stage
3. Add artefact referencing source code dotnetcore-sqldb-tutorial and name it _source
4. Make sure Agent job running windows-2019
5. Add Docker task command buildAndPublish
    - add Docker registry pointed to Azure Container Repository cpacr
    - container registry type cpweb
    - select dockerfile
    - tags $(Release.DeploymentID)-windows

Repeat steps for Linux deployment with Release pipeline name CPWEBLINUX-CD

1. Add new Agent job running Ubuntu and repeat same steps
    - tags $(Release.DeploymentID)-linux

**Sample of definition for Docker build task**

```yaml
steps:
- task: Docker@2
  displayName: 'buildAndPush - Windows'
  inputs:
    containerRegistry: cpacr
    repository: cpweb
    Dockerfile: '$(System.DefaultWorkingDirectory)/_source/Dockerfile'
    tags: '$(Release.DeploymentID)-windows'
```

### Build Docker image with ACR Tasks (option)

You can run docker build remotely with [Azure Container Registry Tasks](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview)

Run this script to build docker image for Windows and Linux

```powershell
az acr build -r cpacr https://github.com/tkubica12/dotnetcore-sqldb-tutorial.git -f Dockerfile --platform Windows -t cpweb:0-windows --no-wait
az acr build -r cpacr https://github.com/tkubica12/dotnetcore-sqldb-tutorial.git -f Dockerfile --platform Linux -t cpweb:0-linux --no-wait
```

## Using containers with Azure Container Instances (optional)

You can run simply docker image in [Azure Container Instances](https://docs.microsoft.com/en-us/azure/container-instances/).

```powershell
az acr credential show --name cpacr
$acrPassword="YOUR_ACR_PASSWORD"
az group create -n cp-aci -l westeurope
az container create -g cp-aci -n cpwebw --image cpacr.azurecr.io/cpweb:0-windows --cpu 1 --memory 4 --registry-login-server cpacr.azurecr.io --registry-username cpacr --registry-password $acrPassword --ports 80 --os-type Windows --ip-address Public
az container create -g cp-aci -n cpwebl --image cpacr.azurecr.io/cpweb:0-linux --cpu 1 --memory 2 --registry-login-server cpacr.azurecr.io --registry-username cpacr --registry-password $acrPassword --ports 80 --os-type Linux --ip-address Public
```

*Note: ACI with Windows containers supports 10.0.14393 (Windows 2016) and 10.0.17763 (Windows 2019) versions only.*

*Note: In Preview program you can run ACI in existing Azure Virtual Network and get private IP address, check [link](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-vnet)*

## Using containers with Azure Web Apps for Containers

### Windows docker images running on WebApps with DevOps

We will modify existing pipeline CPWEBWINDOWS-CD to deploy Windows container to Azure App Service for Containers

1. Add new task *Azure Web App for Containers* to Windows part
    - select Subscription and App name
    - image name: cpacr.azurecr.io/cpweb:$(Release.DeploymentID)-windows
    - Configuration settings:  TODO: SQL connection string

Repeat same steps for Linux container deployment to Azure App Service for Containers.

### Test application changes and automated deployment

TODO: DevOps 2 stage - stage DEV, PROD na WebApp sloty
TODO: zmena kodu a novy deployment - schvaleni prehozeni stage (zmena image - upravej soubor VersionController.cs na nove cislo)

## Creating and connecting Azure Kubernetes Service

Before we jump into Kubernetes discussion let's create Azure Kubernetes Service cluster. We will use simple solution (no AAD login integration, no custom networking etc.) to start with.

Create resource group

```powershell
az group create -n cp-aks -l westeurope
```

Create AKS. Note for this to work you need to be Contributor on your subscription and your AAD account needs to have permissions to create service principal (if not, read next). Note that CLI creates service principal account and it might take some to replicate - if command fails initialy, try again.

```powershell
az aks create -g cp-aks -n yourname-aks -c 2 --enable-addons monitoring --generate-ssh-keys
```

If you do not have permissions to do so, try to:
- You may use service principal account you have prepared beforehand. Make sure service principal is Contributor in your subscription. Use --client-secret and --service-principal
- AKS needs to create additional resource group with raw resources. Default name is MC_myResourceGroup_myAKSCluster_myregion. You can configure different name, but as of this writing you cannot point AKS to group that already exists. For now your service principal need to have rights on subscription level.

Next we need to download kubectl and we can use Azure CLI to do so. 

```powershell
az aks install-cli
```

Or you can download it yourself - it is single binary for Windows or Linux found [here for Windows](https://storage.googleapis.com/kubernetes-release/release/v1.16.0/bin/windows/amd64/kubectl.exe) and [here for Linux](https://storage.googleapis.com/kubernetes-release/release/v1.16.0/bin/linux/amd64/kubectl).

Wait for deployment to finish and then download connection information:

```powershell
az aks get-credentials -n yourname-aks -g cp-aks --admin
```

## Kubernetes: basic theory

## Kubernetes: Pods and Deployments

## Kubernetes: Monitoring and logging

## Kubernetes: exposing apps with Services

## Kubernetes: using ConfigMaps and Secrets

## Kubernetes: packaging deployments with Helm

## Kubernetes: exposing apps with Ingress

## Advanced Kubernetes (optional section for attendies using Kubernetes already)

### Network Policy

### AAD Pod identity and Key Vault

### Azure Application Gateway as Ingress

### Advanced monitoring and alerting

### Service mesh

## Contacts

### Tomas Kubica - Cloud Solutions Architect
- https://www.linkedin.com/in/tkubica
- https://github.com/tkubica12
- https://twitter.com/tkubica

### Jaroslav Jindrich - Cloud Solutions Architect
- https://www.linkedin.com/in/jjindrich
- https://github.com/jjindrich
- https://twitter.com/jjindrich_cz
