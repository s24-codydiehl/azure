# Simple Server Azure AKS Demonstration  <!-- omit in toc -->


# Table of Contents  <!-- omit in toc -->
- [Introduction](#introduction)
- [Azure Configurations for Terraform](#azure-configurations-for-terraform)
  - [Basic Azure Command Line Commands](#basic-azure-command-line-commands)
  - [Create the Azure Storage Account for Terraform Backend](#create-the-azure-storage-account-for-terraform-backend)
  - [Running Terraform with Your Own AZ Login vs Creating a Service Principal](#running-terraform-with-your-own-az-login-vs-creating-a-service-principal)
  - [Create an Azure Environmental Variables Export Bash Script](#create-an-azure-environmental-variables-export-bash-script)
- [Using Terraform to Create the Azure AKS Infrastructure](#using-terraform-to-create-the-azure-aks-infrastructure)
- [Azure Terraform Configuration](#azure-terraform-configuration)
  - [AKS - Azure Kubernetes Service](#aks---azure-kubernetes-service)
  - [ACR - Azure Container Registry](#acr---azure-container-registry)
    - [Pushing the Docker Images to ACR](#pushing-the-docker-images-to-acr)
  - [Public IPs](#public-ips)
  - [Role Assignment](#role-assignment)
- [Some Azure Terraform Observations](#some-azure-terraform-observations)
  - [Service Principal Hassle](#service-principal-hassle)
    - [NOT USED - Create Service Principal for Use with Terraform](#not-used---create-service-principal-for-use-with-terraform)
    - [NOT USED - Create Service Principal for Use with Terraform](#not-used---create-service-principal-for-use-with-terraform-1)
    - [NOT USED - Create an Azure Environmental Variables Export Bash Script](#not-used---create-an-azure-environmental-variables-export-bash-script)
  - [The ACR Image Pull from AKS Hassle](#the-acr-image-pull-from-aks-hassle)


# Introduction

There is a blog post regarding this project: [Creating Azure Kubernetes Service (AKS) the Right Way](https://medium.com/@kari.marttila/creating-azure-kubernetes-service-aks-the-right-way-9b18c665a6fa).

This Simple Server Azure AKS project relates to my previous project [Azure Table Storage with Clojure](https://medium.com/@kari.marttila/azure-table-storage-with-clojure-12055e02985c) in which I implemented the Simple Server to use Azure Table Storage as the application database. The Simple Server Azure version is in my Github account: [Clojure Simple Server](https://github.com/karimarttila/clojure/tree/master/clj-ring-cljs-reagent-demo/simple-server).

In this new Simple Server Azure AKS project I created a Terraform deployment configuration for the Simple Server Azure version. There is also a Kubernetes deployment configuration in which you can test the Simple Server Docker container both in Minikube and in Azure AKS service.
 
So, the rationale of this project was mainly to learn how to use Terraform with Azure, how to create a Kubernetes deployment and use the Kubernetes deployment configuration with Minikube and Azure AKS. In the next project I create the deployment in the AWS side for EKS and Fargate.

 
# Azure Configurations for Terraform

Read the following documents before starting to use Terraform with Azure:

- [Install and configure Terraform to provision VMs and other infrastructure into Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/terraform-install-configure)
 - [Store Terraform state in Azure Storage](https://docs.microsoft.com/en-us/azure/terraform/terraform-backend)
 - [Terraform Azure Provider](https://www.terraform.io/docs/providers/azurerm/)

You need to download and install Azure and Terraform command line tools, of course.


## Basic Azure Command Line Commands

```bash
az login                                       # Login to Azure.
az account list --output table                 # List your Azure accounts.
az account set -s "<choose-the-account>"       # Set the Azure account you want to use.
```
 

## Create the Azure Storage Account for Terraform Backend

Use script [create-azure-storage-account.sh](https://github.com/karimarttila/azure/blob/master/simple-server-aks/scripts/create-azure-storage-account.sh) to create an Azure Storage Account to be used for Terraform Backend. You need to supply for arguments for the script:

- The Azure location to be used.
- The resource group name for this exercise.
- The Storage account name for this exercise.
- The container name for the Terraform backend state (stored in the Storage account the script creates).

NOTE: You might want to use prefix "dev-" with your container name if you are going to store several Terraform environment backends in the same Azure Storage account.


## Running Terraform with Your Own AZ Login vs Creating a Service Principal

I spent quite a lot of time with this issue. You can read the whole story in the end of this README file in chapter "Service Principal Hassle". To make a long story short: AKS needs a Service principal and you have two possibilities to provide it:

1. You can create an external Service principal and populate its id and secret via TF_ENV_VARIABLENAME to terraform. This solution works but is a bit ugly since the cloud infra best practice is to create all infra using the same configuration as code.
2. You can use your own az login which has the right to create the necessary AD app and Service principal. 

I'm using the solution #2 here so that I'm able to create all resources in the terraform configuration and not create the AKS Service principal outside terraform.


## Create an Azure Environmental Variables Export Bash Script

Create a bash script in which you export the environmental variables you need to work with this project. Store the file e.g. in ~/.azure (i.e. **DO NOT STORE THE FILE IN GIT** since you don't want these secrets to be in plain text in your Git repository!). Example:

```bash
#!/bin/bash
echo "Setting environment variables for Simple Server Azure AKS Terraform project."
export AZURE_STORAGE_ACCOUNT=your-storage-account-name from storage account command result
export AZURE_STORAGE_KEY=your-storage-account-key from storage account command result
export ARM_ACCESS_KEY=your-storage-account-name from storage account command result
```

NOTE: Terraform requires the account key in environmental variable "ARM_ACCESS_KEY", I have used AZURE_STORAGE_KEY in some other scripts that's why I have the value twice.

You can then source the file using bash command:

```bash
source ~/.azure/your-bash-file.sh
```

Just to remind myself that I created file with name "kari-aks-demo.sh" for this project. So, I source it like:

```bash
source ~/.azure/kari-aks-demo.sh
```

# Using Terraform to Create the Azure AKS Infrastructure

Go to terraform directory. Give commands:

```bash
terraform init    # => Initializes Terraform, gets modules...
terraform plan    # => Shows the plan (what is going to be created...)
terraform apply   # => Apply changes
```

**NOTE**: Terraform apply fails the first time with some AD issue. Most probably this is caused because some AD app or Service principal resource is not ready for AKS. Wait a couple of minutes and run terraform plan/apply again - the second time terraform should be able to create all of the rest AKS related resources succesfully. 

# Azure Terraform Configuration

## AKS - Azure Kubernetes Service

I followed these three documentation:

- [Create a Kubernetes cluster with Azure Kubernetes Service and Terraform](https://docs.microsoft.com/en-us/azure/terraform/terraform-create-k8s-cluster-with-tf-and-aks)
- [Terraform azurerm_kubernetes_cluster resource](https://www.terraform.io/docs/providers/azurerm/r/kubernetes_cluster.html)
- [Creating a Kubernetes Cluster with AKS and Terraform](https://www.hashicorp.com/blog/kubernetes-cluster-with-aks-and-terraform)

The Azure AKS Terraform configuration is pretty straightforward. There was quite a lot of hassle with the Service principal but finally I managed to create a Service principal in the Terraform code so that AKS uses this Service principal to create the nodes for the Kubernetes cluster and also the Terraform code injects this Service principal id to the Role assignment in the ACR side for giving rights for AKS to pull images from ACR.


## ACR - Azure Container Registry

The terraform configuration also comprises the ACR - Azure Container Registry which is used for Docker images that the Kubernetes deployment running in AKS uses. After the original blog post I tried to deploy the Simple Server single-node version Kubernetes deployment to the first version of this AKS and there was an authentication error in the pull image phase. I needed to fix this by creating a role assignment for acr scope for AKS Service principal.


### Pushing the Docker Images to ACR

The Terraform configuration only creates the Cloud infra. We could automate pushing the Docker images to ACR but since the Docker images are not part of the cloud infra but part of the application layer I have documented this task in the Kubernetes repo: [Simple Server Kubernetes](https://github.com/karimarttila/kubernetes/tree/master/simple-server).

## Public IPs

The Azure AKS Simple Server deployment needs one public ip per configuration (one for single-node configuration and one for azure table storage configuration). The public ip is needed for the Kubernetes Load balancer resource so that we can test the Kubernetes deployment and access Simple Server via the Kubernetes Load balancer over internet. The terraform configuration creates the two public ips as part of the whole cloud infra. You can query those ips using terraform, e.g.:

```bash
terraform output -module=env-def.single-node-pip  # => public_ip_address = PUBLIC-IP
```

... and populate those ips to Kubernetes deployment scripts.

**NOTE**: The public IPs must be put in to the resource group that AKS creates - not in the main resource group of the project. See the details in the public-ip terraform module.

## Role Assignment

This is part of the configuration that I realized I had forgotten from the original setup. We need to give AKS's Service principal "Contributor" role for accessing ACR or Kubernetes deployment fails. I later on refactored this as part of the ACR module where it logically belongs (ACR gives permission for AKS to pull images). 

## Storage Account for Application Tables

I created a Storage account for hosting the application tables. Creating the actual tables is not part of the terraform infra but belongs to the application layer.


# Some Azure Terraform Observations

## Service Principal Hassle

I first thought that it would be nice to create the Service Principal that AKS uses in Terraform as well. I created a Terraform module for it but when running ```terraform apply``` I got error ```azurerm_azuread_application.application: graphrbac.ApplicationsClient#Create: ...Details=[{"odata.error":{"code":"Authorization_RequestDenied","message":{"lang":"en","value":"Insufficient privileges to complete the operation."}}}]``` I pretty soon realized that Terraform is running under the service principal I created earlier (see chapter "Create Service Principal for Use with Terraform") and there I assigned role "Contributor" for this principal. So, a contributor role can create Azure resources but it cannot create other roles. Therefore there are three solutions: 1. Create the service principal with role "owner" -> owner can create other roles. 2. Add some right to the service principal to create other roles. 3. Remove the service principal creation in Terraform code and use some existing service principal that has been created outside Terraform. Option 1. is a bit overkill. Option 3 is not elegant since you should be able to create all cloud resources in your cloud infra configuration and not have scripts here and there. Option 2 would be the optimal solution but it takes a bit time. 

I tried option 1 and created a service principal with owner role. Terraform apply command gave the same error when trying to create the service principal for AKS, damn.

Then some googling. I found a way to provide option 2, i.e. a custom service principal definition: [Terraform and Multi Tenanted Environments](https://azurecitadel.com/automation/terraform/lab5/#advanced-service-principal-configuration). I tried these instructions but I finally noticed that I should have AD admin rights in our corporation AD that is linked to the Azure subscription that I'm using - didn't work. So, I had to fall back to option 3 and create the service principal for AKS outside terraform code and inject the service principal using environmental variables. Not cloud infra best practice but after this hassle I thought that I just need to move on.

I managed to create the AKS finally so that I just populated the same Service principal I used to run the terraform to the AKS. The commit is "Creating service principal outside terraform and injecting it in environmental variable - now terraform init/plan/apply work - created AKS cluster ok" if you want to try this version. I have saved the Service principal chapters for historical reasons below as subchapters of this chapter.

Then I had a conversation with my colleague Julius Eerola regarding this issue and he said that he is running terraform with his own Azure user (see [Azure Provider: Authenticating using the Azure CLI](https://www.terraform.io/docs/providers/azurerm/auth/azure_cli.html)). I remembered earlier that I had also tried this but terraform apply had failed to some AD issue. Then I remembered that in some cases also in the AWS side terraform apply had failed the first time since the creation of resources were not right or the resource didn't have time to initialize before terraform tried to create the next resource. I removed the external Service Principal dependency and configured terraform to create the Service principal itself and AKS to use this Service principal. I destroyed previous AKS infra, removed the sourced bash export Service principal credentials and tried to run terraform init/plan/apply again. Once again there was some AD error: ```azurerm_kubernetes_cluster.aks: Error creating/updating Managed Kubernetes Cluster ... containerservice.ManagedClustersClient#CreateOrUpdate: Failure sending request: StatusCode=0 -- Original Error: Code="ServicePrincipalNotFound" Message="Service principal clientID: 0000000000000000000000 not found in Active Directory tenant 000000000000000000000000 , Please see https://aka.ms/acs-sp-help for more details."```. I waited a couple of minutes and tried to run terraform apply again - this time terraform created AKS and the creation was successful. So, the trick not to use external Service principal with Terraform is to run terraform apply twice (and wait a bit between commmands) :-)  . 


### NOT USED - Create Service Principal for Use with Terraform 

NOTE: Use this service principal if you want to inject the same service principal you are using with Terraform for AKS as well (as environmental variables). If you want more elegant solution, see next chapter.

Create a service principal for Terraform:

```bash
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/SUBSCRIPTION_ID"
```

Terraform uses this service principal when it creates resources in Azure.


### NOT USED - Create Service Principal for Use with Terraform

I keep this chapter for historical reasons. I tried to create a custom Terraform Service Principal to be used with Terraform so that in terraform scripts I could be able to create other service principals (e.g. AKS needs one). Seemed to be pretty hard and I finally couldn't do this since I lack my corporation AD admin rights. 

Most Azure Terraform examples that need to use service principal just create the service principal beforehand using azure cli and then use this service principal id in terraform code. This solution works in examples but is not that elegant since the cloud infra best practice is to create all infra using the configuration only (if possible). Azure AKS needs a service principal to create AKS resources. The default contributor role service principal cannot create apps and other service principals and therefore we need to create a custom service principal for Terraform.

Create file ~/.azure/<your-custom-terraform-role>.json:

```json
{
    "Name":  "Terraform",
    "IsCustom":  true,
    "Description":  "Custom Contributor to be able to create apps and roles in Terraform.",
    "Actions":  [
        "*"
        ],
    "NotActions":  [
        "Microsoft.Authorization/classicAdministrators/write",
        "Microsoft.Authorization/classicAdministrators/delete",
        "Microsoft.Authorization/denyAssignments/write",
        "Microsoft.Authorization/denyAssignments/delete",
        "Microsoft.Authorization/locks/write",
        "Microsoft.Authorization/locks/delete",
        "Microsoft.Authorization/policyAssignments/write",
        "Microsoft.Authorization/policyAssignments/delete",
        "Microsoft.Authorization/policyDefinitions/write",
        "Microsoft.Authorization/policyDefinitions/delete",
        "Microsoft.Authorization/policySetDefinitions/write",
        "Microsoft.Authorization/policySetDefinitions/delete",
        "Microsoft.Authorization/elevateAccess/Action",
        "Microsoft.Blueprint/*/write",
        "Microsoft.Blueprint/*/delete"
        ],
    "DataActions": [],
    "NotDataActions": [],
    "AssignableScopes":  [
        "/subscriptions/00000000000000000000000000000000"
        ]
}
``` 

Change your Azure subscription in that file. NOTE: Do not add this file to your Git project - keep it in your ~/.azure directory!

Create role and role assignment:

```bash
az role definition create --role-definition ~/.azure/kari-aks-demo-terraform.json
az role definition list -o table | grep -i terraform   # => List that you see it was created. 
az ad sp list --all | jq '.[] | select(.displayName == "Terraform") | { id: .objectId, name: .displayName }'  # => Lists the role definition name and id.
az role assignment create --role "Terraform" --assignee "ID-YOU-GOT-FROM-PREVIOUS-COMMAND"
az role assignment list -o table # => You should see the new role assignment Terraform.
az role definition list --name Terraform  # => List the role definition.
az ad sp create-for-rbac --name "http://0000000000000000000000/TerraformSP" --role "Terraform" --scopes="/subscriptions/0000000000000000000000" # => Change the 0-string with your subscription id. You get the appId, password and tenant as result, save them, you need them in the next chapter.
```

After this I updated the Environmental variables as explained in the next chapter and sourced the new variables, and tried terraform init and apply - again same error: "Insufficient privileges to complete the operation". I tried to follow the document [Using Terraform to extend beyond ARM](https://azurecitadel.com/automation/terraform/lab8/) :

- Navigate to Azure Active Directory (AAD)
- Under the Manage list, select App registrations (Preview)
- Ensure the All Applications tab is selected
- Search for, and select the Terraform Service Principal application
- Select API Permissions
- Add Permissions
- In "Azure Active Directory Graph" choose:
  - Application.ReadWrite.All
  - User.Read

This does not work since Application.ReadWrite.All requires admin rights for my corporation AD in which the subscription is linked. 

I left the Terraform SP and the right grant request - let's see if my corporation AD admin grants the right. If yes, I'll try to use this service principal with terraform. I saved this service principal in file "ss-aks-profile-custom-terraform.sh" (for my information if I need it later).

 
### NOT USED - Create an Azure Environmental Variables Export Bash Script

Create a bash script in which you export the environmental variables you need to work with this project. Store the file e.g. in ~/.azure (i.e. **DO NOT STORE THE FILE IN GIT** since you don't want these secrets to be in plain text in your Git repository!). Example:

```bash
#!/bin/bash
echo "Setting environment variables for Simple Server Azure AKS Terraform project."
export AZURE_STORAGE_ACCOUNT=your-storage-account-name from storage account command result
export AZURE_STORAGE_KEY=your-storage-account-key from storage account command result
export ARM_ACCESS_KEY=your-storage-account-name from storage account command result
export ARM_SUBSCRIPTION_ID=your-subscription-id
export ARM_CLIENT_ID=app-id from service principal command result
export ARM_CLIENT_SECRET=password from Service Principal command result
export ARM_TENANT_ID=tenant id from Service Principal command result
# Since there was some hassle to create the service principal in
# Terraform code, let's just use the service principal for AKS we
# created using azure cli.
export TF_VAR_aks_client_id=${ARM_CLIENT_ID}
export TF_VAR_aks_client_secret=${ARM_CLIENT_SECRET}
```

(NOTE: Terraform requires the account key in environmental variable "ARM_ACCESS_KEY", I have used AZURE_STORAGE_KEY in some other scripts that's why I have the value twice).

You can then source the file using bash command:

```bash
source ~/.azure/your-bash-file.sh
```

Just to remind myself that I created file with name "kari-aks-demo.sh" for this project. So, I source it like:

```bash
source ~/.azure/kari-aks-demo.sh
```

## The ACR Image Pull from AKS Hassle

I must admit now that I created the original version of the blog post of this project a bit too early :-) . After the original version of the blog post I tried to use the terraform version of that time to deploy the Simple Server single-node Kubernetes deployment to that AKS. Didn’t work. AKS didn’t have authorization to pull images from ACR. When examining the failed pod (using kubectl describe command…) I saw that the image pull was failed: “Failed to pull image … unauthorized: authentication required…”. It took me quite some time to figure out how to do this. In this kind of situation it is a good idea to create a working reference infra e.g. using Portal or command line tool. So, I created everything (AKS, ACR, Public IPs, Service principal, Role assignment etc.) manually step by step using az cli and deployed the Simple Server single-node version there and tested the deployment by curling the Simple Server API — the reference infra worked ok. Now I had a working reference infra for examining what was wrong with my Terraform configuration. I examined the created azure resources side by side (reference infra created by az cli and my terraform code). Finally I figured out the issues and was able to fix them. While spending some 8h with the terraform code I also quite extensively refactored it (e.g. put Service principal to aks module where it belongs, put role assignment to acr module where it belongs, introduced locals etc.).

The lesson of the story: If terraform apply goes smoothly it doesn’t yet mean that the cloud infra is working properly — deploy what ever you are doing (Kubernetes, Docker, apps to VMs…) to the infra — if this part also works ok then you have a working cloud infra.