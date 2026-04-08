# AWS EC2 to Azure Migration Lab — Azure Migrate (Lift & Shift)

**Author:** Dylan Bryson  
**Role:** Cloud Security Specialist  
**Video Walkthrough:** [▶ Watch on Loom](https://www.loom.com/share/a8790388210245a198ff64b1d3d2154d)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Cost Estimate](#cost-estimate)
- [Prerequisites](#prerequisites)
- [Folder Setup](#folder-setup)
- [Part 1 — Build the AWS Source Environment](#part-1--build-the-aws-source-environment)
- [Part 2 — Build the Azure Target Environment](#part-2--build-the-azure-target-environment)
- [Part 3 — Deploy and Configure the Azure Migrate Appliance](#part-3--deploy-and-configure-the-azure-migrate-appliance)
- [Part 4 — Assess the EC2 Instance](#part-4--assess-the-ec2-instance)
- [Part 5 — Set Up Replication](#part-5--set-up-the-replication-appliance-and-replicate)
- [Part 6 — Test Migration and Cutover](#part-6--test-migration-and-cutover)
- [Verification Checklist](#verification-checklist)
- [Troubleshooting](#troubleshooting)
- [Teardown](#teardown)
- [Summary](#summary)
- [Next Steps](#next-steps)

---

## Overview

This lab performs a complete **lift-and-shift migration** of a Windows Server running on AWS EC2 into Azure using **Azure Migrate**. It covers the full migration lifecycle — discovery, assessment, agentless replication, and cutover — using Terraform for infrastructure provisioning on both the AWS and Azure sides.

Cloud-to-cloud migrations are one of the most common and highest-value engagements in cloud engineering. This lab mirrors the real-world process used when companies consolidate workloads, move between cloud providers, or migrate acquired businesses onto a standardized platform.

> **Interview framing:** "I performed an end-to-end cloud migration from AWS to Azure using Azure Migrate — discovery, assessment, agentless replication, and cutover — and I can walk through every step of that process and explain the decisions made at each stage."

---

## Architecture

```
AWS Account                            Azure Subscription
─────────────────────────              ──────────────────────────────────────
VPC: 10.0.0.0/16                       rg-migrate-source-dylan
└── Subnet: 10.0.1.0/24               ├── vnet-migrate (10.1.0.0/16)
    └── EC2: Windows Server 2022       │   └── snet-migrate (10.1.1.0/24)
        (source machine)               ├── Azure Migrate Appliance VM
                                       ├── Replication Appliance VM
                                       └── Storage Account (replication cache)

                                       rg-migrate-target-dylan
                                       ├── Azure Migrate Project
                                       ├── Recovery Services Vault
                                       └── Target VM (post-cutover)
```

**Migration flow:**
1. Azure Migrate appliance is deployed in Azure and given AWS credentials
2. Appliance discovers the EC2 instance and reports it to the Migrate project
3. Assessment verifies readiness and recommends an Azure VM size
4. Replication appliance continuously syncs disk changes from AWS to Azure
5. Cutover finalizes replication and creates the target Azure VM

> **Important:** The appliance communicates with the EC2 instance over its **public IP**. There is no VPN between AWS and Azure in this lab — private IPs are unreachable from Azure.

---

## Technologies Used

| Technology | Purpose |
|------------|---------|
| Azure Migrate | Discovery, assessment, and migration orchestration |
| Azure Site Recovery | Underlying replication engine |
| Terraform (AWS + Azure) | Infrastructure as Code for both cloud environments |
| AWS EC2 (Windows Server 2022) | Source machine to be migrated |
| AWS VPC, Subnet, Security Group, IAM | Source networking and access control |
| Azure Virtual Network | Target network for the migrated VM |
| Azure Recovery Services Vault | Replication state and orchestration |
| Azure Storage Account | Replication disk cache |
| Azure Log Analytics Workspace | Discovery data and migration history |
| RDP | VM access for validation |

---

## Cost Estimate

> ⚠️ This lab uses paid resources on both AWS and Azure. Destroy everything immediately after completing to minimize charges.

| Resource | Estimated Cost |
|----------|---------------|
| EC2 t3.medium (Windows) | ~$0.08/hour |
| Azure Migrate appliance VM (Standard_A4_v2) | ~$0.40/hour |
| Azure Storage (replication cache, ~30GB) | ~$0.60/day |
| Azure target VM (Standard_B2s, post-cutover) | ~$0.05/hour |
| **Total for a full-day lab** | **~$5–8** |

---

## Prerequisites

**AWS Setup:**

1. Create an IAM user with programmatic access named `terraform-migrate-lab`
2. Attach `AmazonEC2FullAccess` and `AmazonVPCFullAccess` policies
3. Generate and save the Access Key ID and Secret Access Key

**Tools — Mac:**

```bash
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
brew install awscli
aws configure
brew install azure-cli
az login
az account set --subscription "Azure subscription 1"
```

**Tools — Windows (PowerShell):**

```powershell
# Download AWS CLI from https://aws.amazon.com/cli/
aws configure
# Download Azure CLI from https://aka.ms/installazurecliwindows
az login
az account set --subscription "Azure subscription 1"
```

**Verify both CLIs are working:**

```bash
aws sts get-caller-identity
az account show
```

---

## Folder Setup

This lab uses two separate Terraform roots — one for AWS, one for Azure. Keep them separate so each side can be destroyed independently.

**Mac:**

```bash
mkdir ~/aws-to-azure-migrate
cd ~/aws-to-azure-migrate
mkdir aws-side azure-side
touch aws-side/main.tf aws-side/variables.tf aws-side/outputs.tf aws-side/terraform.tfvars
touch azure-side/main.tf azure-side/variables.tf azure-side/outputs.tf azure-side/terraform.tfvars
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Path "$HOME\aws-to-azure-migrate"
cd "$HOME\aws-to-azure-migrate"
New-Item -ItemType Directory -Path aws-side, azure-side
New-Item -ItemType File aws-side\main.tf, aws-side\variables.tf, aws-side\outputs.tf, aws-side\terraform.tfvars
New-Item -ItemType File azure-side\main.tf, azure-side\variables.tf, azure-side\outputs.tf, azure-side\terraform.tfvars
```

---

## Part 1 — Build the AWS Source Environment

### `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region to deploy the source EC2 instance into."
  type        = string
  default     = "us-east-1"
}

variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to make resource names unique."
  type        = string
}

variable "windows_ami" {
  description = "Windows Server 2022 Base AMI ID for us-east-1."
  type        = string
  default     = "ami-0c2b0d3fb02824d92"
}

variable "instance_type" {
  type    = string
  default = "t3.medium"
}

variable "admin_password" {
  description = "Administrator password for the Windows Server instance."
  type        = string
  sensitive   = true
}
```

### `terraform.tfvars`

```hcl
aws_region     = "us-east-1"
yourname       = "dylan"
admin_password = "YourSecureP@ssw0rd123!"
```

> **Password requirements:** Minimum 12 characters including uppercase, lowercase, numbers, and symbols. AWS will reject weak passwords.

### `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "vpc-migrate-${var.yourname}", project = "azure-migrate-lab" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "igw-migrate-${var.yourname}" }
}

resource "aws_subnet" "main" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  tags                    = { Name = "snet-migrate-${var.yourname}" }
}

resource "aws_route_table" "main" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = { Name = "rt-migrate-${var.yourname}" }
}

resource "aws_route_table_association" "main" {
  subnet_id      = aws_subnet.main.id
  route_table_id = aws_route_table.main.id
}

resource "aws_security_group" "source_vm" {
  name        = "migrate-source-sg-${var.yourname}"
  description = "Allow HTTPS and RDP for Azure Migrate lab"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS for Azure Migrate appliance communication"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "RDP for admin access"
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "WinRM for Azure Migrate discovery"
    from_port   = 5985
    to_port     = 5985
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "migrate-source-sg-${var.yourname}" }
}

resource "aws_instance" "source_vm" {
  ami                    = var.windows_ami
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.main.id
  vpc_security_group_ids = [aws_security_group.source_vm.id]

  root_block_device {
    volume_type = "gp3"
    volume_size = 30
    encrypted   = false
  }

  user_data = <<-EOF
    <powershell>
    net user Administrator "${var.admin_password}"
    </powershell>
  EOF

  tags = { Name = "ec2-migrate-source-${var.yourname}", project = "azure-migrate-lab" }
}

resource "aws_iam_user" "migrate_user" {
  name = "svc-azure-migrate-${var.yourname}"
  tags = { project = "azure-migrate-lab" }
}

resource "aws_iam_access_key" "migrate_user_key" {
  user = aws_iam_user.migrate_user.name
}
```

### `outputs.tf`

```hcl
output "ec2_public_ip" {
  description = "Public IP of the source EC2 instance — use this for RDP and as the discovery source in Azure Migrate."
  value       = aws_instance.source_vm.public_ip
}

output "ec2_private_ip" {
  value = aws_instance.source_vm.private_ip
}

output "migrate_access_key_id" {
  value = aws_iam_access_key.migrate_user_key.id
}

output "migrate_secret_access_key" {
  value     = aws_iam_access_key.migrate_user_key.secret
  sensitive = true
}
```

### Deploy the AWS Side

**Mac:**

```bash
cd ~/aws-to-azure-migrate/aws-side
terraform init && terraform plan && terraform apply
```

**Windows (PowerShell):**

```powershell
cd "$HOME\aws-to-azure-migrate\aws-side"
terraform init; terraform plan; terraform apply
```

Expect 10 resources. After apply, save these outputs — you'll need them in Part 3:

```bash
terraform output ec2_public_ip
terraform output migrate_access_key_id
terraform output -raw migrate_secret_access_key
```

> Wait 5 minutes after apply before connecting via RDP — Windows instances take time to fully initialize.

---

## Part 2 — Build the Azure Target Environment

### `terraform.tfvars`

```hcl
yourname = "dylan"
location = "East US"
```

### `main.tf` (key resources)

```hcl
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.0" }
  }
}

provider "azurerm" { features {} }

resource "azurerm_resource_group" "source" {
  name     = "rg-migrate-source-${var.yourname}"
  location = var.location
}

resource "azurerm_resource_group" "target" {
  name     = "rg-migrate-target-${var.yourname}"
  location = var.location
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-migrate-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.source.name
  address_space       = ["10.1.0.0/16"]
}

resource "azurerm_subnet" "main" {
  name                 = "snet-migrate"
  resource_group_name  = azurerm_resource_group.source.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.1.1.0/24"]
}

resource "azurerm_storage_account" "replication_cache" {
  name                     = "stmigrate${var.yourname}"
  resource_group_name      = azurerm_resource_group.source.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
  min_tls_version          = "TLS1_2"
}

resource "azurerm_recovery_services_vault" "main" {
  name                = "rsv-migrate-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.source.name
  sku                 = "Standard"
  soft_delete_enabled = false
}
```

Deploy:

```bash
cd ~/aws-to-azure-migrate/azure-side
terraform init && terraform plan && terraform apply
```

> **Note:** The Azure Migrate project must be created **manually in the portal** — it is not supported by the Terraform AzureRM provider. Search "Azure Migrate" → Create project → select `rg-migrate-source-dylan`.

---

## Part 3 — Deploy and Configure the Azure Migrate Appliance

The appliance is a pre-built VM that bridges your AWS environment and the Azure Migrate service. It handles discovery and assessment data collection.

### Create the Appliance VM via Terraform

Add to `azure-side/main.tf`:

```hcl
resource "azurerm_windows_virtual_machine" "appliance" {
  name                  = "vm-mig-appl-${var.yourname}"
  location              = var.location
  resource_group_name   = azurerm_resource_group.source.name
  size                  = "Standard_A4_v2"
  admin_username        = "migrateadmin"
  admin_password        = var.appliance_admin_password
  network_interface_ids = [azurerm_network_interface.appliance.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = 80
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-Datacenter"
    version   = "latest"
  }
}
```

> **Why Standard_A4_v2?** The appliance requires at least 4 vCPUs and 8GB RAM. A4_v2 meets this at ~$0.40/hour.

### Install and Register the Appliance

1. In the Azure portal → **Azure Migrate** → **Discover** → select **Yes, with another cloud provider**
2. Generate a project key and **save it** — you'll paste it into the appliance config manager
3. Download the appliance installer (~1.5GB zip)
4. RDP into the appliance VM, extract the zip, and run `AzureMigrateInstaller.ps1` as Administrator
5. Answer the installer prompts: Physical/other → Azure Public → Public endpoint
6. In the configuration manager that opens, paste your project key and sign in with your Azure account

### Add Credentials and Start Discovery

Add two credential sets in the appliance config manager:

| Credential | Friendly Name | Username | Password |
|------------|--------------|---------|---------|
| AWS IAM | `awsmigratesvc` | Access Key ID (from Terraform output) | Secret Access Key (from Terraform output) |
| Windows Admin | `ec2winadmin` | `Administrator` | Password from `aws-side/terraform.tfvars` |

Add the **EC2 public IP** as the discovery source — not the private IP. There is no VPN between AWS and Azure so the private IP is unreachable from the appliance.

Click **Start discovery**. Discovery takes 5–15 minutes.

---

## Part 4 — Assess the EC2 Instance

1. Azure portal → Azure Migrate → **Assess** → Azure VM
2. Set sizing criteria to **Performance-based**
3. Create a new group, add your EC2 instance, and create the assessment
4. Review results — the instance should show **Ready for Azure**

The assessment recommends an Azure VM size based on actual CPU and memory utilization and provides a monthly cost estimate.

---

## Part 5 — Set Up the Replication Appliance and Replicate

The replication appliance (Configuration Server) is separate from the discovery appliance and handles disk-level replication via Azure Site Recovery.

Deploy a dedicated Windows Server 2022 VM for it (minimum Standard_A4_v2, must be 2022 — 2019 will fail), then:

1. In Azure Migrate → **Migration and modernization** → **Discover** → download the replication appliance installer
2. Run `DRInstaller.ps1` inside the RDP session and register with your Recovery Services Vault
3. Once registered, click **Replicate** and configure:
   - Source: your EC2 instance
   - Target resource group: `rg-migrate-target-dylan`
   - Replication storage: `stmigrate-dylan`
   - Target VNet/subnet: `vnet-migrate-dylan / snet-migrate`

Initial replication of a 30GB disk takes **30–45 minutes**. Wait for status to show **Protected** before proceeding.

---

## Part 6 — Test Migration and Cutover

### Test Migration (Recommended)

1. In replicating machines, click your EC2 instance → **Test migration**
2. Azure creates a temporary copy of the VM — RDP in and verify it boots correctly
3. Click **Clean up test migration** when done

### Cutover

1. Click **Migrate** → confirm
2. Azure finalizes replication and creates the target VM (~5–10 minutes)
3. Navigate to `rg-migrate-target-dylan` and attach a public IP to the new VM
4. RDP in using the original `Administrator` credentials from AWS

✅ Migration complete.

---

## Verification Checklist

- [ ] EC2 instance running in AWS console
- [ ] Azure Migrate project exists with appliance registered
- [ ] EC2 instance visible in discovered machines list
- [ ] Assessment shows **Ready for Azure**
- [ ] Replication status shows **Protected**
- [ ] Test migration succeeded and cleaned up
- [ ] Cutover complete — migrated VM visible in target resource group
- [ ] RDP to migrated VM succeeds with original credentials
- [ ] Hostname and OS version match the source EC2 instance

---

## Troubleshooting

| Error | Cause | Resolution |
|-------|-------|------------|
| EC2 not discovered | AWS credentials entered incorrectly | Re-enter access key and secret in appliance config manager |
| Discovery shows 0 machines | Region mismatch | Verify the region in the appliance matches the EC2 region |
| Error 322008 — Mobility Service install failed | Appliance trying to reach private IP | Use the EC2 **public IP** as the discovery source — there's no VPN between AWS and Azure |
| Replication stuck at 0% | Storage account inaccessible | Verify `stmigrate-dylan` is in the same subscription and region as the Migrate project |
| RDP to migrated VM fails | NSG not attached to VM NIC | Attach `nsg-migrate-target-dylan` to the VM's NIC in the portal |
| `terraform apply` fails — AMI not found | AMI ID invalid for selected region | Run: `aws ec2 describe-images --region us-east-1 --owners amazon --filters "Name=name,Values=Windows_Server-2022-English-Full-Base-*" "Name=state,Values=available" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text` |
| Recovery Services Vault won't delete | Vault still has replication items | Go to portal → RSV → Replication items → delete all, then retry `terraform destroy` |

---

## Teardown

Destroy in this order to avoid dependency errors:

**Step 1 — Stop replication** (if active): Azure Migrate → Replicating machines → Stop replication

**Step 2 — Destroy AWS resources:**

```bash
cd ~/aws-to-azure-migrate/aws-side
terraform destroy
```

**Step 3 — Destroy Azure resources:**

```bash
cd ~/aws-to-azure-migrate/azure-side
terraform destroy
```

**Step 4 — Delete the target resource group** (contains migrated VM, not tracked by Terraform):

```bash
az group delete --name rg-migrate-target-dylan --yes
```

> ⚠️ Always use `terraform destroy` before deleting resource groups manually. Deleting in the portal without destroying first will cause state file conflicts.

---

## Summary

- ✅ Provisioned a Windows Server EC2 instance on AWS using Terraform
- ✅ Built the Azure target environment including VNet, Recovery Services Vault, and storage cache
- ✅ Deployed and registered the Azure Migrate discovery appliance
- ✅ Discovered and assessed the EC2 instance — confirmed Ready for Azure
- ✅ Configured the replication appliance and replicated disk data from AWS to Azure
- ✅ Performed a test migration and validated the VM boots correctly
- ✅ Executed cutover and verified the migrated VM in Azure using original credentials

---

## Next Steps

| Enhancement | Description |
|-------------|-------------|
| **Private Endpoint for Replication** | Route replication traffic privately instead of over the public internet |
| **Azure Bastion** | Replace public RDP on the migrated VM with secure browser-based access |
| **DNS Update** | Update any DNS records pointing to the old EC2 public IP to the new Azure VM IP |
| **NSG Hardening** | Restrict RDP access to specific source IPs on the migrated VM |
| **Azure Monitor** | Enable monitoring and alerting on the migrated VM post-cutover |

---

*© Dylan Bryson — Cloud Security Specialist*
