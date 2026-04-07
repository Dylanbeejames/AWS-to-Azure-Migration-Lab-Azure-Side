# AWS-to-Azure Migration Lab — Azure Side

**Author:** Dylan  
**Video Walkthrough:** [▶ Watch on Loom](https://www.loom.com/share/a8790388210245a198ff64b1d3d2154d)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Resources Deployed](#resources-deployed)
- [Step 1 — Set Up Working Folders](#step-1--set-up-working-folders)
- [Step 2 — Generate SSH Key](#step-2--generate-ssh-key)
- [Step 3 — Create Terraform Configuration Files](#step-3--create-terraform-configuration-files)
- [Step 4 — Run Terraform Commands](#step-4--run-terraform-commands)
- [Step 5 — Connect to the Linux VM](#step-5--connect-to-the-linux-vm)
- [Step 6 — Cleanup](#step-6--cleanup)
- [Summary](#summary)
- [Next Steps](#next-steps)

---

## Overview

This lab provisions the **Azure-side infrastructure** for an AWS-to-Azure migration scenario using **Terraform** and the **Azure CLI**. The goal is to deploy a fully functional Azure environment — including networking, a public IP, and a Linux VM — and verify SSH connectivity, establishing the foundation for a cross-cloud migration workflow.

All resources are defined as Infrastructure as Code, making the environment fully repeatable and version-controlled.

---

## Architecture

```
Public Internet
      │
      ▼ SSH (port 22)
Public IP — myPublicIP (13.68.145.247)
      │
      ▼
Network Interface — myNic
      │
      ▼
Subnet — mySubnet (10.0.1.0/24)
      │
      ▼
Virtual Network — myVnet (10.0.0.0/16)
      │
      ▼
Linux VM — myVM (Ubuntu 22.04 LTS)
```

---

## Technologies Used

| Technology | Purpose |
|------------|---------|
| Terraform | Infrastructure as Code provisioning |
| Azure CLI | Authentication and subscription context |
| Azure Virtual Network | Network isolation for the VM |
| Azure Public IP (Standard SKU) | External SSH access endpoint |
| Azure Linux VM (Ubuntu 22.04 LTS) | Target migration workload |
| SSH Key Authentication | Secure, passwordless VM access |
| PowerShell / Command Prompt | Local CLI environment |

---

## Resources Deployed

| Resource | Name | Resource Group |
|----------|------|----------------|
| Resource Group | `my-azure-lab-rg` | — |
| Virtual Network | `myVnet` | `my-azure-lab-rg` |
| Subnet | `mySubnet` | `my-azure-lab-rg` |
| Network Interface | `myNic` | `my-azure-lab-rg` |
| Public IP | `myPublicIP` | `my-azure-lab-rg` |
| Linux VM | `myVM` | `my-azure-lab-rg` |

---

## Step 1 — Set Up Working Folders

Open PowerShell or Command Prompt and create the project directory structure:

```powershell
cd %USERPROFILE%
mkdir aws-to-azure-migrate
cd aws-to-azure-migrate
mkdir azure-side aws-side
cd azure-side
```

> This structure separates the Azure-side and AWS-side configurations, keeping the migration project organized as both environments are built out.

---

## Step 2 — Generate SSH Key

Generate an RSA key pair for authenticating to the VM:

```powershell
ssh-keygen -t rsa -b 2048
```

When prompted, save to:

```
C:\Users\17576\.ssh\id_rsa
```

Leave the passphrase empty for lab purposes.

**Two files are created:**

| File | Purpose |
|------|---------|
| `id_rsa` | Private key — stays on your local machine |
| `id_rsa.pub` | Public key — injected into the VM by Terraform |

> **Security Note:** Never share or commit your private key (`id_rsa`). In production, use a secrets manager or Azure Key Vault to handle key material. Add `.ssh/` to your `.gitignore`.

---

## Step 3 — Create Terraform Configuration Files

### `variables.tf`

Declares the input variables used across the configuration:

```hcl
variable "resource_group_name" {}
variable "location" {}
```

---

### `terraform.tfvars`

Assigns values to the declared variables:

```hcl
resource_group_name = "my-azure-lab-rg"
location            = "eastus"
```

> This file can be duplicated and modified to redeploy the same environment in a different region or subscription — useful for testing or DR scenarios.

---

### `main.tf`

Defines all Azure resources:

```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_virtual_network" "vnet" {
  name                = "myVnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet" {
  name                 = "mySubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "nic" {
  name                = "myNic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_public_ip" "pip" {
  name                = "myPublicIP"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
  allocation_method   = "Static"
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                  = "myVM"
  resource_group_name   = azurerm_resource_group.rg.name
  location              = azurerm_resource_group.rg.location
  size                  = "Standard_B1s"
  network_interface_ids = [azurerm_network_interface.nic.id]
  admin_username        = "azureuser"

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("C:/Users/17576/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}
```

---

### `outputs.tf`

Exposes key values after deployment:

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.rg.name
}

output "resource_group_location" {
  value = azurerm_resource_group.rg.location
}

output "vm_name" {
  value = azurerm_linux_virtual_machine.vm.name
}

output "vnet_name" {
  value = azurerm_virtual_network.vnet.name
}

output "public_ip" {
  value = azurerm_public_ip.pip.ip_address
}
```

---

## Step 4 — Run Terraform Commands

**Authenticate to Azure:**

```bash
az login
```

**Initialize, plan, and apply:**

```powershell
terraform init
terraform plan
terraform apply
```

Type `yes` when prompted to confirm resource creation.

**Example output after a successful apply:**

```
resource_group_location = "eastus"
resource_group_name     = "my-azure-lab-rg"
vm_name                 = "myVM"
vnet_name               = "myVnet"
public_ip               = "13.68.145.247"
```

---

## Step 5 — Connect to the Linux VM

Use the public IP from the Terraform output to SSH into the VM:

```powershell
ssh -i C:\Users\17576\.ssh\id_rsa azureuser@13.68.145.247
```

Type `yes` to accept the host fingerprint on first connection.

> Replace `13.68.145.247` with the actual public IP returned by `terraform output public_ip`.

✅ A successful login confirms the VM is live and SSH authentication is working correctly.

---

## Step 6 — Cleanup

Destroy all provisioned resources to avoid ongoing charges:

```powershell
terraform destroy
```

Type `yes` to confirm. All resources in the configuration will be permanently removed.

> ⚠️ This removes the VM, networking resources, and public IP. Ensure you no longer need access to the environment before confirming.

---

## Summary

This lab establishes the Azure-side foundation for a cross-cloud migration:

- ✅ Structured a multi-environment Terraform project layout
- ✅ Generated and configured SSH key authentication
- ✅ Deployed full Azure networking stack and Linux VM via Terraform IaC
- ✅ Used Terraform outputs to surface key infrastructure values post-deploy
- ✅ Verified SSH connectivity to the provisioned VM

---

## Next Steps

| Enhancement | Description |
|-------------|-------------|
| **AWS Side Build-Out** | Deploy the corresponding AWS VPC, EC2 instance, and networking for the full migration scenario |
| **VNet Peering / VPN Gateway** | Establish private connectivity between Azure and AWS environments |
| **Azure Bastion** | Replace public SSH exposure with browser-based secure access |
| **NSG Hardening** | Restrict SSH (port 22) to specific source IPs instead of open access |
| **Terraform Remote State** | Store `terraform.tfstate` in Azure Blob Storage for team collaboration |

---

*© Dylan — Cloud Security Specialist*
