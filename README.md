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