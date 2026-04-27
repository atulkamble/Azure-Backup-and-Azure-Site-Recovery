# 🌍 Azure Disaster Recovery (ASR) – Practical Lab

## 🎯 Objective

Simulate DR by replicating an Azure VM to another region and perform:

* Failover
* Test failover
* Failback

---

# 🧱 Architecture

```
Primary Region (East US)
        ↓ Replication
Secondary Region (West US)
        ↓
Failover VM runs here during disaster
```

---

# ⚙️ Prerequisites

* Azure Account (Free / Pay-as-you-go)
* Azure CLI installed OR Cloud Shell
* One Linux VM (source)
* Two regions (Primary + DR region)

---

# 🚀 Step 1: Create Resource Groups

```bash
az group create --name rg-primary --location eastus
az group create --name rg-dr --location westus
```

---
```
az vm list-sizes --location eastus -o table
```
---
# 🖥️ Step 2: Create Source VM (Primary Region)

```bash
az vm create \
  --resource-group rg-primary \
  --name vm-primary \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --location eastus

The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'. (Code: ResourceDeploymentFailure)
Azure Site Recovery does not support protection of virtual machines using an NVMe disk controller when the guest operating system is not compatible with ASR NVMe requirements. Detected OS: ubuntu, Version: 22.04. (Code: 151273)

az vm create \
  --resource-group rg-primary \
  --name vm-primary \
  --image Ubuntu2204 \
  --size Standard_L2aos_v4 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --location eastus
```

👉 Verify:

```bash
az vm list -d -o table
```

---

# 🛡️ Step 3: Create Recovery Services Vault

```bash
az resource create \
  --resource-group rg-dr \
  --name asr-vault \
  --resource-type Microsoft.RecoveryServices/vaults \
  --is-full-object \
  --properties '{
    "location": "westus",
    "sku": {
      "name": "Standard"
    },
    "properties": {
      "publicNetworkAccess": "Enabled"
    }
  }'
```

---

# 🔐 Step 4: Set Vault Context

```bash
az recoveryservices vault set-context \
  --name asr-vault \
  --resource-group rg-dr
```

---

# 🔁 Step 5: Enable Replication

⚠️ CLI for full ASR setup is complex → Use **Portal for replication enablement**

👉 Portal Steps:

1. Go to **Recovery Services Vault | asr-vault | Site Recovery | Enable Site Recovery**
2. Click **+Enable Replication**
3. Select:

   * Source: Azure
   * Resource Group: rg-primary
   * Account Automation: Create
   * VM: `vm-primary`
   * Target region: `westus`
4. Enable replication

---

# 📊 Step 6: Monitor Replication

```bash
az recoveryservices vault replication-protected-item list \
  --resource-group rg-dr \
  --vault-name asr-vault \
  --output table
```

---

# 🧪 Step 7: Test Failover (SAFE TEST)

👉 Portal:

* Go to Vault → Replicated Items
* Select VM → **Test Failover**
* Choose network (create test VNet)

✔️ This does NOT impact production

---

# 🔥 Step 8: Actual Failover (Disaster Simulation)

```bash
# Failover is mostly portal-driven for clarity
```

👉 Portal Steps:

1. Select replicated VM
2. Click **Failover**
3. Choose recovery point (latest)
4. Start failover

✔️ VM starts in DR region

---

# 🔄 Step 9: Commit Failover

👉 After validation:

```bash
# Portal → Commit Failover
```

✔️ Now DR VM becomes primary

---

# 🔁 Step 10: Re-Protect (Failback Setup)

```bash
# Reverse replication direction
```

👉 Portal:

* Click **Re-protect**
* Set original region as target

---

# 🔙 Step 11: Failback

* Perform failover again → back to original region

---

# 📌 Validation Commands

## Check VM in DR Region

```bash
az vm list -g rg-dr -o table
```

## Check replication health

```bash
az recoveryservices vault replication-protected-item list \
  --vault-name asr-vault \
  --resource-group rg-dr \
  --query "[].{Name:name,State:provisioningState}"
```

---

# 💡 Important Concepts

| Concept    | Meaning                              |
| ---------- | ------------------------------------ |
| RPO        | Recovery Point Objective (data loss) |
| RTO        | Recovery Time Objective (downtime)   |
| Failover   | Switch to DR region                  |
| Failback   | Return to primary                    |
| Re-protect | Reverse replication                  |

---

# ⚠️ Best Practices

* Use **separate VNet in DR region**
* Enable **Monitoring (Azure Monitor)**
* Test DR regularly (monthly)
* Use **Availability Zones + ASR**
* Tag DR resources properly

---

# 🧪 Bonus: Automate DR using PowerShell

```powershell
# Login
Connect-AzAccount

# Select Vault
$vault = Get-AzRecoveryServicesVault -Name "asr-vault"
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

# Get replicated items
Get-AzRecoveryServicesAsrReplicationProtectedItem
```

---

# 📦 Real-World Use Case

* Production App in East US
* Database + Web Server replicated to West US
* During outage → Failover in minutes
* Zero manual infra setup

---

# 📚 Next Level (Advanced Practice)

* Multi-tier DR (App + DB)
* DR with **Azure SQL Failover Groups**
* DR with **Terraform + ASR**
* DR Drill Automation (Runbooks)

---
