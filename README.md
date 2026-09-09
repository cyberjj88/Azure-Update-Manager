Video of the lab
https://www.loom.com/share/474721d8bf344733881207368cfd8301


# 🔄 Lab - Azure Update Manager: Patch Management & Compliance Reporting


**Author:** Jair Smith
**Difficulty:** 🔴 Advanced
**Deploy Time:** 15-20 minutes (Terraform) - fully automated
**Estimated Cost:** ~$0.17/hr (~$4/day) - always run `terraform destroy` when finished
**Platform:** Azure - Windows Server 2022 - Modular Terraform

| Certification Alignment | Tools Required |
|---|---|
| AZ-104 · SC-200 · CompTIA Security+ | Terraform >= 1.5.0 · Azure CLI · PowerShell 7+ · Az PowerShell Module |

| Career Relevance |
|---|
| Cloud Operations Engineer · Security Engineer · SOC Analyst · Infrastructure Engineer |

| Relationship to Other Labs |
|---|
| **Fully standalone** - independent from Lab 1 (NTFS) and Lab 2 (RBAC). Run this first, last, or by itself. |

---

## 📋 Overview

Unpatched systems are one of the most common root causes of security incidents. A Windows Server missing current Critical and Security patches is a known vulnerability. The challenge at scale is not applying patches to one server - it is knowing which machines across an entire environment are missing which patches, enforcing a consistent schedule, and producing documentation that proves compliance to auditors.

Azure Update Manager is the cloud-native replacement for WSUS. It is agentless for Azure VMs, integrates with Azure Policy for automatic enrollment, maintains a compliance record per machine, and supports both scheduled patching and manual approval workflows.

This lab builds a complete patch management pipeline from zero: infrastructure deployment, policy-based enrollment, maintenance window configuration, on-demand assessment, compliance validation, and structured report export. Every step maps directly to what a cloud operations team does in production.

---

## 🏗️ Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AZURE - East US Region                                │
│                    Resource Group: rg-aumlab                             │
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │              modules/networking                                  │  │
│   │              VNet: vnet-aumlab (10.0.0.0/16)                     │  │
│   │              Subnet: snet-aumlab (10.0.1.0/24)                   │  │
│   │              NSG: Allow RDP from your IP only                    │  │
│   │                                                                  │  │
│   │   ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐  │  │
│   │   │  DC01            │  │  WS01            │  │  WS02       │  │  │
│   │   │  10.0.1.4 Static │  │  Dynamic IP      │  │  Dynamic IP │  │  │
│   │   │  Win Server 2022 │  │  Win Server 2022 │  │  Win Server │  │  │
│   │   │                  │  │                  │  │  2022       │  │  │
│   │   │  Domain          │  │  Member Server   │  │  Member     │  │  │
│   │   │  Controller      │  │  aumlab.local    │  │  Server     │  │  │
│   │   │  aumlab.local    │  │                  │  │  aumlab.    │  │  │
│   │   │  DNS Server      │  │  DNS: 10.0.1.4   │  │  local      │  │  │
│   │   └──────────────────┘  └──────────────────┘  └─────────────┘  │  │
│   │              │                    │                    │         │  │
│   └──────────────┼────────────────────┼────────────────────┼─────────┘  │
│                  │                    │                    │            │
│   ┌──────────────▼────────────────────▼────────────────────▼─────────┐  │
│   │              modules/update-manager                              │  │
│   │                                                                  │  │
│   │   ┌────────────────────────────────────────────────────────┐    │  │
│   │   │  Azure Policy Assignment (policy 59efceea)             │    │  │
│   │   │  Scope: rg-aumlab                                      │    │  │
│   │   │  Effect: Auto-enroll ALL VMs into periodic assessment  │    │  │
│   │   │  New VMs added to rg-aumlab are enrolled automatically │    │  │
│   │   └────────────────────────────────────────────────────────┘    │  │
│   │                                                                  │  │
│   │   ┌────────────────────────────────────────────────────────┐    │  │
│   │   │  Maintenance Configuration: aum-weekly-patches         │    │  │
│   │   │  Schedule: Weekly - Fridays 02:00 EST - 3hr window     │    │  │
│   │   │  Scope: InGuestPatch (inside OS, not hypervisor)       │    │  │
│   │   │  Classifications: Critical, Security, UpdateRollup     │    │  │
│   │   │  Reboot: IfRequired                                    │    │  │
│   │   └────────────────────────────────────────────────────────┘    │  │
│   │                        │                                        │  │
│   │           ┌────────────┼────────────┐                          │  │
│   │           ▼            ▼            ▼                          │  │
│   │   [Assignment:DC01] [Assignment:WS01] [Assignment:WS02]        │  │
│   │   Links each VM to the weekly maintenance schedule             │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│   modules/keyvault                                                      │
│   Key Vault: kv-aum-[random] - stores VM admin password                │
│                                                                          │
│   scripts/validate-lab.ps1                                              │
│   Queries compliance state per VM - exports aum-compliance-report.json  │
└──────────────────────────────────────────────────────────────────────────┘

  AUTOMATION FLOW:
  terraform apply (15-20 min)
    Module 1: networking  - VNet, subnet, NSG
    Module 2: keyvault    - Key Vault, secret, RBAC
    Module 3: compute     - 3 VMs, DC01 promotion, WS01/WS02 domain join
    Module 4: update-mgr  - Policy assignment, maintenance config, 3 assignments

  PATCH MANAGEMENT FLOW:
  Policy (auto-enroll) → Assessment (what is missing) → Maintenance Window (when to patch)
       → Assignments (which VMs) → Patch Applied → validate-lab.ps1 → JSON Report

  ASSESSMENT vs PATCHING:
  Policy assignment  = enrollment for periodic assessment only (no patches applied)
  Maintenance assign = links VM to patch schedule (patches applied on schedule)
  Both are required for a VM to be assessed AND automatically patched
```

> **Key Concept:** Assessment and patching are two separate operations in Azure Update Manager. A VM can show compliance data (assessment) without ever being automatically patched if it is not linked to a maintenance window. Both the policy assignment and the maintenance assignment are required for the complete patch management lifecycle.

---

## 📁 Lab Series Context

| Lab | What It Deploys | Relationship |
|---|---|---|
| **Lab 1 - NTFS File Server** | DC01, FS01, CLIENT01, VNet, NSG, Key Vault in RG-FileServerLab | Standalone |
| **Lab 2 - Azure RBAC** | 3 role assignments on FS01 only - no new VMs | Depends on Lab 1 |
| **AUM Lab - Azure Update Manager (this lab)** | DC01, WS01, WS02, VNet, Key Vault in rg-aumlab | Fully standalone |

> 💡 **Remote state:** If you have already done Lab 1 or Lab 2, reuse the same `RG-TerraformState` storage account - just use the key `aum-lab.tfstate`. If this is your first lab, the remote state setup commands in Step 2 create everything from scratch.

---

## 🎯 Objectives

By completing this lab, you will be able to:

- [x] Build a **modular Terraform** infrastructure with four independently managed modules
- [x] Use **Azure Policy** to auto-enroll VMs into periodic patch assessment
- [x] Configure a **Maintenance Window** defining when, what, and how to patch
- [x] Understand the difference between **assessment and patching** as separate operations
- [x] Trigger an **on-demand patch assessment** via the Azure REST API
- [x] Write a **compliance validation script** that exports structured JSON output
- [x] Produce a **compliance report** suitable for feeding a SIEM, ticketing system, or dashboard

---

## 💼 Why This Lab Matters

| Skill | Why It Matters in a Real Environment |
|---|---|
| **Modular Terraform** | Module-based Terraform is the professional standard. Each module owns one concern, is independently testable, and reusable across environments. This is the pattern used in real engineering teams. |
| **Azure Policy for enrollment** | Policy-based enrollment means you define the rule once and every matching VM is automatically enrolled - including VMs created in the future. Without policy, each VM must be enrolled manually. |
| **Maintenance Windows** | A maintenance window is a contract with the business: patches are applied during this window, with this reboot behavior, targeting these update classifications. Without a defined window, patches either never get applied or cause unplanned outages. |
| **Assessment vs patching** | These are two distinct operations. A machine can show compliance data without ever receiving automated patches. Understanding this prevents both under-patching and unexpected reboots. |
| **On-demand assessment** | Scheduled assessments run on a cadence. On-demand assessment immediately surfaces compliance state after a new deployment or a new CVE disclosure. This is what ops teams do when a critical vulnerability is announced. |
| **Structured compliance export** | A compliance report is not optional in regulated industries. The JSON export pattern feeds SIEMs, ticketing systems, and executive dashboards in production environments. |

---

## 🌐 The Real-World Scenario

A new CVE is disclosed with a CVSS score of 9.8. The security team asks: which of our servers are missing the patch that addresses it?

The ops engineer triggers an on-demand assessment across all VMs and pulls the compliance report within 15 minutes. Affected machines are identified. An emergency maintenance window is created targeting Critical updates. Patching runs, machines reboot as required, and a new compliance report confirms all machines are now compliant. The entire workflow is automated, documented, and auditable.

That is what this lab teaches.

---

## ✅ Prerequisites

Verify all tools and providers are ready before starting:

```powershell
terraform -version     # Must be >= 1.5.0
az version             # Azure CLI - any recent version
pwsh --version         # PowerShell 7+
Install-Module Az -Scope CurrentUser -Force   # Az PowerShell module for validate-lab.ps1

# Confirm correct subscription
az account show

# Register required Azure resource providers
# Microsoft.Maintenance is needed for azurerm_maintenance_configuration
# Microsoft.GuestConfiguration is needed for in-guest patching
# Both must show Registered before terraform apply
az provider register --namespace Microsoft.Maintenance
az provider register --namespace Microsoft.GuestConfiguration
az provider show --namespace Microsoft.Maintenance --query registrationState -o tsv
az provider show --namespace Microsoft.GuestConfiguration --query registrationState -o tsv
# Both must show: Registered
```

- [ ] Terraform >= 1.5.0
- [ ] Azure CLI (any recent version)
- [ ] PowerShell 7+
- [ ] Az PowerShell module installed
- [ ] `Microsoft.Maintenance` provider shows `Registered`
- [ ] `Microsoft.GuestConfiguration` provider shows `Registered`
- [ ] Your public IP address (find at [whatismyip.com](https://whatismyip.com))

---

## 🏷️ Lab Variables and Reference

| Field | Value |
|---|---|
| **Domain** | `aumlab.local` |
| **NetBIOS Name** | `AUMLAB` |
| **Region** | East US |
| **Resource Group** | `rg-aumlab` |
| **VNet** | `vnet-aumlab` - `10.0.0.0/16` |
| **Subnet** | `snet-aumlab` - `10.0.1.0/24` |
| **DC01 Static IP** | `10.0.1.4` |
| **VM Size** | `Standard_B2s` |
| **VM Admin Username** | `labadmin` |
| **Patch Classifications** | Critical - Security - UpdateRollup |
| **Maintenance Window** | Weekly - 3 hour duration - reboot IfRequired |
| **Policy Definition** | `59efceea-0c96-497e-a4a1-4eb2290dac15` |
| **Compliance Report** | `aum-compliance-report.json` |

---

## 💡 Why Each Component Exists

| Component | What It Does | Why It Is Needed |
|---|---|---|
| **Modular Terraform (4 modules)** | Networking, keyvault, compute, and update-manager are separate units | Each module owns exactly one concern. Changing VM size only requires editing `modules/compute/main.tf` - networking and update-manager are untouched. |
| **modules/networking** | VNet, subnet, NSG, subnet-NSG association | Creates the private network. The NSG-subnet association is a separate resource - without it the NSG exists but applies to nothing. |
| **modules/keyvault** | Key Vault with RBAC model, admin password secret | `enable_rbac_authorization = true` is required. Without it, role assignments are silently ignored and all secret operations return 403. |
| **modules/compute** | Three VMs, DC01 promotion extension, domain join extensions | DC01 gets a static IP so WS01 and WS02 can always find DNS. `depends_on` on join extensions forces them to wait for DC01 promotion - without this Terraform runs all extensions in parallel and domain join fails. |
| **modules/update-manager** | Azure Policy assignment, Maintenance Configuration, 3 Maintenance Assignments | Three distinct concepts: Policy handles enrollment, Maintenance Configuration handles the schedule, Maintenance Assignments link the schedule to each VM. |
| **Azure Policy (59efceea)** | Auto-enrolls all VMs in rg-aumlab into periodic assessment | Without this policy, each VM must be individually enrolled. With it, any VM added to the resource group is automatically enrolled - no manual action required. |
| **azurerm_maintenance_configuration** | Defines WHEN patches run, WHICH classifications, and reboot behavior | `scope = InGuestPatch` means patches apply inside the guest OS. `in_guest_user_patch_mode = User` means this config owns the schedule. |
| **azurerm_maintenance_assignment (x3)** | Links the maintenance schedule to each individual VM | Assessment and patching are separate. Without assignments, VMs show compliance data but are never automatically patched. |
| **Static IP on DC01 (10.0.1.4)** | DC01 always has the same private IP | WS01 and WS02 point DNS at 10.0.1.4 during domain join. If this IP changed after a restart, DNS for `aumlab.local` would break. |
| **depends_on on join extensions** | Enforces ordering - DC01 promotion must finish before domain join attempts | Terraform's default parallel execution would start all extensions simultaneously. `depends_on` creates explicit ordering in the execution graph. |
| **validate-lab.ps1 JSON export** | Produces machine-readable compliance artifact alongside console output | JSON can be ingested by a SIEM, parsed by ServiceNow, or stored in Blob Storage for audit history. A real compliance programme needs both human-readable and machine-readable output. |

---

## 📁 Project Structure

```
azure-update-manager-lab/
├── backend.tf                          - Remote state configuration
├── versions.tf                         - Provider version requirements
├── variables.tf                        - All input variables
├── main.tf                             - Root module wiring all 4 child modules
├── outputs.tf                          - VM public IPs and Key Vault name
├── terraform.tfvars.example            - Safe template, commit this
├── terraform.tfvars                    - Your real values, never commit this
├── .gitignore
├── modules/
│   ├── networking/main.tf              - VNet, subnet, NSG, NSG-subnet association
│   ├── keyvault/main.tf                - Key Vault with RBAC model + admin password secret
│   ├── compute/main.tf                 - 3 VMs, DC01 promotion, WS01/WS02 domain join
│   └── update-manager/main.tf          - Azure Policy + Maintenance Config + 3 Assignments
└── scripts/
    └── validate-lab.ps1                - Compliance validation + JSON report export
```

---

## 🔬 Step 1 - Create Your Project Folder

```powershell
New-Item -ItemType Directory -Path "$HOME\azure-update-manager-lab"
cd "$HOME\azure-update-manager-lab"
New-Item -ItemType Directory -Path modules/networking
New-Item -ItemType Directory -Path modules/keyvault
New-Item -ItemType Directory -Path modules/compute
New-Item -ItemType Directory -Path modules/update-manager
New-Item -ItemType Directory -Path scripts
```

---

## 🔬 Step 2 - Create the Root Terraform Files

---

### 📄 backend.tf

> Already have a state storage account from Lab 1 or Lab 2? Use the same one - just update the name below. The key `aum-lab.tfstate` is unique to this lab and will not conflict with other state files.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "RG-TerraformState"
    storage_account_name = "REPLACE_WITH_YOUR_STORAGE_ACCOUNT_NAME"
    container_name       = "tfstate"
    key                  = "aum-lab.tfstate"
    # Separate from ntfs-lab.terraform.tfstate and rbac-lab.terraform.tfstate
    # All three labs share the same container without affecting each other
  }
}
```

**First time using remote state - create your storage account now:**

```bash
az group create --name RG-TerraformState --location eastus

# Name must be globally unique, 3-24 chars, lowercase and numbers only
az storage account create --name REPLACE_WITH_UNIQUE_NAME \
    --resource-group RG-TerraformState --sku Standard_LRS

az storage container create --name tfstate --account-name REPLACE_WITH_UNIQUE_NAME
```

Then update `backend.tf` with that storage account name before running `terraform init`.

---

### 📄 versions.tf

> `azurerm 3.100+` is required for `azurerm_maintenance_configuration`. The `random` provider generates the Key Vault name suffix.

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm" version = "~> 3.100" }
    random  = { source = "hashicorp/random"  version = "~> 3.6"   }
  }
}
provider "azurerm" { features {} }
```

---

### 📄 variables.tf

> `admin_password` is marked `sensitive = true`. Terraform will never print it in plan or apply output. Set it as an environment variable, never in a file.

```hcl
variable "location"            { type=string  default="eastus" }
variable "resource_group_name" { type=string  default="rg-aumlab" }
variable "admin_username"      { type=string  default="labadmin" }
variable "admin_password" {
  type        = string
  sensitive   = true
  description = "Set as TF_VAR_admin_password env var - never in a file. Min 12 chars, upper+lower+number+symbol."
}
variable "allowed_rdp_ip" {
  type        = string
  description = "Your public IP in CIDR format - e.g. 1.2.3.4/32. Find at whatismyip.com."
}
variable "domain_name"    { type=string  default="aumlab.local" }
variable "domain_netbios" { type=string  default="AUMLAB" }
```

---

### 📄 main.tf

> Root module that calls all four child modules. Networking must exist before compute can attach to it. Key Vault must exist before compute can reference credentials. Update Manager runs after compute because it needs the VM IDs to create maintenance assignments.

```hcl
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

module "networking" {
  source              = "./modules/networking"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  allowed_rdp_ip      = var.allowed_rdp_ip
}

module "keyvault" {
  source              = "./modules/keyvault"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  admin_password      = var.admin_password
}

module "compute" {
  source              = "./modules/compute"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = module.networking.subnet_id
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  domain_name         = var.domain_name
  domain_netbios      = var.domain_netbios
}

# update_manager depends on compute outputs (dc01_id, ws01_id, ws02_id)
# Terraform infers this dependency automatically from the variable references
module "update_manager" {
  source              = "./modules/update-manager"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  resource_group_id   = azurerm_resource_group.main.id
  dc01_id             = module.compute.dc01_id
  ws01_id             = module.compute.ws01_id
  ws02_id             = module.compute.ws02_id
}
```

---

### 📄 outputs.tf

```hcl
output "dc01_public_ip" { value = module.compute.dc01_public_ip }
output "ws01_public_ip" { value = module.compute.ws01_public_ip }
output "ws02_public_ip" { value = module.compute.ws02_public_ip }
output "key_vault_name" { value = module.keyvault.key_vault_name }
output "resource_group" { value = azurerm_resource_group.main.name }
```

---

### 📄 terraform.tfvars.example

```hcl
location            = "eastus"
resource_group_name = "rg-aumlab"
admin_username      = "labadmin"
allowed_rdp_ip      = "YOUR_PUBLIC_IP/32"   # find at whatismyip.com - format: 1.2.3.4/32
domain_name         = "aumlab.local"
domain_netbios      = "AUMLAB"

# admin_password is NOT set here:
#   PowerShell: $env:TF_VAR_admin_password = "YourPassword123!"
#   Bash:       export TF_VAR_admin_password="YourPassword123!"
# Min 12 chars, upper + lower + number + symbol.
```

---

### 📄 .gitignore

```
terraform.tfvars
*.tfvars
!terraform.tfvars.example
terraform.tfstate
terraform.tfstate.backup
*.tfstate
.terraform/
.terraform.lock.hcl
*.tfplan
aum-compliance-report.json
```

---

## 🔬 Step 3 - Create the Module Files

Each module folder contains one `main.tf` file. The folder names must match exactly what is referenced in the root `main.tf` source paths.

---

### 📄 modules/networking/main.tf

> Creates the VNet, subnet, NSG, and the explicit subnet-NSG association. The NSG only permits inbound RDP from your specific IP. The association is a separate resource - if it is missing, the NSG exists but applies to nothing.

```hcl
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "allowed_rdp_ip"      { type = string }

resource "azurerm_virtual_network" "main" {
  name                = "vnet-aumlab"
  address_space       = ["10.0.0.0/16"]
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_subnet" "main" {
  name                 = "snet-aumlab"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_security_group" "main" {
  name                = "nsg-aumlab"
  location            = var.location
  resource_group_name = var.resource_group_name
  security_rule {
    name                       = "AllowRDP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = var.allowed_rdp_ip
    destination_address_prefix = "*"
  }
}

# Without this association, the NSG exists but does not protect anything
resource "azurerm_subnet_network_security_group_association" "main" {
  subnet_id                 = azurerm_subnet.main.id
  network_security_group_id = azurerm_network_security_group.main.id
}

output "subnet_id" { value = azurerm_subnet.main.id }
```

---

### 📄 modules/keyvault/main.tf

> `enable_rbac_authorization = true` switches the vault from the legacy access policy model to the RBAC model. Without it, the role assignment granting the deploying identity permission to write secrets is silently ignored, and the secret write fails with 403.

```hcl
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "admin_password"      { type = string  sensitive = true }

data "azurerm_client_config" "current" {}
resource "random_string" "kv_suffix" { length=8 special=false upper=false }

resource "azurerm_key_vault" "main" {
  name                       = "kv-aum-${random_string.kv_suffix.result}"
  location                   = var.location
  resource_group_name        = var.resource_group_name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  enable_rbac_authorization  = true   # REQUIRED - without this, 403 on all secret operations
  soft_delete_retention_days = 7
  purge_protection_enabled   = false
}

# Grant the identity running terraform apply write access to secrets
resource "azurerm_role_assignment" "kv_deployer" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id         = data.azurerm_client_config.current.object_id
}

# depends_on ensures the role assignment propagates before Terraform writes the secret
resource "azurerm_key_vault_secret" "admin_password" {
  name         = "vm-admin-password"
  value        = var.admin_password
  key_vault_id = azurerm_key_vault.main.id
  depends_on   = [azurerm_role_assignment.kv_deployer]
}

output "key_vault_name" { value = azurerm_key_vault.main.name }
output "key_vault_id"   { value = azurerm_key_vault.main.id }
```

---

### 📄 modules/compute/main.tf

> DC01 gets a `CustomScriptExtension` that installs AD DS and promotes to Domain Controller. WS01 and WS02 get join extensions that set DNS to `10.0.1.4` first, wait for `aumlab.local` to resolve, then run `Add-Computer`. The `depends_on` on both join extensions forces them to wait for DC01 promotion to complete - without this, Terraform runs all extensions in parallel and domain join fails because the domain does not exist yet.

```hcl
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "subnet_id"           { type = string }
variable "admin_username"      { type = string }
variable "admin_password"      { type = string  sensitive = true }
variable "domain_name"         { type = string }
variable "domain_netbios"      { type = string }

# Public IPs - Static so addresses are reserved immediately and do not change
resource "azurerm_public_ip" "dc01" { name="pip-dc01" location=var.location resource_group_name=var.resource_group_name allocation_method="Static" sku="Standard" }
resource "azurerm_public_ip" "ws01" { name="pip-ws01" location=var.location resource_group_name=var.resource_group_name allocation_method="Static" sku="Standard" }
resource "azurerm_public_ip" "ws02" { name="pip-ws02" location=var.location resource_group_name=var.resource_group_name allocation_method="Static" sku="Standard" }

# DC01 NIC - STATIC IP 10.0.1.4 so WS01/WS02 DNS pointing here never breaks
resource "azurerm_network_interface" "dc01" {
  name="nic-dc01" location=var.location resource_group_name=var.resource_group_name
  ip_configuration {
    name="internal" subnet_id=var.subnet_id
    private_ip_address_allocation="Static" private_ip_address="10.0.1.4"
    public_ip_address_id=azurerm_public_ip.dc01.id
  }
}
resource "azurerm_network_interface" "ws01" {
  name="nic-ws01" location=var.location resource_group_name=var.resource_group_name
  ip_configuration { name="internal" subnet_id=var.subnet_id private_ip_address_allocation="Dynamic" public_ip_address_id=azurerm_public_ip.ws01.id }
}
resource "azurerm_network_interface" "ws02" {
  name="nic-ws02" location=var.location resource_group_name=var.resource_group_name
  ip_configuration { name="internal" subnet_id=var.subnet_id private_ip_address_allocation="Dynamic" public_ip_address_id=azurerm_public_ip.ws02.id }
}

locals {
  img = { pub="MicrosoftWindowsServer" offer="WindowsServer" sku="2022-Datacenter" ver="latest" }
}

resource "azurerm_windows_virtual_machine" "dc01" {
  name="DC01" location=var.location resource_group_name=var.resource_group_name
  size="Standard_B2s" admin_username=var.admin_username admin_password=var.admin_password
  network_interface_ids=[azurerm_network_interface.dc01.id]
  os_disk { caching="ReadWrite" storage_account_type="Standard_LRS" }
  source_image_reference { publisher=local.img.pub offer=local.img.offer sku=local.img.sku version=local.img.ver }
}
resource "azurerm_windows_virtual_machine" "ws01" {
  name="WS01" location=var.location resource_group_name=var.resource_group_name
  size="Standard_B2s" admin_username=var.admin_username admin_password=var.admin_password
  network_interface_ids=[azurerm_network_interface.ws01.id]
  os_disk { caching="ReadWrite" storage_account_type="Standard_LRS" }
  source_image_reference { publisher=local.img.pub offer=local.img.offer sku=local.img.sku version=local.img.ver }
}
resource "azurerm_windows_virtual_machine" "ws02" {
  name="WS02" location=var.location resource_group_name=var.resource_group_name
  size="Standard_B2s" admin_username=var.admin_username admin_password=var.admin_password
  network_interface_ids=[azurerm_network_interface.ws02.id]
  os_disk { caching="ReadWrite" storage_account_type="Standard_LRS" }
  source_image_reference { publisher=local.img.pub offer=local.img.offer sku=local.img.sku version=local.img.ver }
}

# DC01: promote to Domain Controller via CustomScriptExtension
# LAB ONLY: DSRM password is hardcoded. Never do this in production.
resource "azurerm_virtual_machine_extension" "setup_dc" {
  name                 = "SetupDC"
  virtual_machine_id   = azurerm_windows_virtual_machine.dc01.id
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.10"
  settings = jsonencode({ commandToExecute = join(" ", [
    "powershell -ExecutionPolicy Unrestricted -Command",
    "\"Install-WindowsFeature AD-Domain-Services -IncludeManagementTools;",
    "Import-Module ADDSDeployment;",
    "Install-ADDSForest -DomainName '${var.domain_name}'",
    "-DomainNetBiosName '${var.domain_netbios}'",
    "-SafeModeAdministratorPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force)",
    "-InstallDns -Force\""
  ])})
}

# WS01 and WS02 join extensions
# depends_on = [setup_dc] means these CANNOT start until DC01 promotion succeeds
# Without this, Terraform runs all extensions in parallel and domain join fails
locals {
  join_cmd = join(" ", [
    "powershell -ExecutionPolicy Unrestricted -Command",
    "\"$a=Get-NetAdapter|?{$_.Status -eq 'Up'}|Select -First 1;",
    "Set-DnsClientServerAddress -InterfaceIndex $a.InterfaceIndex -ServerAddresses '10.0.1.4';",
    "do{Start-Sleep 15}until([bool](Resolve-DnsName '${var.domain_name}' -ErrorAction SilentlyContinue));",
    "Add-Computer -DomainName '${var.domain_name}'",
    "-Credential (New-Object PSCredential('${var.domain_netbios}\\${var.admin_username}',",
    "(ConvertTo-SecureString '${var.admin_password}' -AsPlainText -Force)))",
    "-Restart -Force\""
  ])
}
resource "azurerm_virtual_machine_extension" "join_ws01" {
  name="JoinDomain" virtual_machine_id=azurerm_windows_virtual_machine.ws01.id
  publisher="Microsoft.Compute" type="CustomScriptExtension" type_handler_version="1.10"
  settings   = jsonencode({ commandToExecute = local.join_cmd })
  depends_on = [azurerm_virtual_machine_extension.setup_dc]
}
resource "azurerm_virtual_machine_extension" "join_ws02" {
  name="JoinDomain" virtual_machine_id=azurerm_windows_virtual_machine.ws02.id
  publisher="Microsoft.Compute" type="CustomScriptExtension" type_handler_version="1.10"
  settings   = jsonencode({ commandToExecute = local.join_cmd })
  depends_on = [azurerm_virtual_machine_extension.setup_dc]
}

output "dc01_id"        { value = azurerm_windows_virtual_machine.dc01.id }
output "ws01_id"        { value = azurerm_windows_virtual_machine.ws01.id }
output "ws02_id"        { value = azurerm_windows_virtual_machine.ws02.id }
output "dc01_public_ip" { value = azurerm_public_ip.dc01.ip_address }
output "ws01_public_ip" { value = azurerm_public_ip.ws01.ip_address }
output "ws02_public_ip" { value = azurerm_public_ip.ws02.ip_address }
```

---

### 📄 modules/update-manager/main.tf

> Three distinct operations: Policy handles enrollment (which VMs get assessed), Maintenance Configuration handles the schedule (when and what to patch), and Maintenance Assignments link the schedule to each VM (who gets patched). Update `start_date_time` to a future date before deploying.

```hcl
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "resource_group_id"   { type = string }
variable "dc01_id"             { type = string }
variable "ws01_id"             { type = string }
variable "ws02_id"             { type = string }

# Azure Policy: auto-enroll ALL VMs in this resource group into periodic assessment.
# Policy 59efceea = built-in "Configure periodic checking for missing system updates on Azure VMs"
# This runs assessment only - it does not apply patches.
# Any new VM added to rg-aumlab is automatically enrolled without any additional action.
resource "azurerm_resource_group_policy_assignment" "aum_assessment" {
  name                 = "aum-periodic-assessment"
  resource_group_id    = var.resource_group_id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/59efceea-0c96-497e-a4a1-4eb2290dac15"
}

# Maintenance Configuration: WHEN to patch and WHAT to patch.
# scope = InGuestPatch: patches run inside the guest OS, not at the hypervisor.
# in_guest_user_patch_mode = User: this Terraform config owns the schedule.
# classifications: Critical, Security, UpdateRollup - not all available patches.
# reboot = IfRequired: only reboots if a patch requires it.
# IMPORTANT: Update start_date_time to a future date before running terraform apply.
resource "azurerm_maintenance_configuration" "weekly" {
  name                     = "aum-weekly-patches"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  scope                    = "InGuestPatch"
  in_guest_user_patch_mode = "User"
  window {
    start_date_time = "2026-08-01 02:00"   # Update to a future date before deploying
    time_zone       = "Eastern Standard Time"
    duration        = "03:00"
    recur_every     = "Week"
  }
  install_patches {
    windows { classifications_to_include = ["Critical","Security","UpdateRollup"] }
    reboot = "IfRequired"
  }
}

# Maintenance Assignments: link the weekly schedule to each VM.
# Without these, VMs are assessed (via policy) but never automatically patched.
# Assessment and patching are separate operations - both are required.
resource "azurerm_maintenance_assignment_virtual_machine" "dc01" {
  location                     = var.location
  maintenance_configuration_id = azurerm_maintenance_configuration.weekly.id
  virtual_machine_id           = var.dc01_id
}
resource "azurerm_maintenance_assignment_virtual_machine" "ws01" {
  location                     = var.location
  maintenance_configuration_id = azurerm_maintenance_configuration.weekly.id
  virtual_machine_id           = var.ws01_id
}
resource "azurerm_maintenance_assignment_virtual_machine" "ws02" {
  location                     = var.location
  maintenance_configuration_id = azurerm_maintenance_configuration.weekly.id
  virtual_machine_id           = var.ws02_id
}
```

---

## 🔬 Step 4 - Create the Validation Script

### 📄 scripts/validate-lab.ps1

> Authenticates to Azure, queries Update Manager for patch assessment results on each VM, prints PASS/FAIL per machine, and exports `aum-compliance-report.json`. Run this after triggering assessments in Step 6.

> **Why a separate validation step instead of just checking the portal?** The JSON export is machine-readable. In a real environment you would parse this file to feed a compliance dashboard, open a ticket in ServiceNow for non-compliant machines, or archive it in Blob Storage for audit history.

```powershell
param([string]$ResourceGroup="rg-aumlab", [string]$SubscriptionId)

Connect-AzAccount -SubscriptionId $SubscriptionId

$vms     = @("DC01","WS01","WS02")
$results = @()
$allPass = $true

Write-Host "`n=== Azure Update Manager Compliance Validation ===" -ForegroundColor Cyan

foreach ($vm in $vms) {
    $assessment = Get-AzVMPatchAssessmentResult `
        -ResourceGroupName $ResourceGroup `
        -VMName $vm `
        -ErrorAction SilentlyContinue

    # PASS = assessment ran successfully AND no Critical/Security patches are missing.
    # A brand-new VM will often FAIL this check - it has patches outstanding.
    # This is the correct and expected result: the lab is working as designed.
    # Apply patches via the maintenance window to resolve it.
    $compliant = $assessment.Status -eq "Succeeded" -and $assessment.CriticalAndSecurityPatchCount -eq 0
    $status    = if ($compliant) { "PASS" } else { "FAIL" }
    if (-not $compliant) { $allPass = $false }

    Write-Host "[$status] $vm - Critical missing: $($assessment.CriticalAndSecurityPatchCount) | Status: $($assessment.Status)"

    $results += [PSCustomObject]@{
        VMName                   = $vm
        AssessmentStatus         = $assessment.Status
        CriticalAndSecurityCount = $assessment.CriticalAndSecurityPatchCount
        OtherPatchCount          = $assessment.OtherPatchCount
        LastAssessmentTime       = $assessment.StartDateTime
        Compliant                = $compliant
        Result                   = $status
    }
}

Write-Host ""
Write-Host "Overall: $(if ($allPass){"ALL PASS"}else{"FAILURES DETECTED"})" `
    -ForegroundColor $(if ($allPass){"Green"}else{"Red"})

# Export JSON - feeds SIEM, ServiceNow, or compliance dashboard in production
$report = @{
    GeneratedAt   = (Get-Date -Format "o")
    ResourceGroup = $ResourceGroup
    VMs           = $results
}
$report | ConvertTo-Json -Depth 5 | Out-File "./aum-compliance-report.json" -Encoding UTF8
Write-Host "Report exported: aum-compliance-report.json" -ForegroundColor Cyan
```

---

## 🔬 Step 5 - Configure Variables and Deploy

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars

# Edit terraform.tfvars:
#   allowed_rdp_ip = your current public IP from whatismyip.com in format 1.2.3.4/32

# Edit modules/update-manager/main.tf:
#   Update start_date_time to any future date - if this date is in the past, terraform plan fails

# Edit backend.tf:
#   Replace REPLACE_WITH_YOUR_STORAGE_ACCOUNT_NAME with your actual storage account name

# Set admin password as environment variable - never put it in a file
$env:TF_VAR_admin_password = "YourStrongPassword123!"
echo $env:TF_VAR_admin_password   # If nothing prints, set it again before apply

az login && az account show
terraform init
terraform plan -out=aum-lab.tfplan
# Review - expect: 3 VMs, 1 VNet+NSG+subnet+association, 1 Key Vault,
# 1 Maintenance Configuration, 3 Maintenance Assignments, 1 Policy Assignment

terraform apply aum-lab.tfplan
# Takes 15-20 minutes - DC01 domain promotion is the longest step
```

---

## 🔬 Step 6 - Trigger Assessment and Validate

After deployment, trigger an on-demand assessment on each VM immediately. The Azure Policy will run assessments on its own schedule, but triggering manually surfaces compliance data right now without waiting.

```powershell
$subId = az account show --query id -o tsv

# Trigger on-demand assessment via the Azure REST API.
# This is the same operation the portal uses when you click "Assess Now".
# Each assessment takes 5-10 minutes per VM to complete.
foreach ($vm in @("DC01","WS01","WS02")) {
    az rest --method POST `
        --url "https://management.azure.com/subscriptions/$subId/resourceGroups/rg-aumlab/providers/Microsoft.Compute/virtualMachines/$vm/assessPatches?api-version=2022-03-01"
    Write-Host "Assessment triggered: $vm"
}

# Wait 5-10 minutes, then validate
pwsh ./scripts/validate-lab.ps1 -ResourceGroup "rg-aumlab" -SubscriptionId $subId
```

> 💡 **If validate-lab.ps1 shows FAIL with Critical missing > 0:** This is expected on a brand-new VM - the machine has patches outstanding. It proves the assessment is working correctly. The lab is functioning as designed. Apply patches via the maintenance window to resolve it, then re-run validation to confirm.

---

## ✅ Verification Checklist

| Check | How to Verify |
|---|---|
| **All VMs assessed** | Azure Update Manager - Machines - all 3 VMs show as Assessed |
| **Maintenance config linked** | Maintenance Configurations - aum-weekly-patches - all 3 VMs are listed |
| **Policy applied** | Policy - Assignments - periodic assessment policy shows Applied on rg-aumlab |
| **Compliance script runs** | validate-lab.ps1 executes without errors and prints PASS/FAIL per VM |
| **JSON report exported** | `aum-compliance-report.json` exists in the project root with data for all 3 VMs |

---

## 🔬 Step 7 - Teardown

> ⚠️ Always destroy when finished. ~$0.17/hr = ~$4/day = ~$28/week if left running.

```powershell
terraform destroy -auto-approve

# Verify everything is removed
az group show --name rg-aumlab 2>&1
# Expected: ResourceGroupNotFound
```

---

## 🔧 Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Provider registration error during apply | `Microsoft.Maintenance` or `Microsoft.GuestConfiguration` not Registered | Run `az provider show` for each namespace. Wait until both show Registered, then re-apply. |
| 403 on Key Vault secret during apply | `enable_rbac_authorization` missing or RBAC propagation delay | Confirm `enable_rbac_authorization=true` is in the keyvault module. Re-run `terraform apply` - it resumes from the failed resource. |
| DC01 extension fails or times out | AD promotion failure or Azure extension timeout | Re-run `terraform apply` - Terraform skips succeeded resources and only retries what failed. Check Azure portal - DC01 - Extensions for detailed error output. |
| WS01/WS02 domain join fails - DNS not resolving | DC01 took longer than 3 minutes to finish promotion | Re-run `terraform apply`. The join extension retries. The join script waits up to 3 minutes for `aumlab.local` DNS to resolve before attempting `Add-Computer`. |
| validate-lab.ps1 shows `Status: null` | Assessment triggered but not yet complete | Wait 5-10 minutes and re-run the script. |
| validate-lab.ps1 shows `Critical missing > 0` | Expected on a fresh VM - patches are outstanding | This is correct behaviour. Trigger the maintenance window to apply patches, then re-validate. |
| `start_date_time` error during plan | The date in `update-manager/main.tf` is in the past | Update `start_date_time` to any future date and re-run `terraform plan`. |
| terraform apply changes nothing after a failed run | Resources were partially created | Re-run `terraform apply` - Terraform reads current state and only creates what is missing. |

---

## 🔒 Security Notes

**Why patch management is a security control, not just an operations task:**

- **CVE exposure window** - The time between a vulnerability being disclosed and a patch being applied is the window during which attackers actively exploit it. Azure Update Manager closes that window on a defined, auditable schedule.
- **Compliance requirements** - PCI DSS requires critical patches to be applied within one month. SOC 2 and ISO 27001 require documented vulnerability management processes. The JSON report export from `validate-lab.ps1` is the foundation for those audit artefacts.
- **Policy-based enforcement** - Azure Policy ensures that compliance is not dependent on an operator remembering to enrol each VM. Any VM added to `rg-aumlab` is automatically enrolled. This is the difference between a security control and a security hope.
- **Separation of assessment and patching** - Understanding that these are separate operations prevents two common failures: teams that assess but never patch (compliance theatre), and teams that patch without assessing first (uncontrolled change management).

---

## 💡 Key Concepts Reference

| Concept | Definition |
|---|---|
| **Azure Update Manager** | The cloud-native patch management service for Azure VMs. Agentless, integrates with Azure Policy, replaces WSUS for cloud workloads. |
| **Maintenance Configuration** | The Terraform resource that defines WHEN to patch (schedule), WHAT to patch (classifications), and HOW to handle reboots. |
| **Maintenance Assignment** | Links a specific VM to a Maintenance Configuration. Required for automated patching - assessment and patching are separate. |
| **Azure Policy** | A governance service that evaluates resources against defined rules and can automatically remediate non-compliant resources. |
| **Policy Definition 59efceea** | The built-in Azure policy that configures periodic patch assessment checking on Azure VMs. |
| **InGuestPatch** | The maintenance scope that applies patches inside the guest OS (as opposed to hypervisor-level maintenance). |
| **in_guest_user_patch_mode = User** | Tells Azure that this Terraform configuration - not Azure's automatic settings - controls the patch schedule. |
| **On-demand assessment** | A manually triggered assessment that immediately surfaces current patch compliance state without waiting for the scheduled cadence. |
| **CVSS** | Common Vulnerability Scoring System - the 0-10 scale used to rate vulnerability severity. Critical patches have CVSS scores >= 9.0. |
| **Compliance Theatre** | When an organisation runs assessments and generates reports but never applies the patches - creating the appearance of compliance without the substance. |
| **depends_on** | A Terraform meta-argument that creates explicit resource ordering in the execution graph. Used here to ensure DC01 promotion completes before domain join attempts. |
| **Remote State** | Terraform state stored in Azure Blob Storage. Shared across the lab series using different state keys per lab. |

---

## 🔗 References

- [Azure Update Manager Documentation](https://learn.microsoft.com/en-us/azure/update-manager/overview)
- [Azure Update Manager - Maintenance Configurations](https://learn.microsoft.com/en-us/azure/update-manager/manage-maintenance-configurations)
- [Azure Policy Built-in Definitions - Update Manager](https://learn.microsoft.com/en-us/azure/update-manager/policy-reference)
- [Terraform azurerm_maintenance_configuration](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/maintenance_configuration)
- [Terraform Module Structure - Best Practices](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- [Azure Key Vault RBAC Guide](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)
- [PCI DSS Patch Management Requirements](https://www.pcisecuritystandards.org/documents/PCI_DSS_v3-2-1.pdf)

---

*Lab authored by **Jair Smith** · Cloud Operations Series · Azure Update Manager Lab*
