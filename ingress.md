# Azure Application Gateway as Ingress
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