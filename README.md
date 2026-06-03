# Azure Active Directory Domain Controller — Terraform Deployment Lab

> Deploy a fully functional Active Directory Domain Controller on Azure in under 10 minutes using Infrastructure as Code.

---

## What Problem Does This Solve?

Standing up an Active Directory lab environment manually through the Azure Portal is slow, error-prone, and impossible to reproduce consistently. Every click is undocumented. This project solves that by codifying the entire stack: networking, compute, and AD DS promotion into a single `terraform apply`. It gives you a repeatable, version-controlled lab environment that can be torn down and rebuilt on demand.

---

## Project Overview

This lab provisions a **Windows Server 2022 virtual machine** on Microsoft Azure and automatically configures it as an **Active Directory Domain Controller** using a Terraform `CustomScriptExtension`. The extension fires a PowerShell command at first boot that installs the `AD-Domain-Services` role and promotes the server to a new forest. No manual steps are required inside the VM.

### What Gets Deployed

| Resource | Name Pattern | Purpose |
|---|---|---|
| Resource Group | `rg-ad-{yourname}` | Logical container for all resources |
| Virtual Network | `vnet-ad-{yourname}` | Isolated network — `10.0.0.0/16` |
| Subnet | `snet-ad` | VM subnet — `10.0.1.0/24` |
| Public IP | `pip-ad-{yourname}` | Static IP for RDP access |
| NSG | `nsg-ad-{yourname}` | Inbound RDP rule (port 3389) |
| NIC | `nic-ad-{yourname}` | NIC bound to static private IP `10.0.1.4` |
| Windows VM | `vm-ad-{yourname}` | Windows Server 2022 Datacenter, Standard_D2s_v3 |
| Custom Script Extension | `install-ad-ds` | Installs AD DS + promotes to domain controller |

### Architecture

<br>
<img width="904" height="772" alt="addc architectural diagram" src="https://github.com/user-attachments/assets/bb39b827-2586-4887-b58a-bf648bfad22e" />
<br>

> After deployment, the VM reboots automatically to complete AD DS promotion. Authenticate using `CORP\adadmin` or `adadmin@corp.charles.com` — not the local account format.

---

## Tools Used & Why

| Tool | Why |
|---|---|
| **Terraform (azurerm ~> 3.0)** | Declarative IaC with a mature Azure provider. One `apply` creates and one `destroy` tears down the full stack |
| **Azure Custom Script Extension** | Runs PowerShell at VM boot without needing a bastion host, Ansible, or manual RDP. Keeps everything in a single Terraform apply |
| **Windows Server 2022 Datacenter** | Current-gen OS with the latest AD DS functional level support out of the box |
| **Azure CLI** | Required for `az login` authentication before Terraform can communicate with the Azure control plane |

---

## Prerequisites

Before running this lab, confirm the following are in place:

- [ ] **Azure CLI** installed and authenticated — `az login`
- [ ] **Terraform v1.3 or later** installed ([install guide](https://developer.hashicorp.com/terraform/downloads))
- [ ] An **active Azure subscription** with permissions to create resource groups and VMs
- [ ] A local directory to hold the Terraform project files

---

## File Structure

```
az-ad-vm/
├── main.tf           # All Azure resources + Custom Script Extension
├── variables.tf      # Input variable declarations
├── terraform.tfvars  # Your environment-specific values (gitignore this)
└── outputs.tf        # Public IP, domain name, admin username
```

---

## Step-by-Step Deployment

### Step 1: Scaffold the project

```bash
mkdir -p ~/repos/az-ad-vm && cd ~/repos/az-ad-vm
touch main.tf variables.tf outputs.tf terraform.tfvars
```

---

### Step 2: Populate `main.tf`

This file defines every Azure resource and the extension that installs AD DS. Paste the full block below:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = "rg-ad-${var.yourname}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
  tags                = var.tags
}

resource "azurerm_subnet" "main" {
  name                 = "snet-ad"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "main" {
  name                = "pip-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_network_security_group" "main" {
  name                = "nsg-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "allow-rdp"
    priority                   = 1000
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

resource "azurerm_network_interface" "main" {
  name                = "nic-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.1.4"
    public_ip_address_id          = azurerm_public_ip.main.id
  }

  tags = var.tags
}

resource "azurerm_network_interface_security_group_association" "main" {
  network_interface_id      = azurerm_network_interface.main.id
  network_security_group_id = azurerm_network_security_group.main.id
}

resource "azurerm_windows_virtual_machine" "main" {
  name                = "vm-ad-${var.yourname}"
  computer_name       = "ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  size                = "Standard_D2s_v3"
  admin_username      = "adadmin"
  admin_password      = var.admin_password

  network_interface_ids = [azurerm_network_interface.main.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 127
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-Datacenter"
    version   = "latest"
  }

  additional_unattend_content {
    content  = "<AutoLogon><Password><Value>${var.admin_password}</Value></Password><Enabled>true</Enabled><LogonCount>1</LogonCount><Username>adadmin</Username></AutoLogon>"
    setting  = "AutoLogon"
  }

  tags = var.tags
}

resource "azurerm_virtual_machine_extension" "ad_setup" {
  name                 = "install-ad-ds"
  virtual_machine_id   = azurerm_windows_virtual_machine.main.id
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.10"

  settings = jsonencode({
    commandToExecute = "powershell -ExecutionPolicy Unrestricted -Command \"Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools; Import-Module ADDSDeployment; Install-ADDSForest -DomainName '${var.domain_name}' -DomainNetbiosName '${var.domain_netbios}' -ForestMode 'WinThreshold' -DomainMode 'WinThreshold' -InstallDns:$true -SafeModeAdministratorPassword (ConvertTo-SecureString '${var.dsrm_password}' -AsPlainText -Force) -Force:$true\""
  })

  tags = var.tags
}
```

---

### Step 3: Populate `variables.tf`

```hcl
variable "yourname" {
  description = "Your name — used to make resource names unique."
  type        = string
}

variable "location" {
  description = "Azure region."
  type        = string
  default     = "eastus"
}

variable "admin_password" {
  description = "Local admin password for the VM."
  type        = string
  sensitive   = true
}

variable "dsrm_password" {
  description = "Directory Services Restore Mode password for AD DS."
  type        = string
  sensitive   = true
}

variable "domain_name" {
  description = "Fully qualified domain name (e.g. corp.example.com)."
  type        = string
  default     = "corp.example.com"
}

variable "domain_netbios" {
  description = "NetBIOS name for the domain (max 15 characters)."
  type        = string
  default     = "CORP"
}

variable "tags" {
  description = "Tags to apply to all resources."
  type        = map(string)
  default = {
    project = "ad-lab"
  }
}
```

---

### Step 4: Populate `terraform.tfvars`

> ⚠️ **Add `terraform.tfvars` to your `.gitignore`**. It contains sensitive credentials and should never be committed.

```hcl
yourname       = "charles"
location       = "eastus"
admin_password = "YourPassword123!"
dsrm_password  = "YourDSRMPassword123!"
domain_name    = "corp.charles.com"
domain_netbios = "CORP"
```

**DSRM Password note:** The Directory Services Restore Mode password is separate from the VM admin password. Store it in a password manager, it is only needed for AD recovery operations and cannot be retrieved after deployment.

---

### Step 5: Populate `outputs.tf`

```hcl
output "public_ip" {
  description = "Public IP — use this to RDP into the domain controller"
  value       = azurerm_public_ip.main.ip_address
}

output "domain_name" {
  description = "Active Directory domain name"
  value       = var.domain_name
}

output "admin_username" {
  description = "Local admin username"
  value       = "adadmin"
}
```

---

### Step 6: Deploy

Run these commands from the `~/repos/az-ad-vm` directory:

```bash
terraform init
terraform plan
terraform apply
```

Deployment takes approximately **5–8 minutes** for the VM, plus an additional **3–5 minutes** for the Custom Script Extension to install AD DS and trigger an automatic reboot.

---

## Connecting via RDP

Get the public IP once the apply completes:

```bash
terraform output public_ip
```

| Method | Username | When to use |
|---|---|---|
| Domain prefix | `CORP\adadmin` | Use this first — standard post-promotion |
| UPN format | `adadmin@corp.charles.com` | If domain prefix fails |
| Local account | `.\adadmin` | Only if AD promotion failed entirely |

> ⏳ **Wait 5–10 minutes after `terraform apply` completes before connecting.** The VM reboots automatically after AD DS installs. Connecting too early may result in a failed or black-screen RDP session.

---

## Verifying AD DS

Once connected via RDP, open **PowerShell as Administrator** and run:

```powershell
# Confirm the NTDS service is running
Get-Service NTDS | Select-Object Name, Status

# Confirm domain details
Get-ADDomain

# List domain controllers
Get-ADDomainController -Filter *

# Verify DNS is resolving the domain
Resolve-DnsName corp.charles.com
```

All four commands should return without errors. `Get-ADDomain` will show the full forest and domain functional levels, confirming a successful deployment.

---

## Troubleshooting

Check the extension provisioning status from your local machine:

```bash
az vm extension show \
  --resource-group rg-ad-charles \
  --vm-name vm-ad-charles \
  --name install-ad-ds \
  --query "provisioningState" \
  --output tsv
```

| Symptom | Cause | Fix |
|---|---|---|
| RDP password rejected | VM rebooted into domain mode — local auth no longer works | Use `CORP\adadmin` or UPN format |
| Extension status: Failed | AD DS install failed mid-run | Check `C:\WindowsAzure\Logs` on the VM; re-run `terraform apply` to retry |
| RDP black screen | VM still rebooting after AD DS install | Wait 5 minutes and retry |
| `Get-ADDomain` not found | AD DS module not loaded in session | Run `Import-Module ActiveDirectory` then retry |

---

## Teardown

To destroy all resources when done:

```bash
terraform destroy
```

This removes the resource group and everything inside it: VM, managed disk, NIC, public IP, NSG, VNet, and subnet.

---

## Trade-offs

**RDP open to `0.0.0.0/0`**: the NSG allows RDP from any source IP, which is acceptable for a short-lived lab but is a deliberate security concession for convenience. In any shared or long-lived environment, this should be scoped to your IP address.

**Credentials in `terraform.tfvars`**: passwords are stored in a plaintext local file. This is acceptable for a lab, but it isn't a secrets management strategy.

**CustomScriptExtension vs cloud-init**: the extension approach is quick and Azure-native, but it runs as a single opaque command with limited retry logic. If the PowerShell fails mid-execution, the extension status shows `Failed` and you re-run `apply` to retry — there is no partial rollback.

---

## What I'd Do Differently in Production

- **Replace the open NSG with a Just-In-Time (JIT) access policy** or scope the RDP rule to a known IP range. Better yet, use Azure Bastion to eliminate the public RDP surface entirely.
- **Store passwords in Azure Key Vault** and reference them in Terraform using the `azurerm_key_vault_secret` data source; no credentials in flat files.
- **Use a private IP only** with a VPN Gateway or ExpressRoute for management access rather than exposing a public IP to the internet.
- **Modularize the Terraform** break networking, compute, and AD configuration into reusable child modules so the patterns can be composed into larger hub-and-spoke or multi-region topologies.
- **Add a second domain controller** for redundancy; a single DC is a single point of failure for the entire domain.
- **Parameterize the VM size and OS SKU** so the same code can be promoted from a lab-grade `Standard_D2s_v3` to a production-grade SKU without code changes.

---
