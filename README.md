# Azure Container Apps on Azure Arc

This step-by-step tutorial will teach you one of the many available options for creating Azure Container Apps on-prem.

What are required for this?  
> - Microsoft Azure Subscription, [you can create a free account](https://azure.microsoft.com/en-us/free/) if you don't have any.
> - [Windows machine with virtualization technology (AMD-V / Intel VT-x)](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization)
>   - Windows Server 2016/Windows 10 or greater for Intel processor with VT-x
>   - Windows Server 2022/Windows 11 or greater AMD EPYC/Ryzen processor
  
> *Please note that almost all modern x86-64 CPUs support virtualization technology.*  
### High-level roadmap for this tutorial:
  
#### [1. Create a Kubernetes cluster with WSL 2 on Windows](https://learn.microsoft.com/en-us/windows/wsl/install)
#### [2. Connect the Kubernetes cluster to Azure Arc](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster)
#### [3. Create an Azure Container App on Azure Arc-enabled Kubernetes](https://learn.microsoft.com/en-us/azure/container-apps/azure-arc-create-container-app)


### 1. Create a Kubernetes cluster with WSL 2 on Windows

Create guest Windows VM on your Host Windows PC. Enables nested virtualization for the virtual machine.

```
Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true
```
Install [WSL 2](https://learn.microsoft.com/en-us/windows/wsl/install) on guest VM.
```
wsl --install
```
Download and install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) with WSL 2 option enabled. [Enable Kubernetes](https://docs.docker.com/desktop/kubernetes/) in Docker Desktop. Next [install](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows) Azure CLI.

### 2. Connect the Kubernetes cluster to Azure Arc

For additional information [please see](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster) the documentation page.

Create identity (user or service principal) which can be used to log in to Azure CLI and connect your cluster to Azure Arc. For example create user:
```
azurearc@[your primary domain part].onmicrosoft.com
```
then assign role
```
Kubernetes Cluster - Azure Arc Onboarding
```
to the newly created account. Next open PowerShell terminal on the guest VM and login with Azure CLI.
```
az login
```
Keep the latest terminal session open. Install the following Azure CLI extensions.
```
az extension add --name connectedk8s  --upgrade --yes
az extension add --name k8s-extension --upgrade --yes
az extension add --name customlocation --upgrade --yes
az extension remove --name containerapp
az extension add --source https://download.microsoft.com/download/5/c/2/5c2ec3fc-bd2a-4615-a574-a1b7c8e22f40/containerapp-0.0.1-py2.py3-none-any.whl --yes
```
Register the required namespaces with an account that have enough permissions.
```
az provider register --namespace Microsoft.ExtendedLocation --wait
az provider register --namespace Microsoft.KubernetesConfiguration --wait
az provider register --namespace Microsoft.App --wait
az provider register --namespace Microsoft.OperationalInsights --wait
az provider register --namespace Microsoft.Kubernetes --wait
```
Set environment variables based on your Kubernetes cluster deployment.
```
$LOCATION="eastus" 
$GROUP_NAME="rg-azure-arc-sample-1"
$CLUSTER_NAME="aks-azure-arc-sample-1" 
```
Create a resource group to contain your Azure Arc resources. Use an account that have enough permissions.
```
az group create --name $GROUP_NAME --location $LOCATION
```
Connect the cluster you created to Azure Arc.
```
az connectedk8s connect --resource-group $GROUP_NAME --name $CLUSTER_NAME
```
Validate the connection with the following command
```
az connectedk8s show --resource-group $GROUP_NAME --name $CLUSTER_NAME
```

### 3. Create an Azure Container App on Azure Arc-enabled Kubernetes

Set the following environment variables to the desired name of the Container Apps extension, the cluster namespace in which resources should be provisioned, and the name for the Azure Container Apps connected environment.
```
$EXTENSION_NAME="appenv-ext"
$NAMESPACE="appplat-ns" 
$CONNECTED_ENVIRONMENT_NAME="cae-azure-arc-sample-1"
```
Install the Container Apps extension to your Azure Arc-connected cluster.
```
az k8s-extension create `
    --resource-group $GROUP_NAME `
    --name $EXTENSION_NAME `
    --cluster-type connectedClusters `
    --cluster-name $CLUSTER_NAME `
    --extension-type 'Microsoft.App.Environment' `
    --release-train stable `
    --auto-upgrade-minor-version true `
    --scope cluster `
    --release-namespace $NAMESPACE `
    --configuration-settings "Microsoft.CustomLocation.ServiceAccount=default" `
    --configuration-settings "appsNamespace=${NAMESPACE}" `
    --configuration-settings "CLUSTER_NAME=${CONNECTED_ENVIRONMENT_NAME}" `
    --configuration-settings "envoy.annotations.service.beta.kubernetes.io/azure-load-balancer-resource-group=${AKS_CLUSTER_GROUP_NAME}"
```
Save the id property of the Container Apps extension.
```
$EXTENSION_ID=$(az k8s-extension show `
    --cluster-type connectedClusters `
    --cluster-name $CLUSTER_NAME `
    --resource-group $GROUP_NAME `
    --name $EXTENSION_NAME `
    --query id `
    --output tsv)
```
Set the following environment variables to the desired name of the custom location and for the ID of the Azure Arc-connected cluster.
```
$CUSTOM_LOCATION_NAME="personal-vm-1" # Name of the custom location
$CONNECTED_CLUSTER_ID=$(az connectedk8s show --resource-group $GROUP_NAME --name $CLUSTER_NAME --query id --output tsv)
```
Create the custom location
```
az customlocation create `
    --resource-group $GROUP_NAME `
    --name $CUSTOM_LOCATION_NAME `
    --host-resource-id $CONNECTED_CLUSTER_ID `
    --namespace $NAMESPACE `
    --cluster-extension-ids $EXTENSION_ID
```
and validate that the custom location is successfully created with the following command.
```
az customlocation show --resource-group $GROUP_NAME --name $CUSTOM_LOCATION_NAME
```
Read the custom location ID for the next step.
```
$CUSTOM_LOCATION_ID=$(az customlocation show `
    --resource-group $GROUP_NAME `
    --name $CUSTOM_LOCATION_NAME `
    --query id `
    --output tsv)
```
Create the Container Apps connected environment.
```
az containerapp connected-env create `
    --resource-group $GROUP_NAME `
    --name $CONNECTED_ENVIRONMENT_NAME `
    --custom-location $CUSTOM_LOCATION_ID --location=eastus
```
Validate that the Container Apps connected environment is successfully created with the following command. The output should show the provisioningState property as Succeeded. If not, run it again after a minute.
```
az containerapp connected-env show --resource-group $GROUP_NAME --name $CONNECTED_ENVIRONMENT_NAME
```
Retrieve connected environment ID.
```
$CONNECTED_ENVIRONMENT_ID = az containerapp connected-env list --custom-location $CUSTOM_LOCATION_ID -o tsv --query '[].id'
```
Create a sample .NET app.
```
az containerapp create `
    --resource-group $GROUP_NAME `
    --name ca-azure-arc-sample-2 `
    --environment $CONNECTED_ENVIRONMENT_ID `
    --environment-type connected `
    --image mcr.microsoft.com/dotnet/samples:aspnetapp `
    --target-port 80 `
    --ingress 'external'

az containerapp browse --resource-group $GROUP_NAME --name ca-azure-arc-sample-1
```