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

# Automating Azure Dev Infrastructure with Terraform & Azure DevOps

Spin up a secure development foundation in Azure using Infrastructure as Code (IaC) and Azure Pipelines. This guide walks through the exact steps we used to stand up a dev resource group and virtual network with Terraform, backed by a remote state store and automated CI/CD.

---

## Prerequisites

- Azure subscription (with contributor rights): `89382f9a-deac-49be-a1af-9a3a8d7ceed3`
- Azure CLI (`az`) and Terraform (`≥ 1.13.4`)
- Azure DevOps project with repo + service connection (`krss-dev-subscription`) using workload identity federation
- Remote state storage account: `tfstatekrss1` in RG `dev-tf-state`
- Local workstation (VS Code) or pipeline agent with network access to Azure services

---

## Step 1 – Bootstrap Remote State (One-Time)

1. Create the resource group and storage account (if not already):

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

   