# 📦 Azure Backup & Azure Site Recovery (ASR)

# Enterprise-Grade Backup & Disaster Recovery – Architected & Automated

![Image](https://learn.microsoft.com/en-us/azure/backup/media/guidance-best-practices/azure-backup-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-architecture/architecture-on-premises-mars.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-azure-vms-introduction/vmbackup-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-overview/azure-backup-overview.png)

---

## 1️⃣ Core Concepts (Clear Separation)

### Backup vs Disaster Recovery

| Capability    | Azure Backup            | Azure Site Recovery (ASR) |
| ------------- | ----------------------- | ------------------------- |
| Primary goal  | **Data protection**     | **Business continuity**   |
| Recovery type | Point-in-time restore   | Near-real-time failover   |
| Scope         | Files, disks, VMs, DBs  | Entire VM / application   |
| Typical RPO   | Hours / Days            | Minutes                   |
| Typical RTO   | Minutes–Hours           | Minutes                   |
| Azure service | Recovery Services Vault | Recovery Services Vault   |

> **Architect rule:**
> Backup protects **data integrity**.
> ASR protects **service availability**.
> **Production workloads require both.**

---

## 2️⃣ Azure Backup – Architecture & Flow

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-azure-vms-introduction/vmbackup-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-azure-vms/instant-rp-flow.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-architecture/architecture-on-premises-mars.png)

### How Azure VM Backup Works

1. **Recovery Services Vault (RSV)** is created in the same region as the VM
2. **Backup policy** defines:

   * Schedule (daily / enhanced multiple times)
   * Retention (daily, weekly, monthly, yearly)
3. Azure takes a **snapshot of managed disks**
4. Only **changed blocks (delta)** are transferred to the vault
5. **Recovery points** are securely stored (encrypted, immutable optional)

### What Gets Backed Up

* OS Disk
* Data Disks
* Application-consistent state (VSS on Windows, scripts on Linux)

---

## 3️⃣ Azure Backup – Enterprise Design Decisions

### Vault Configuration (Non-Negotiable in Production)

| Setting            | Recommendation                             |
| ------------------ | ------------------------------------------ |
| Storage redundancy | **GRS** (default for prod)                 |
| Soft delete        | **Enabled**                                |
| Immutability       | **Locked (if compliance/ransomware risk)** |
| RBAC               | Backup Operator ≠ Subscription Owner       |
| Monitoring         | Azure Monitor + alerts                     |

### Policy Design Example

| Workload          | Frequency        | Retention          |
| ----------------- | ---------------- | ------------------ |
| Tier-1 (DB, ERP)  | Daily + Enhanced | 7D / 4W / 12M / 5Y |
| Tier-2 (App)      | Daily            | 30D                |
| Tier-3 (Dev/Test) | Daily            | 7D                 |

---

## 4️⃣ Azure Backup – Automation Patterns

### Terraform (Production-Ready Pattern)

```hcl
resource "azurerm_recovery_services_vault" "vault" {
  name                = var.vault_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
}

resource "azurerm_backup_policy_vm" "policy" {
  name                = "daily-retention-policy"
  resource_group_name = azurerm_resource_group.rg.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name

  time = "23:00"

  retention_daily {
    count = 30
  }
}

resource "azurerm_backup_protected_vm" "vm" {
  resource_group_name = azurerm_resource_group.rg.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
  source_vm_id        = var.vm_id
  backup_policy_id    = azurerm_backup_policy_vm.policy.id
}
```

### CI/CD Integration

* Terraform applies vault & policy
* Post-deploy script:

  * Validate first backup success
  * Alert on failure
* GitHub Actions / Azure DevOps:

  * `terraform plan` gate
  * Policy drift detection

---

## 5️⃣ Restore Scenarios (What Architects Must Document)

### Supported Restore Options

| Restore Type         | Use Case                     |
| -------------------- | ---------------------------- |
| Create new VM        | Full recovery                |
| Restore disks        | Forensics / partial recovery |
| File-level restore   | Accidental deletion          |
| Cross-region restore | Regional outage              |

> **Best practice:**
> Every backup design must include **tested restore steps** in runbooks.

---

## 6️⃣ Azure Site Recovery (ASR) – DR Architecture

![Image](https://docs.microsoft.com/en-us/azure/architecture/solution-ideas/media/disaster-recovery-smb-azure-site-recovery.png)

![Image](https://learn.microsoft.com/en-us/azure/site-recovery/media/azure-to-azure-how-to-enable-replication-private-endpoints/architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/site-recovery/media/concepts-azure-to-azure-architecture/failover-v2.png)

### What ASR Solves

* Azure region outage
* Large-scale infrastructure failure
* Regulatory RTO/RPO commitments
* Recovery orchestration for complete VM-based applications

### Azure-to-Azure (A2A) Flow

1. Source VM disks replicate to the target region
2. Continuous replication keeps recovery points current
3. Recovery plans orchestrate boot order, network mapping, and app dependencies
4. Operational sequence is **Test Failover -> Failover -> Commit -> Re-protect -> Failback**

---

## 7️⃣ ASR Core Components

| Component                  | Purpose                             |
| -------------------------- | ----------------------------------- |
| Recovery Services Vault    | Central DR control plane            |
| Fabric                     | Region abstraction used by ASR      |
| Protection Container       | Logical grouping of protected items |
| Replication Policy         | Recovery point and snapshot rules   |
| Replication-Protected Item | A protected VM                      |
| Recovery Plan              | Ordered failover workflow           |

---

## 7️⃣.1️⃣ ASR Operational Facts

### Supported Workloads

* Azure Virtual Machines
* VMware virtual machines
* Hyper-V virtual machines
* Physical Windows and Linux servers

### Recovery Behavior

* Continuous replication is used for DR readiness
* Crash-consistent and application-consistent recovery points are supported
* Recovery point retention can be configured based on workload needs

### Failover Modes

* **Test Failover** for DR validation without production impact
* **Planned Failover** when shutdown and controlled cutover are possible
* **Unplanned Failover** during an actual outage scenario

### Pricing Reminder

* Billing is based primarily on protected instances
* The first 31 days are free per protected instance for many ASR scenarios, which is useful for pilots and initial testing

---

## 8️⃣ Azure Site Recovery – Step-by-Step Runbook

### Prerequisites

* Azure CLI installed and logged in: `az login`
* `site-recovery` extension installed if your CLI does not already include the commands

```bash
az extension add --name site-recovery
```

* Source VM running in the primary region
* Target region chosen and validated for ASR support
* Target networking planned before replication is enabled
* Recovery Services Vault available for DR operations

### Recommended Variables

```bash
LOCATION_SRC="eastus"
LOCATION_DR="westus2"
RG_PRIMARY="rg-primary"
RG_DR="rg-dr"
VAULT_RG="$RG_DR"
VAULT_NAME="asr-vault"
VM_NAME="vm-primary"
VNET_DR="vnet-dr"
SUBNET_DR="subnet-dr"
ASR_POLICY="A2A-5min-24hr"
MAPPING_NAME="map-primary-to-dr"
RECOVERY_PLAN="rp-vm-primary"
FABRIC_NAME="Azure"
```

### Step 1: Create Resource Groups

```bash
az group create --name "$RG_PRIMARY" --location "$LOCATION_SRC"
az group create --name "$RG_DR" --location "$LOCATION_DR"
```

### Step 2: Validate VM Size Before Protection

```bash
az vm list-sizes --location "$LOCATION_SRC" -o table
```

### Step 3: Create the Source VM

```bash
az vm create \
  --resource-group "$RG_PRIMARY" \
  --name "$VM_NAME" \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --location "$LOCATION_SRC"
```

> **Compatibility note:**
> If the deployment fails with ASR compatibility issues related to the NVMe controller, select a supported size and retry. One example from lab testing was switching from an unsupported configuration to `Standard_L2aos_v4`.

```bash
az vm list -d -o table
```

### Step 4: Create the Recovery Services Vault

```bash
az resource create \
  --resource-group "$VAULT_RG" \
  --name "$VAULT_NAME" \
  --resource-type Microsoft.RecoveryServices/vaults \
  --is-full-object \
  --properties "{\
    \"location\": \"$LOCATION_DR\",\
    \"sku\": {\
      \"name\": \"Standard\"\
    },\
    \"properties\": {\
      \"publicNetworkAccess\": \"Enabled\"\
    }\
  }"
```

### Step 5: Set Vault Context

```bash
az recoveryservices vault set-context \
  --name "$VAULT_NAME" \
  --resource-group "$VAULT_RG"
```

### Step 6: Prepare the DR Network

```bash
az network vnet create \
  --resource-group "$RG_DR" \
  --name "$VNET_DR" \
  --location "$LOCATION_DR" \
  --address-prefixes 10.40.0.0/16 \
  --subnet-name "$SUBNET_DR" \
  --subnet-prefix 10.40.1.0/24
```

### Step 7: Create the ASR Policy

```bash
az site-recovery policy create \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$ASR_POLICY" \
  --provider-type A2A \
  --recovery-point-retention-in-hours 24 \
  --application-consistent-snapshot-frequency-in-hours 1
```

### Step 8: Discover Protection Containers

```bash
SRC_CONTAINER=$(az site-recovery protection-container list \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --query "[?contains(name, 'default')].name" -o tsv)

TGT_CONTAINER=$(az site-recovery protection-container list \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --query "[?contains(name, 'default')].name" -o tsv)
```

### Step 9: Map the Source and Target Containers

```bash
az site-recovery protection-container-mapping create \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$MAPPING_NAME" \
  --policy-name "$ASR_POLICY" \
  --source-protection-container-name "$SRC_CONTAINER" \
  --target-protection-container-name "$TGT_CONTAINER"
```

### Step 10: Discover the Protectable VM

```bash
PROTECTABLE_ID=$(az site-recovery protectable-item list \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --fabric-name "$FABRIC_NAME" \
  --protection-container-name "$SRC_CONTAINER" \
  --query "[?properties.friendlyName=='$VM_NAME'].name" -o tsv)
```

### Step 11: Enable Replication

```bash
RPI_NAME="$VM_NAME-rpi"

az site-recovery replication-protected-item create \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --fabric-name "$FABRIC_NAME" \
  --protection-container-name "$SRC_CONTAINER" \
  --name "$RPI_NAME" \
  --protectable-item-name "$PROTECTABLE_ID" \
  --policy-name "$ASR_POLICY" \
  --protectable-item-type VirtualMachine \
  --auto-protect true \
  --recovery-azure-resource-group-id "$(az group show -n "$RG_DR" --query id -o tsv)" \
  --recovery-azure-network-id "$(az network vnet show -g "$RG_DR" -n "$VNET_DR" --query id -o tsv)" \
  --recovery-azure-subnet-name "$SUBNET_DR"
```

> **Operational note:**
> A2A replication can also be enabled from the portal. For teams new to ASR, the portal is often simpler for the first rollout because it exposes network, cache storage, and target settings more clearly.

### Step 12: Monitor Replication Health

```bash
az recoveryservices vault replication-protected-item list \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --output table
```

```bash
az recoveryservices vault replication-protected-item list \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --query "[].{Name:name,State:provisioningState}" \
  --output table
```

Wait until the protection state is healthy and the initial replication completes before testing failover.

---

## 9️⃣ Test Failover, Failover, Re-Protect, and Failback

### Step 13: Create a Recovery Plan

```bash
az site-recovery recovery-plan create \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$RECOVERY_PLAN" \
  --primary-fabric-id "$FABRIC_NAME" \
  --recovery-fabric-id "$FABRIC_NAME" \
  --group-type Boot \
  --replication-protected-items "$RPI_NAME"
```

### Step 14: Perform a Test Failover

```bash
az site-recovery recovery-plan test-failover \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$RECOVERY_PLAN" \
  --failover-direction PrimaryToRecovery
```

This creates a recovery VM in the DR region for validation without affecting production traffic.

### Step 15: Clean Up the Test Failover

```bash
az site-recovery recovery-plan test-failover-cleanup \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$RECOVERY_PLAN" \
  --comments "Validated DR test"
```

### Step 16: Perform the Actual Failover

```bash
az site-recovery recovery-plan failover \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$RECOVERY_PLAN" \
  --failover-direction PrimaryToRecovery
```

### Step 17: Commit the Failover

```bash
az site-recovery recovery-plan failover-commit \
  --resource-group "$VAULT_RG" \
  --vault-name "$VAULT_NAME" \
  --name "$RECOVERY_PLAN"
```

Once committed, the DR-side VM becomes the active production copy.

### Step 18: Re-Protect and Prepare for Failback

Re-protect reverses the replication direction so the recovered workload can be replicated back toward the original region. This step is commonly performed from the portal because the workflow is easier to validate visually.

Portal sequence:

1. Open the Recovery Services Vault.
2. Go to **Replicated items**.
3. Select the failed-over VM.
4. Choose **Re-protect**.
5. Set the original region as the new recovery target.

### Step 19: Fail Back to the Original Region

After reverse replication is healthy, run failover again in the opposite direction or use the portal to return service to the original region.

---

## 🔟 PowerShell Operations

```powershell
Connect-AzAccount

$vault = Get-AzRecoveryServicesVault -Name "asr-vault"
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

Get-AzRecoveryServicesAsrReplicationProtectedItem
```

PowerShell is useful for operational scripting, reporting, and teams already standardizing on Az modules.

---

## 1️⃣1️⃣ ASR Best Practices

* Use a dedicated DR VNet and validate subnet mapping before failover
* Monitor replication health through Azure Monitor and alerting
* Test failover regularly and keep evidence for audit reviews
* Build Recovery Plans for multi-tier workloads so dependencies start in order
* Validate permissions for encrypted VMs and Key Vault access before enabling protection
* Tag DR resources with owner, application, and recovery tier metadata

---

## 1️⃣2️⃣ ASR CLI Scope vs Portal Scope

### What Works Well in CLI

* Create resource groups and target network
* Create policies and container mappings
* Enable and monitor replication
* Create recovery plans
* Run scripted failover and cleanup flows

### What Many Teams Still Prefer in Portal

* First-time A2A configuration
* Re-protect and failback validation
* Reviewing replication health visually
* Multi-VM orchestration checks before production DR drills

---

## 1️⃣3️⃣ Backup + DR Reference Architecture (Recommended)

![Image](https://docs.microsoft.com/en-us/azure/architecture/solution-ideas/media/disaster-recovery-smb-azure-site-recovery.png)

![Image](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/azure-virtual-desktop/images/azure-virtual-desktop-bcdr-pooled-host-pool.png)

![Image](https://learn.microsoft.com/en-us/azure/architecture/data-guide/images/dr-for-azure-data-platform-landing-zone-architecture.png)

### Layered Protection Model

```
Application
 ├─ Azure Site Recovery (Availability)
 │   ├─ Region failover
 │   ├─ Recovery Plans
 │
 └─ Azure Backup (Data Protection)
     ├─ Immutable backups
     ├─ Long-term retention
     ├─ Ransomware recovery
```

---

## 🔐 Security & Compliance Checklist

* ✅ Soft delete enabled
* ✅ Vault immutability locked
* ✅ RBAC separated (Operator ≠ Owner)
* ✅ Key Vault permissions validated (for encrypted VMs)
* ✅ Audit logs enabled
* ✅ Restore tested quarterly
* ✅ DR test failover recorded and reviewed

---

## 🧪 Operational Excellence (What Auditors Ask)

| Item                   | Evidence                |
| ---------------------- | ----------------------- |
| Last successful backup | Backup job logs         |
| Restore test           | Screenshot / ticket     |
| DR test                | Test failover report    |
| RPO achieved           | ASR replication health  |
| RTO achieved           | Recovery plan execution |
| Failback readiness     | Re-protect status       |

---

## 📌 Architect Takeaways

* **Backup ≠ DR** and production workloads need both
* Automate setup and validation, but govern failover authority tightly
* Test restore and test failover on a scheduled basis
* Treat the Recovery Services Vault as a security boundary
* Document failover, commit, re-protect, and failback ownership in runbooks

---

## 1. Theory & Concepts

### 1.1 What is Azure Backup?

* Azure Backup provides “simple, secure, cost-effective” solutions to back up data and recover it from the Azure cloud. ([Microsoft Learn][1])
* It covers multiple workloads: on-premises files/folders (via MARS agent), Azure VMs (Windows/Linux), Azure managed disks, Azure Files, SQL/SAP HANA on Azure VMs, etc. ([Microsoft Learn][1])
* Backups are stored inside a Recovery Services vault (RS vault) which acts as a management entity for backup jobs, recovery points, policies, monitoring. ([Microsoft Learn][2])

### 1.2 Why use Azure Backup?

Key benefits:

* Offload on-premises backup infrastructure; hybrid support. ([Microsoft Learn][1])
* Back up Azure IaaS VMs with independent and isolated recovery points. ([Azure Documentation][3])
* Scale via Azure’s underlying infrastructure; minimal backup infrastructure to manage. ([Microsoft Learn][1])
* Frequently unlimited data ingress / egress (within Azure) for backup/restore operations. ([Microsoft Learn][1])
* Security: encryption at rest and in transit, RBAC, immutable vaults, ransomware protection. ([Azure Documentation][3])
* Flexible retention: support for short- and long-term retention, LRS/GRS/ZRS replication options. ([Microsoft Learn][1])

### 1.3 How does Azure Backup work (for VMs)?

* For a VM protected by Azure Backup:

  1. A backup policy defines schedule and retention. ([Azure Documentation][4])
  2. At a scheduled time (or on-demand), a snapshot is taken of the VM’s disks (and, if configured, application-consistent via VSS or custom scripts). ([Azure Documentation][3])
  3. Only changed blocks (delta) since last backup are transferred to the vault, optimizing storage/time. ([Azure Documentation][3])
  4. Recovery point(s) are stored in the vault. You can restore the VM (create new VM / restore disks) or files/folders depending on workload. ([Microsoft Learn][5])

### 1.4 Backup vs Disaster Recovery (DR)

* **Backup**: Data-level protection (files, VMs, disks, DBs). Good for accidental deletion, corruption, ransomware.
* **Disaster Recovery (DR)**: Focus is on application and service continuity (failover to another region or site). Good for outages, region failures.

  * In Azure this is handled by Azure Site Recovery (ASR) rather than just Backup.
* Use both: backup for data protection, DR for availability.

### 1.5 What is Azure Site Recovery (ASR)?

* ASR enables replication of VMs (on-premises → Azure, Azure-to-Azure) and orchestration of failover/failback. ([Microsoft Learn][1])
* Key features include test failover, multi-tier application ordering, cross-region failover.
* Use case: ensure business continuity when region or major outage happens (vs simple restore from backup).

---

## 2. Prerequisites & Planning Considerations

### 2.1 Prerequisites for Azure VM Backup

* Azure VM must support Azure Backup (agent/VM extension may need to be installed). ([Microsoft Learn][6])
* Recovery Services vault must be created in **same region** as the VM for best performance. ([Microsoft Learn][2])
* Storage redundancy (LRS/GRS/ZRS) decision for the vault: e.g., geo-redundant for higher durability. ([Microsoft Learn][7])
* For encrypted VMs (with BEK/KEK) you must grant Azure Backup service permissions to the Azure Key Vault. ([Microsoft Learn][7])
* Ensure backup policy aligns with your RPO/RTO, retention requirements, business SLAs.
* Consider cost: retention periods, storage redundancy, frequency.

### 2.2 Prerequisites for Disaster Recovery (ASR)

* Ensure source and target regions are defined and supported.
* Define replication policy (RPO, recovery point retention, failover plan).
* Ensure network, NSGs, subscription/tenant permissions are in place.
* Test failover to validate.

### 2.3 Planning Considerations / Best Practices

* Choose correct storage redundancy for vault: GRS for region failover, ZRS if zone-resilient.
* Use immutable vaults (if required) to prevent deletion of recovery points. ([Microsoft Learn][2])
* Use application-consistent backups for stateful workloads (e.g., SQL, SAP). For Linux, you may need custom pre/post scripts. ([Azure Documentation][3])
* Monitor backup jobs & alert on failures—automate retry workflows. ([Microsoft Learn][8])
* Ensure you document and test restore procedures (not just backup). Frequent drills matter.
* For DR, perform test failovers periodically to validate.
* Tag resources appropriately (vault, protected items) for visibility in your CI/CD pipeline.
* Align backup policy with GC (governance & compliance) – retention, WORM-capable storage if needed.

---

## 3. Code / Automation Examples

Below are sample code snippets you can embed into your infrastructure-as-code (IaC) and CI/CD. I’ve given CLI first, then ARM/Bicep, then Terraform stubs.

### 3.1 Azure CLI for VM Backup

```bash
# Sign in
az login

# Set subscription (optional)
az account set --subscription "MySubscriptionId"

# Create resource group (if not exists)
az group create --name MyBackupRG --location eastus

# Create Recovery Services vault
az backup vault create \
  --resource-group MyBackupRG \
  --name MyRecoveryVault \
  --location eastus

# Configure storage redundancy (optional)
az backup vault backup-properties set \
  --resource-group MyBackupRG \
  --name MyRecoveryVault \
  --backup-storage-redundancy GeoRedundant

# Enable backup for VM
az backup protection enable-for-vm \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --vm MyVMName \
  --policy-name DefaultPolicy

# Trigger an on-demand backup
az backup protection backup-now \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --item-name MyVMName \
  --backup-management-type AzureIaasVM \
  --workload-type VM
```

These commands derive from official documentation. ([Microsoft Learn][7])

### 3.2 CLI for Restore (VM Disks)

```bash
# List recovery points
az backup recoverypoint list \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --backup-management-type AzureIaasVM \
  --container-name IaasVMContainer;iaasvmcontainerv2;MyVMName \
  --item-name MyVMName \
  --query [0].name -o tsv

# Restore disks from recovery point
az backup restore restore-disks \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --container-name IaasVMContainer;iaasvmcontainerv2;MyVMName \
  --item-name MyVMName \
  --recovery-point-id <RecoveryPointID> \
  --storage-account MyStorageAccount
```

From Microsoft docs. ([Microsoft Learn][9])

### 3.3 ARM / Bicep Snippets

**ARM Template Sample: (JSON snippet)**

```json
{
  "type": "Microsoft.RecoveryServices/vaults",
  "apiVersion": "2024-01-01",
  "name": "[parameters('vaultName')]",
  "location": "[parameters('location')]",
  "properties": {
    // properties omitted for brevity
  }
}
```

**Bicep Sample:**

```bicep
param vaultName string = 'rg-backup-vault'
param location string = resourceGroup().location

resource recoveryVault 'Microsoft.RecoveryServices/vaults@2024-01-01' = {
  name: vaultName
  location: location
  properties: {}
}

// Now configure backup for a VM
resource backupPolicy 'Microsoft.RecoveryServices/vaults/backupPolicies@2024-01-01' = {
  parent: recoveryVault
  name: 'DefaultPolicy'
  properties: {
    schedulePolicy: {
      scheduleRunFrequency: 'Daily'
      scheduleRunTime: '23:00'
    }
    retentionPolicy: {
      dailySchedule: {
        retentionTimes: [ '23:00' ]
        retentionDuration: {
          count: 30
          durationType: 'Days'
        }
      }
    }
  }
}
```

(You’d expand to include enabling backup for VM via `Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems` etc.)

### 3.4 Terraform Module Stub

Here’s a conceptual Terraform module you can refine.

```hcl
variable "location"          { default = "eastus" }
variable "resource_group"    { default = "rg-backup" }
variable "vault_name"        { default = "recoveryVault" }
variable "vm_id"             { description = "ID of VM to protect" }

resource "azurerm_resource_group" "backup" {
  name     = var.resource_group
  location = var.location
}

resource "azurerm_recovery_services_vault" "vault" {
  name                = var.vault_name
  location            = azurerm_resource_group.backup.location
  resource_group_name = azurerm_resource_group.backup.name

  sku = "Standard"  # or other supported SKU
}

resource "azurerm_backup_policy_vm" "policy" {
  name                = "defaultPolicy"
  resource_group_name = azurerm_recovery_services_vault.vault.resource_group_name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name

  time = "23:00"
  retention_daily {
    count        = 30
  }
}

resource "azurerm_backup_protected_vm" "protected" {
  resource_group_name = azurerm_recovery_services_vault.vault.resource_group_name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
  source_vm_id        = var.vm_id
  backup_policy_id    = azurerm_backup_policy_vm.policy.id
}
```

Note: This is “stub” style — you’ll need to refine based on latest Terraform provider version and workload.

---

## 4. Disaster Recovery (Azure Site Recovery) – High-level Steps

Here’s a summary for DR using ASR:

1. Create Recovery Services vault (or reuse) in primary region.
2. Configure replication policy (RPO, retention).
3. For each VM, enable replication to target region/zone.
4. Perform **Test failover** to validate.
5. In outage case, perform **Failover**. After recovery, plan **Failback** to primary region.

CLI/ARM/Bicep code for ASR is more involved (fabric containers, replication-protected-items etc) — I can share a full module if you want.

---

## 5. Best Practices & Tips for Your Role

Since you are a cloud solutions architect, here are some **architectural and operational best practices** you should incorporate:

* **Policy design**: Define backup policies aligned with business RPO/RTO. e.g., hourly backups for critical VMs (use Enhanced policy) vs daily for less critical. ([Microsoft Learn][2])
* **Immutability & ransomware defense**: Use immutable vaults and WORM policies if required. Document “who can delete backups” etc.
* **Zone/region redundancy**: Use vault redundancy options (ZRS/GRS) depending on requirement for disaster resilience.
* **Automation & monitoring**: Integrate with Azure Automation or Logic Apps to auto-retry failed backup jobs, send alerts, trigger runbooks. ([Microsoft Learn][8])
* **Restore drills**: Regularly test restores (both data recoveries and full VM/disk/DR scenarios) to validate procedures.
* **Cost & governance**: Include tagging, retention lifecycle, review old recovery points, purge unused protected items.
* **IaC & repeatability**: Define your backups and DR via Terraform/ARM/Bicep, version control them, include into CI/CD pipelines.
* **Documentation and runbooks**: Produce operations runbooks showing how to recover VMs, how to failover region, how to attach recovered disks, etc.
* **Security**: Limit RBAC for vaults, ensure Key Vault and encryption keys are protected, enable soft delete for vaults.
* **Business continuity planning**: Backup is part of your BC/DR strategy; align with application owners about app-consistency, downtime windows, etc.

---

## 6. What’s New / Recent Features to Note

* Support for multiple daily backups (Enhanced policy) for VMs. ([Azure Documentation][3])
* Cross-Region Restore (CRR) and Cross-Subscription Restore capabilities. ([Microsoft Learn][5])
* Vault immutability and enhanced ransomware protection features. ([Microsoft Learn][2])
* Automation of backup job retry logic. ([Microsoft Learn][8])

---

If you like, I can **generate a full GitHub-ready repository** with:

* Terraform modules (vault, policy, protected VM)
* Bicep/ARM templates
* CLI scripts for on-demand backup, restore, cleanup
* README (English + Marathi + Hindi) with architecture diagrams
* CI/CD pipeline (GitHub Actions) that triggers backup policy checks, alerts on failures, etc.

Would you like me to build that repository for you now (or pick a specific workload: e.g., Azure Files or SQL in VM)?

[1]: https://learn.microsoft.com/en-us/azure/backup/?utm_source=chatgpt.com "Azure Backup service documentation - Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/backup/quick-backup-vm-portal?utm_source=chatgpt.com "Quickstart: Back up a virtual machine in Azure - Learn Microsoft"
[3]: https://docs.azure.cn/en-us/backup/backup-azure-vms-introduction?utm_source=chatgpt.com "About Azure VM backup"
[4]: https://docs.azure.cn/en-us/backup/quick-backup-vm-cli?utm_source=chatgpt.com "Quickstart - Back up a VM with Azure CLI - Azure Backup | Azure Docs"
[5]: https://learn.microsoft.com/en-us/azure/backup/backup-azure-arm-restore-vms?utm_source=chatgpt.com "Restore VMs by using the Azure portal using Azure Backup"
[6]: https://learn.microsoft.com/en-us/azure/backup/backup-azure-vms-first-look-arm?utm_source=chatgpt.com "Back up an Azure VM from the VM settings - Microsoft Learn"
[7]: https://learn.microsoft.com/en-us/azure/backup/quick-backup-vm-cli?utm_source=chatgpt.com "Back up a VM with Azure CLI - Learn Microsoft"
[8]: https://learn.microsoft.com/en-us/azure/backup/automation-backup?utm_source=chatgpt.com "Automation in Azure Backup"
[9]: https://learn.microsoft.com/en-us/azure/backup/tutorial-restore-disk?utm_source=chatgpt.com "Restore a VM with Azure CLI - Azure Backup | Microsoft Learn"

Below is an updated **code-and-automation reference sheet** for Azure Site Recovery (ASR) covering CLI, Terraform and key modules/resources. As your Cloud Solutions Architect role demands production-ready patterns, I’ve included real-world tips and caveats.

---

## ✅ CLI / Azure PowerShell Snippets

### 1. Install/Configure CLI extension

```bash
# Install the ASR extension for the Azure CLI
az extension add --name site-recovery
```

(This gives you the `az site-recovery` commands). ([N2W Software][1])

### 2. Create Recovery Services Vault (if not already)

```bash
az group create --name MyRG --location eastus

az recovery-services vault create \
  --resource-group MyRG \
  --name MyRSVault \
  --location eastus
```

Then set vault context for ASR:

```bash
az site-recovery vault set \
  --vault-name MyRSVault \
  --resource-group MyRG
```

(Use the vault in subsequent commands) ([N2W Software][1])

### 3. Enable replication for an Azure VM (Azure to Azure scenario)

```bash
az site-recovery protection-container mapping create \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --recovery-fabric RecoveryFabric \
  --recovery-protection-container RecoveryPC \
  --policy-name MyReplicationPolicy

az site-recovery protected-item create \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --protection-container-name PrimaryPC \
  --name MyVMName-RPI \
  --policy-id <policyId> \
  --provider-details '{ "azureToAzure": { "osType": "Windows", "vmId": "<sourceVmId>", "recoveryResourceGroupId": "<rgId>", "recoveryAzureNetworkId": "<vnetId>", "recoverySubnetName": "<subnetName>" } }'
```

This is based on the `az site-recovery protected-item create` command. ([Microsoft Learn][2])
(You’ll need to replace placeholders with your specific fabric, containers, VM IDs etc.)

### 4. Failover / Test Failover / Commit / Re-protect

```bash
# Test failover (without committing)
az site-recovery protected-item planned-failover \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --name MyVMName-RPI \
  --failover-direction PrimaryToRecovery

# Unplanned failover
az site-recovery protected-item unplanned-failover \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --name MyVMName-RPI

# Commit failover
az site-recovery protected-item failover-commit \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --name MyVMName-RPI

# Reprotect (after recovery)
az site-recovery protected-item reprotect \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name RecoveryFabric \
  --protection-container RecoveryPC \
  --name MyVMName-RPI
```

These commands are documented under the `az site-recovery protected-item` section. ([Microsoft Learn][3])

---

## 🔧 Terraform / IaC Snippets

Here are Terraform resources for ASR (Azure → Azure). Use these inside your modules/CI pipeline.

### Terraform resources you’ll likely use

* `azurerm_site_recovery_fabric` – defines the replication fabric. ([Terraform Registry][4])
* `azurerm_site_recovery_protection_container` – defines a protection container within a fabric. ([Shisho Cloud byGMO - 開発組織のための 脆弱性診断ツール][5])
* `azurerm_site_recovery_protection_container_mapping` – maps source & target containers with a policy. ([Shisho Cloud byGMO - 開発組織のための 脆弱性診断ツール][6])
* `azurerm_site_recovery_replication_policy` – defines replication cadence, retention etc. ([Shisho Cloud byGMO - 開発組織のための 脆弱性診断ツール][7])
* `azurerm_site_recovery_replicated_vm` – the actual VM replication configuration. ([Terraform Registry][8])

### Example Terraform stub

```hcl
provider "azurerm" {
  features {}
}

variable "location_primary"   { default = "eastus" }
variable "location_recovery"  { default = "westus2" }
variable "vm_id"              { description = "Source VM Resource ID" }

resource "azurerm_resource_group" "primary" {
  name     = "rg-asr-primary"
  location = var.location_primary
}

resource "azurerm_resource_group" "recovery" {
  name     = "rg-asr-recovery"
  location = var.location_recovery
}

resource "azurerm_recovery_services_vault" "vault" {
  name                = "rsvault-asr"
  location            = azurerm_resource_group.primary.location
  resource_group_name = azurerm_resource_group.primary.name
  sku                 = "Standard"
}

resource "azurerm_site_recovery_fabric" "primary_fabric" {
  name                = "fabric-primary"
  resource_group_name = azurerm_resource_group.primary.name
  location            = azurerm_resource_group.primary.location
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
}

resource "azurerm_site_recovery_fabric" "recovery_fabric" {
  name                = "fabric-recovery"
  resource_group_name = azurerm_resource_group.recovery.name
  location            = azurerm_resource_group.recovery.location
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
}

resource "azurerm_site_recovery_protection_container" "primary_pc" {
  name                = "pc-primary"
  resource_group_name = azurerm_resource_group.primary.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
  fabric_name         = azurerm_site_recovery_fabric.primary_fabric.name
}

resource "azurerm_site_recovery_protection_container" "recovery_pc" {
  name                = "pc-recovery"
  resource_group_name = azurerm_resource_group.recovery.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
  fabric_name         = azurerm_site_recovery_fabric.recovery_fabric.name
}

resource "azurerm_site_recovery_replication_policy" "policy" {
  name                = "asr-policy"
  resource_group_name = azurerm_resource_group.primary.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name

  # Example settings – you’ll tune these
  recovery_point_retention_in_minutes = 60
  application_consistent_snapshot_frequency_in_hours = 4
  multi_vm_sync_status = "Enabled"
}

resource "azurerm_site_recovery_protection_container_mapping" "mapping" {
  name                            = "mapping-pc"
  resource_group_name             = azurerm_resource_group.primary.name
  recovery_vault_name             = azurerm_recovery_services_vault.vault.name
  source_fabric_name              = azurerm_site_recovery_fabric.primary_fabric.name
  target_fabric_name              = azurerm_site_recovery_fabric.recovery_fabric.name
  source_protection_container_name = azurerm_site_recovery_protection_container.primary_pc.name
  target_protection_container_name = azurerm_site_recovery_protection_container.recovery_pc.name
  replication_policy_id           = azurerm_site_recovery_replication_policy.policy.id
}

resource "azurerm_site_recovery_replicated_vm" "vm_replication" {
  name                            = "asr-replica-vm"
  resource_group_name             = azurerm_resource_group.primary.name
  recovery_vault_name             = azurerm_recovery_services_vault.vault.name
  source_vm_id                    = var.vm_id
  recovery_fabric_name            = azurerm_site_recovery_fabric.recovery_fabric.name
  target_resource_group_id        = azurerm_resource_group.recovery.id
  target_availability_zone         = "1"  # if using zones
}
```

⚠️ **Note**: Actual parameters vary by region, subscription, VM type, encryption, network mappings and workloads. There’s no Terraform resource to *initiate* failover – you’ll need to trigger via CLI or PowerShell (or Azure REST) in pipeline. ([Microsoft Learn][9])

---

## 🔍 Key Tips & Caveats for Production-Readiness

* Ensure your target region has **sufficient quota and resources** (VM sizes, storage, vNet) for recovery. ([Microsoft Learn][10])
* Application-consistent snapshots require agents/extensions and may incur impact.
* If your source VM uses encryption (BYOK, KEK, Disk Encryption), validate compatibility with ASR.
* Regularly test failover (both planned and unplanned) to validate your recovery plan.
* Monitor the vault and replication health; use alerts for “replication lag” or “health unhealthy”. ([Microsoft Learn][11])
* Treat your ASR setup as part of your broader BCDR strategy — document roles, responsibilities, workflows (failover, failback, reverse-replication).
* In Terraform, consider using `depends_on` and null-resources/hooks for post-deployment tasks (e.g., initiating failover, updating network mappings) since some operations are not yet available as first-class resources.
* Maintain the IaC definitions in source control (GitHub) and integrate with CI/CD (GitHub Actions or Azure DevOps) so you can version, review and audit changes to DR configuration.

---

If you like, I can **generate a full GitHub-ready repository** (with module structure, CI/CD pipeline YAML, README (English + Hindi + Marathi), and architecture diagrams) tailored for ASR (VM replication + failover + failback) — would that be helpful to you?

[1]: https://n2ws.com/blog/microsoft-azure-cloud-services/azure-site-recovery-cli-tutorial?utm_source=chatgpt.com "Setting Up Azure Site Recovery: A Step-by-Step CLI Tutorial"
[2]: https://learn.microsoft.com/en-us/cli/azure/site-recovery/protected-item?view=azure-cli-latest&utm_source=chatgpt.com "az site-recovery protected-item | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/cli/azure/site-recovery?view=azure-cli-latest&utm_source=chatgpt.com "az site-recovery | Microsoft Learn"
[4]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/site_recovery_fabric?utm_source=chatgpt.com "azurerm_site_recovery_fabric | Resources | hashicorp/azurerm"
[5]: https://shisho.dev/dojo/providers/azurerm/Recovery_Services/azurerm-site-recovery-protection-container/?utm_source=chatgpt.com "Azure Recovery Services Protection Container - Shisho Cloud"
[6]: https://shisho.dev/dojo/providers/azurerm/Recovery_Services/azurerm-site-recovery-protection-container-mapping/?utm_source=chatgpt.com "Azure Recovery Services Protection Container Mapping | Shisho Dojo"
[7]: https://shisho.dev/dojo/providers/azurerm/Recovery_Services/azurerm-site-recovery-replication-policy/?utm_source=chatgpt.com "Azure Recovery Services Replication Policy - Shisho Cloud"
[8]: https://registry.terraform.io/providers/hashicorp/azurerm/3.109.0/docs/resources/site_recovery_replicated_vm?utm_source=chatgpt.com "azurerm_site_recovery_replicate..."
[9]: https://learn.microsoft.com/en-us/answers/questions/2169252/how-do-i-initiate-azure-site-recovery-failover-usi?utm_source=chatgpt.com "How do I initiate Azure Site Recovery failover using Terraform?"
[10]: https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-tutorial-enable-replication?utm_source=chatgpt.com "Tutorial: Set up disaster recovery for Azure VMs - Microsoft Learn"
[11]: https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-troubleshoot-errors?utm_source=chatgpt.com "Troubleshoot Azure VM replication in Azure Site Recovery"

# 🚀 Azure Site Recovery (ASR) – Complete Guide

---

# 🔹 1. What is Azure Site Recovery?

![Image](https://images.openai.com/static-rsc-4/d03dwhZ844O4OTr52_2M6LfQq_yvcglOTGYQyacKXgMSLWt1yxT8Q6BmYPPsWkri_tvZjgyWtZZYk2a7qCfOp0DW1StW1y8lsPpdp41eSE0K0L5UHfYESSqEX8w_yDFFgu9HE2lYgJqsBpPceGMpiyk69XQbsUBIplQIpRmLimRzKOdMz1BB7hRgC8xGAgtN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/AD03toxVIrbgkFq3o69e6WecINrqae-1DBZz-dTTczGK48r9fJYu_D85spR92v_KZCAB-ygU5YoADOjoW59yr8Fa8QkF6Xa_JftWfRpwEkI_2vPQQH3TVi-WzLEILwSAs7yarQ6mcyFfS81SC8s9u83w8xlZuZ_DcdjMc_94LJEJMDr-dfrOuIQAYZHcuj_I?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9ftZ6GShOotqieFTiF_jqJkMpxNYB3eZE2MQ_iViZ-yREKBEdN1hcmZY-wN43GtTd82tYsjH7LkvNIvaFdYC1qWH5uKjk4tw_8nJ4FIcJGV-T9cPukThrfgLFFjSFSbU_RnOnARvto-pfmn228GM4Q_IDY4RvgFwcRFwAm0eHFnEv3-mo_Lxq2LmF2MKRCA8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/v103p6J6HPQrC2ozpm3lUKtbavb9EZk_IAukpAsVIdZFr5RVS1pHj784LQH1yb9o53mQuo7ZvMnMNOkEnf41Om3lDV1k8ya1JRx19fIoKNXT02PSn-uPOqJdrFd19Y-lJcoMt0YQENZq--3mpRh_wBj8RRpusKLYlpjOlLrJt8uKTURPWkd207jgOrpTyGfT?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3K7Ug2a1Hgl7WPw_BFR8F92nyEtSbC0ANPZp7Qqxs7L4QrWuoPwoXkJMoy4NCx6aEtUU2xPdAPWhyAt49JWYIWkDt6N_O1FZPEkA9NdEAceq1RrZfsAxvocdwqPmedfrsFRbJyZw8gprnmH3UNRJFCTDbJ_KBm-THIBoZE5ZztYZvR3Rg_vgGf6xGVWd94x-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EqXyVZ2zgTHeHdb3IvOUWZnZNGb7jWzJ6nOxfq3DCQvyZzUftHgtJ4uRAaonsShdvMc08_V7mqtgWE57mkh0t97UbDgvI-_5UpiBa8GAk87cwD2iClWNXGvE-ctKLIoY_4lT1Zl3JvLnAzf0VmEG6jokthIyPMSyQLXSTxyM_FUlG4_P6YI_2Kw6TChern24?purpose=fullsize)

👉 **Azure Site Recovery (ASR)** = Disaster Recovery (DR) solution

### ✔️ It provides:

* VM replication (Primary → Secondary region)
* Failover (disaster switch)
* Failback (return to primary)

---

# 🔹 2. Core Components

| Component               | Purpose                    |
| ----------------------- | -------------------------- |
| Recovery Services Vault | Central DR management      |
| Source VM               | Primary workload           |
| Target Region           | DR location                |
| Cache Storage           | Temporary replication data |
| Mobility Agent          | Replication engine         |

---

# 🔹 3. Supported Matrix (Important)

## ✔️ OS Support (from your doc)

* Windows Server: 2008 R2 → 2025
* Linux:

  * RHEL: 6 → 10
  * Ubuntu: 14.04 → 24.04
  * Debian: 7 → 12
  * SLES: 11 → 15
  * Oracle Linux, Rocky, Alma

📌 Must use **stock kernel only** 

---

## ✔️ VM Size (CRITICAL)

* Supported:

  * B-series
  * Dv3 / Dv4
  * Ev3 / Ev4
* Avoid:

  * Dv5 / Ev5 (NVMe issue ❌)

---

## ✔️ Disk Support

* Managed disks ✅
* Premium SSD ✅
* NVMe ❌ (common failure)

---

## ✔️ Region Support

* Any Azure region → Any region
* Same geo cluster recommended

📌 Global DR supported 

---

# 🔹 4. Architecture Flow

```
Primary VM
   ↓
Mobility Agent
   ↓
Cache Storage (same region)
   ↓
Target Region Storage
   ↓
Failover VM created
```

---

# 🔹 5. Step-by-Step Setup (Portal)

## 🔸 Step 1: Create Vault

* Go to Azure Portal
* Create → Recovery Services Vault

---

## 🔸 Step 2: Enable Replication

* Vault → Site Recovery → Enable Replication
* Source:

  * Azure VM
* Target:

  * Region (example: Central India → South India)

---

## 🔸 Step 3: Configure Settings

* Select VM
* Choose:

  * Disk type
  * Cache storage account
  * Replication policy

---

## 🔸 Step 4: Install Mobility Agent

* Auto-installed (Azure-to-Azure)
* Or manual (on-prem scenarios)

---

## 🔸 Step 5: Start Replication

* Initial sync starts
* Monitor in:

  * Replicated Items

---

# 🔹 6. Failover Types

| Type               | Use                      |
| ------------------ | ------------------------ |
| Test Failover      | DR testing (no downtime) |
| Planned Failover   | Controlled migration     |
| Unplanned Failover | Disaster scenario        |

---

## 🔸 Failover Steps

* Go to VM → Replicated Items
* Click **Failover**
* Select recovery point
* Choose network

---

# 🔹 7. Failback (Reverse DR)

* After failover:

  * Re-protect VM
  * Sync back to original region
  * Perform failback

---

# 🔹 8. Real Issue (Your Case – NVMe)

## ❌ Error

> NVMe disk controller not supported

---

## ✔️ Fix Steps

### Step 1: Check Disk Type

```bash
lsblk -o NAME,MODEL
```

If shows:

```
nvme0n1 ❌
```

---

### Step 2: Use Supported VM Size

```bash
Standard_D2s_v3
Standard_B2s
```

---

### Step 3: Ensure Kernel Supported

```bash
uname -r
```

---

### Step 4: Re-enable Replication

* Disable → Enable again

---

# 🔹 9. Azure CLI (VM Creation – Non NVMe)

```bash
az vm create \
  --resource-group rg-dr \
  --name vm-asr \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --location centralindia
```

---

# 🔹 10. Key Requirements (Exam Focus)

✔ Supported OS
✔ Supported kernel
✔ Supported VM size
✔ Managed disk
✔ Mobility Agent

---

# 🔹 11. Common Errors & Fix

| Error                | Reason             | Fix             |
| -------------------- | ------------------ | --------------- |
| NVMe not supported   | New VM series      | Use Dv3         |
| Replication failed   | Kernel mismatch    | Update kernel   |
| Agent not installed  | OS issue           | Reinstall agent |
| Region not available | Subscription issue | Enable region   |

---

# 🔹 12. Best Practices (Architect Level)

* Use **region pairs**
* Test failover regularly
* Use **Premium SSD for DR**
* Monitor via Azure Monitor
* Use **separate DR VNet**

---

# 🔹 13. Quick Cheat Sheet

| Area    | Recommendation       |
| ------- | -------------------- |
| VM Size | D2s_v3               |
| OS      | Ubuntu 20.04 / 22.04 |
| Disk    | Premium SSD          |
| Region  | Paired region        |
| Agent   | Latest               |

---

# 🔹 14. Interview Questions

### Q1: What is ASR?

👉 DR solution for VM replication

### Q2: Why NVMe issue?

👉 Unsupported disk controller

### Q3: What is Mobility Agent?

👉 Replication agent

### Q4: Difference: Backup vs ASR?

* Backup = restore data
* ASR = live failover

---

# 🔥 Final Architect Summary

👉 *“Azure Site Recovery replicates workloads across regions using a Mobility Agent and cache storage. For successful DR, VM must use supported OS, kernel, and non-NVMe disk-based VM sizes.”*

---
