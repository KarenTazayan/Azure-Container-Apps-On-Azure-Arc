# Azure Container Apps on Azure Arc

In this tutorial you will learn one of the many available options of how to create Azure Container Apps on-prem.  

What are required for this?  
> - Microsoft Azure Subscription, [you can create a free account](https://azure.microsoft.com/en-us/free/) if you don't have any.
> - [Windows machine with virtualization technology (AMD-V / Intel VT-x)](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization)
>   - Windows Server 2016/Windows 10 or greater for Intel processor with VT-x
>   - Windows Server 2022/Windows 11 or greater AMD EPYC/Ryzen processor

  
> *Please note that almost all modern x86-64 CPUs support virtualization technology.*  
## High-level roadmap for this tutorial:
  
### [1. Create a Kubernetes cluster with WSL 2 on Windows](https://learn.microsoft.com/en-us/windows/wsl/install)
### [2. Connect the Kubernetes cluster to Azure Arc](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster)
### [3. Create an Azure Container App on Azure Arc-enabled Kubernetes](https://learn.microsoft.com/en-us/azure/container-apps/azure-arc-create-container-app)