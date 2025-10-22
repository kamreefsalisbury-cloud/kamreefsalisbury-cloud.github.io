---
title: "Azure DevOPS Terraform Infrastructure As Code"
date: 2025-10-20
published: true
categories:
  - blog
tags:
  - IaC
  - Azure DevOPS
---

### Automating Azure Dev Infrastructure with Terraform & Azure DevOps

Spin up a secure development foundation in Azure using Infrastructure as Code (IaC) and Azure Pipelines. This guide walks through the exact steps I used to stand up a dev resource group and virtual network with Terraform, backed by a remote state store and automated CI/CD hosted in Azure DevOPS.

---

## Prerequisites

- Azure subscription (with contributor rights): `<subscriptionId>`
- Azure CLI (`≥2.77.0`) and Terraform (`≥ 1.13.4`)
- Azure DevOps project with repo + service connection (`krss-dev-subscription`) using workload identity federation
- Remote state storage account: `tfstatekrss1` in RG `dev-tf-state`
- Local workstation (VS Code) or pipeline agent with network access to Azure services

---

## Step 1 – Bootstrap Remote State (One-Time Manual Build)

1. Create the Terraform State resource group and storage account dependecies:

   ```bash
   az group create -n dev-tf-state -l eastus2
   az storage account create \
     -g dev-tf-state \
     -n tfstatekrss1 \
     -l eastus2 \
     --sku Standard_GRS \
     --kind StorageV2 \
     --https-only true \
     --min-tls-version TLS1_2
   az storage container create --account-name tfstatekrss1 --name tfstate
   ```

2. Assign the Terraform service principal (from the Azure DevOPS pipeline connection) the Storage Blob Data Contributor role:
```bash
az role assignment create \
  --assignee-object-id <sp-object-id> \
  --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -g dev-tf-state -n tfstatekrss1 --query id -o tsv)
```

3. Security updates: enable Storage Account soft delete, Defender for Storage, and later restrict network access (service endpoint, private endpoint or conditional access trusted subnet). Public access is protected by RBAC and is adaquate for learning. Development and production environments at a minimum, must use a VNet located VM with Azure DevOPS agent and service endpoint.

## Step 2 – Initialize the Terraform Configuration

The repository root layout:
```
.
├── backend.tf
├── main.tf
├── providers.tf
├── variables.tf
├── versions.tf
└── .gitignore
```

### backend.tf
```yaml
terraform {
  backend "azurerm" {
    resource_group_name  = "dev-tf-state"
    storage_account_name = "tfstatekrss1"
    container_name       = "tfstate"
    key                  = "dev/dev-infrastructure.tfstate"
  }
}
```

### providers.tf
```yaml
provider "azurerm" {
  features {}
  subscription_id = "<subscripitionId>"
}

locals {
  default_tags = {
    environment = "development"
    costCenter  = "dev1"
  }
}
```

### variables.tf
```yaml
variable "location" {
  default = "eastus2"
}
variable "rg_name" {
  default = "dev-infrastructure"
}
variable "vnet_name" {
  default = "vnet-dev-core"
}
variable "vnet_address_space" {
  type    = list(string)
  default = ["10.20.0.0/16"]
}
variable "subnets" {
  type = map(object({
    address_prefix    = string
    service_endpoints = optional(list(string), [])
  }))
  default = {
    snet-dev-azdo-agent = {
      address_prefix    = "10.20.0.0/27"
      service_endpoints = ["Microsoft.Storage"]
    }
  }
}
```

### main.tf
```yaml
resource "azurerm_resource_group" "dev_infrastructure" {
  name     = var.rg_name
  location = var.location
  tags     = local.default_tags
}

resource "azurerm_virtual_network" "dev_vnet" {
  name                = var.vnet_name
  resource_group_name = azurerm_resource_group.dev_infrastructure.name
  location            = azurerm_resource_group.dev_infrastructure.location
  address_space       = var.vnet_address_space
  tags                = local.default_tags
}

resource "azurerm_subnet" "dev_subnets" {
  for_each             = var.subnets
  name                 = each.key
  resource_group_name  = azurerm_resource_group.dev_infrastructure.name
  virtual_network_name = azurerm_virtual_network.dev_vnet.name
  address_prefixes     = [each.value.address_prefix]
  service_endpoints    = lookup(each.value, "service_endpoints", [])
}
```

### versions.tf
```yaml
terraform {
  required_version = "~> 1.13.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.117"
    }
  }
}
```

## Step 3 – Validate Locally (Optional but Recommended)
```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```
This confirms state backend access and catch syntax errors before pushing.

## Step 4 – Build the Azure DevOps Pipeline
Add azure-pipelines.yml to repo root:
```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
- group: vg-terraform-dev   # stores ARM_SUBSCRIPTION_ID etc.
- name: TF_IN_AUTOMATION
  value: 'true'

stages:
- stage: Plan
  jobs:
  - job: plan
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - checkout: self
      clean: true
      fetchDepth: 0
      persistCredentials: true

    - task: TerraformInstaller@1
      inputs:
        terraformVersion: '1.13.4'

    - task: AzureCLI@2
      inputs:
        azureSubscription: 'krss-dev-subscription'
        scriptType: bash
        scriptLocation: inlineScript
        addSpnToEnvironment: true
        workingDirectory: '$(Build.SourcesDirectory)'
        inlineScript: |
          set -euo pipefail
          : "${ARM_SUBSCRIPTION_ID:?ARM_SUBSCRIPTION_ID is blank}"

          az account set --subscription "$ARM_SUBSCRIPTION_ID"
          az account show --query "{id:id, name:name}" -o json

          terraform init -input=false
          terraform fmt -check
          terraform validate
          terraform plan -input=false -out tfplan.bin
      env:
        ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)

    - publish: $(Build.SourcesDirectory)/tfplan.bin
      artifact: tfplan

- stage: Apply
  dependsOn: Plan
  condition: succeeded('Plan')
  jobs:
  - deployment: apply
    environment: dev   # configure approval gate in AzDO
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
            clean: true
            fetchDepth: 0
            persistCredentials: true

          - task: TerraformInstaller@1
            inputs:
              terraformVersion: '1.13.4'

          - download: current
            artifact: tfplan

          - task: AzureCLI@2
            inputs:
              azureSubscription: 'krss-dev-subscription'
              scriptType: bash
              scriptLocation: inlineScript
              addSpnToEnvironment: true
              workingDirectory: '$(Build.SourcesDirectory)'
              inlineScript: |
                set -euo pipefail
                : "${ARM_SUBSCRIPTION_ID:?ARM_SUBSCRIPTION_ID is blank}"

                az account set --subscription "$ARM_SUBSCRIPTION_ID"
                az account show --query "{id:id, name:name}" -o json

                cp "$(Pipeline.Workspace)/tfplan/tfplan.bin" ./tfplan.bin

                terraform init -input=false
                terraform apply -input=false tfplan.bin
            env:
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
```
### Highlights

- Workload identity (addSpnToEnvironment: true) means no secrets in code.
- Plan artifacts enforced; Apply stage reuses the reviewed plan.
- Manual approval via environment: dev.
- Formatting (terraform fmt -check) prevents drift (exit code 3 surfaces issues immediately).

## Step 5 – Commit + Push
```bash
git add .
git commit -m "Add dev infrastructure Terraform and pipeline"
git push origin main
```
Azure DevOps pipeline auto-runs:
1. Plan: If successful, publishes tfplan.bin.
2. Apply: After approving in the dev environment, deploys the RG + VNet.

## Step 6 – Verify Deployment
```bash
az group show -n dev-infrastructure -o table
az network vnet show -g dev-infrastructure -n vnet-dev-core -o jsonc
az network vnet subnet show -g dev-infrastructure -n snet-dev-azdo-agent --vnet-name vnet-dev-core -o jsonc
```
Ensure the subnet displays serviceEndpoints: Microsoft.Storage.

## Step 7 – Harden Storage Networking and Pipeline Agent Considerations (Post-Pipeline)

Once the VNet exists (especially before hosting a self-hosted agent):

1. Switch storage account networking back to Selected networks.
2. Add the subnet (snet-dev-azdo-agent) to the firewall rules.
3. Optionally add a Private Endpoint in the VNet.
4. Confirm Terraform still reaches the backend (if not, use a self-hosted agent inside the subnet).
5. Deploy Azure spot VM/VMSS in snet-dev-azdo-agent as the self-hosted AzDO agent.
6. Add NSG with locked-down inbound/outbound rules.
7. Enable VNet flow logs via Diagnostic Settings to Log Analytics.
8. Use Managed Identity for VM to avoid secrets.
9. Keep Terraform pipeline updated with new modules/resources.

## Governance & Security Checklist

- Tagging: Ensure default tags (environment, costCenter, owner) satisfy policy.
- RBAC: Terraform SP has minimum rights (Contributor on RG, Blob Data Contributor on state storage).
- Policy compliance: Validate lower-case naming, allowed locations, tags etc.
- Secret management: Use variable groups or Key Vault; never commit secrets in code.
- Monitoring: Set up pipeline failure alerts and resource diagnostics.

## Summary

By codifying Azure resources in Terraform, securing state in Azure Storage, and running CI/CD via Azure DevOps, these goals are achieved:

- Repeatable dev environment creation (dev-infrastructure RG & vnet-dev-core VNet)
- Policy-enforced tagging and security practices
- Automated plan/apply with manual approvals
- Foundation ready for advanced scenarios (self-hosted agents, private endpoints)
- This approach keeps infrastructure changes auditable, secure, and efficient—ideal for scaling platform work without sacrificing control.

Happy automating! 🚀