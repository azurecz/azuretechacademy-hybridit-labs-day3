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
We will deploy sample Todo web application from https://github.com/tkubica12/dotnetcore-sqldb-tutorial

### Deploy Azure services manually
TODO: arm script & run

### Deploy Azure services with Azure DevOps
TODO: 
* Druhe repo s arm na ACR a SQL, Web App
* Import repository to Azure DevOps a ARM
* Zakladni pipeline na deploy ARM

## Building and storing container images with Azure Container Registry

### Build with Azure Build Task (option)
TODO: Windows, Linux

### Build Windows docker image with ACR with Azure DevOps
TODO:
* build v DevOps
* zmena image - upravej soubor VersionController.cs na nove cislo
* build v DevOps

### Build Linux docker image with ACR with Azure DevOps
TODO: rozsirit pipeline

## Using containers with WebApps

### Windows docker images running on WebApps with DevOps
TODO: do pipeline pridat deploy do webapp

### Use WebApp slots for deployment
TODO: DevOps 2 stage - stage DEV, PROD na WebApp sloty

## Creating and connecting Azure Kubernetes Service

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
