
# AWS-to-Azure Migration Lab – Azure Side

## Objective
Deploy Azure infrastructure using Terraform and verify connectivity to a Linux VM via SSH.  
This lab includes creating a Resource Group, Virtual Network, Subnet, Network Interface, Public IP, and Linux VM.

**Platform:** Microsoft Azure  
**Tools:** Terraform, Azure CLI, PowerShell/Command Prompt, SSH  
**VM Username:** `azureuser`  
**VM OS:** Ubuntu 22.04 LTS

---

## Step 1: Set Up Working Folders

Open PowerShell or Command Prompt:

```powershell
cd %USERPROFILE%
mkdir aws-to-azure-migrate
cd aws-to-azure-migrate
mkdir azure-side aws-side
cd azure-side
```

---

## Step 2: Generate SSH Key

Generate an SSH key for connecting to the VM:

```powershell
ssh-keygen -t rsa -b 2048
```

- **Save to:** `C:\Users\17576\.ssh\id_rsa`  
- **Leave passphrase empty** for lab purposes.

**Result:** Two files are created:

- `C:\Users\17576\.ssh\id_rsa` (private key)  
- `C:\Users\17576\.ssh\id_rsa.pub` (public key)

---

## Step 3: Create Terraform Configuration Files

### `variables.tf`
```hcl
variable "resource_group_name" {}
variable "location" {}
```

### `terraform.tfvars`
```hcl
resource_group_name = "my-azure-lab-rg"
location            = "eastus"
```

### `main.tf`
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
  name                 = "myVM"
  resource_group_name  = azurerm_resource_group.rg.name
  location             = azurerm_resource_group.rg.location
  size                 = "Standard_B1s"
  network_interface_ids = [azurerm_network_interface.nic.id]
  admin_username       = "azureuser"

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

### `outputs.tf`
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

## Step 4: Run Terraform Commands

```powershell
terraform init
terraform plan
terraform apply
```

- Confirm resource creation by typing `yes`.

**Example Output:**
```text
resource_group_location = "eastus"
resource_group_name     = "my-azure-lab-rg"
vm_name                 = "myVM"
vnet_name               = "myVnet"
public_ip               = "13.68.145.247"
```

---

## Step 5: Connect to the Linux VM

```powershell
ssh -i C:\Users\17576\.ssh\id_rsa azureuser@13.68.145.247
```

- Accept the host fingerprint by typing `yes`.  
- You will be logged in using your SSH public key.

> **Note:** Replace `13.68.145.247` with the actual public IP from Terraform output.

---

## Step 6: Lab Completion

**Resources deployed:**

| Resource Type       | Name             | Resource Group       |
|--------------------|----------------|--------------------|
| Resource Group      | my-azure-lab-rg | my-azure-lab-rg    |
| Virtual Network     | myVnet          | my-azure-lab-rg    |
| Subnet              | mySubnet        | my-azure-lab-rg    |
| Network Interface   | myNic           | my-azure-lab-rg    |
| Public IP           | 13.68.145.247   | my-azure-lab-rg    |
| Linux VM            | myVM            | my-azure-lab-rg    |

✅ Verified SSH login to the VM  
✅ Lab environment fully operational

---

### Notes
- Paths must match where you generated your SSH keys.  
- Terraform tfvars file can be reused for additional environments.  
- Check outputs after `terraform apply` to see your VM public IP.
