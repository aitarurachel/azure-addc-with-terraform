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

This file defines every Azure resource and the extension that installs AD DS. Paste the full block in az-ad-vm/main.tf

---

### Step 3: Populate `variables.tf`

This is where you define what inputs the project accepts. Paste the full block in az-ad-vm/variables.tf

---

### Step 4: Populate `terraform.tfvars`

> ⚠️ **Add `terraform.tfvars` to your `.gitignore`**. It contains sensitive credentials and should never be committed, but for this lab, I've excluded it from the .gitignore.

This is where you fill in the variables declared in variables.tf. Paste the full block in az-ad-vm/terraform.tfvars

**DSRM Password note:** The Directory Services Restore Mode password is separate from the VM admin password. Store it in a password manager, it is only needed for AD recovery operations and cannot be retrieved after deployment.

---

### Step 5: Populate `outputs.tf`

After terraform apply finishes, this file tells Terraform what information to print to your terminal. Paste the full block in az-ad-vm/outputs.tf

---

### Step 6: Deploy

Run these commands from the `~/repos/az-ad-vm` directory:

```bash
terraform init
terraform plan
terraform apply
```

terraform init
<br>
<img width="931" height="380" alt="terra-init" src="https://github.com/user-attachments/assets/3255f79e-d20a-468a-837d-eab81260698c" />
<br>

terraform plan
<br>
<img width="1063" height="514" alt="terra-plan" src="https://github.com/user-attachments/assets/de529ceb-d11b-4272-b4a2-547e144d4c59" />
<br>

terraform apply
<br>
<img width="1112" height="375" alt="terra-apply2" src="https://github.com/user-attachments/assets/bd2d25fd-f4e5-48d1-ac67-afbcd8bc1c14" />
<br>

Resource group created in Azure portal
<br>
<img width="1454" height="489" alt="terra-resourcegroup" src="https://github.com/user-attachments/assets/35a8d7c8-ae4b-4bf1-84a9-21e97998d146" />
<br>
<br>

Deployment takes approximately **5–8 minutes** for the VM, plus an additional **3–5 minutes** for the Custom Script Extension to install AD DS and trigger an automatic reboot.

---

## Connecting via RDP

Get the public IP once the apply completes:

```bash
terraform output public_ip
```
<br>

<img width="509" height="31" alt="terra-public ip" src="https://github.com/user-attachments/assets/8940944a-e346-4ab3-b71c-6c95bb21b94c" />
<br>
<br>

<br>
<img width="1123" height="345" alt="terra-rdp" src="https://github.com/user-attachments/assets/080a3418-e564-4706-ab93-3fa49fbadc0e" />
<br>
<br>

| Method | Username | When to use |
|---|---|---|
| Domain prefix | `CORP\adadmin` | Use this first — standard post-promotion |
| UPN format | `adadmin@corp.charles.com` | If domain prefix fails |
| Local account | `.\adadmin` | Only if AD promotion failed entirely |

> ⏳ **Wait 5–10 minutes after `terraform apply` completes before connecting.** The VM reboots automatically after AD DS installs. Connecting too early may result in a failed or black-screen RDP session.
> 
<br>

<img width="1295" height="840" alt="terra-ad-ds" src="https://github.com/user-attachments/assets/1622a4ce-e4c5-4bbb-8caf-70cfff25afac" />
<br>

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
<br>

<img width="457" height="226" alt="verify1" src="https://github.com/user-attachments/assets/a96622bf-0775-44b2-9997-a373f4138837" />
<br>
<br>

<img width="649" height="695" alt="verify2" src="https://github.com/user-attachments/assets/d148ef40-4152-4351-9a8e-1fb0c2880339" />
<br>
<br>

<img width="764" height="606" alt="verify3" src="https://github.com/user-attachments/assets/c16ebc2b-3baa-4a3f-93d8-f13d2e0092fb" />
<br>
<br>

<img width="615" height="250" alt="verify4" src="https://github.com/user-attachments/assets/7a4e0aa2-20e8-4a71-8a53-a3f6bc06f042" />
<br>

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
<br>

<img width="958" height="414" alt="terra-destroy" src="https://github.com/user-attachments/assets/7371c826-7aee-4ce3-bcfd-964a30767f08" />
<br>

This removes the resource group and everything inside it: VM, managed disk, NIC, public IP, NSG, VNet, and subnet.

---

## Trade-offs

**RDP open to `0.0.0.0/0`**: the NSG allows RDP from any source IP, which is acceptable for a short-lived lab but is a deliberate security concession for convenience. In any shared or long-lived environment, this should be scoped to your IP address.

**Credentials in `terraform.tfvars`**: passwords are stored in a plaintext local file. This is acceptable for a lab, but it isn't a secrets management strategy.

**CustomScriptExtension vs cloud-init**: the extension approach is quick and Azure-native, but it runs as a single opaque command with limited retry logic. If the PowerShell fails mid-execution, the extension status shows `Failed` and you re-run `apply` to retry, there is no partial rollback.

---

## What I'd Do Differently in Production

- **Replace the open NSG with a Just-In-Time (JIT) access policy** or scope the RDP rule to a known IP range. Better yet, use Azure Bastion to eliminate the public RDP surface entirely.
- **Store passwords in Azure Key Vault** and reference them in Terraform using the `azurerm_key_vault_secret` data source; no credentials in flat files.
- **Use a private IP only** with a VPN Gateway or ExpressRoute for management access rather than exposing a public IP to the internet.
- **Modularize the Terraform** break networking, compute, and AD configuration into reusable child modules so the patterns can be composed into larger hub-and-spoke or multi-region topologies.
- **Add a second domain controller** for redundancy; a single DC is a single point of failure for the entire domain.
- **Parameterize the VM size and OS SKU** so the same code can be promoted from a lab-grade `Standard_D2s_v3` to a production-grade SKU without code changes.

---
