# 📦 Azure Backup & Azure Site Recovery (ASR)

### Enterprise-Grade Backup & Disaster Recovery – Architected & Automated

📄 Source file reference: 

---

# 🔹 1. Core Concepts (Foundation First)

## 🔸 Backup vs Disaster Recovery

| Feature       | Azure Backup          | Azure Site Recovery (ASR) |
| ------------- | --------------------- | ------------------------- |
| Purpose       | Data protection       | Business continuity       |
| Recovery Type | Point-in-time restore | Near real-time failover   |
| Scope         | Files, disks, DB, VM  | Full VM / App             |
| RPO           | Hours / Days          | Minutes                   |
| RTO           | Minutes–Hours         | Minutes                   |

### ✅ Architect Rule

* Backup = **Data Integrity**
* ASR = **Service Availability**
* ✔ Production = **Both Required**

---

# 🔹 2. Azure Backup – Architecture Flow

## 🔸 Step-by-Step Flow

1. Create **Recovery Services Vault**
2. Define **Backup Policy**
3. Take **Disk Snapshot**
4. Transfer **Delta Changes**
5. Store **Recovery Points**

---

## 🔸 What Gets Backed Up

* OS Disk
* Data Disk
* App-consistent state (VSS / scripts)

---

# 🔹 3. Azure Backup – Enterprise Design

## 🔸 Vault Configuration (MANDATORY)

| Setting      | Recommendation  |
| ------------ | --------------- |
| Storage      | GRS             |
| Soft Delete  | Enabled         |
| Immutability | Locked          |
| RBAC         | Least Privilege |
| Monitoring   | Azure Monitor   |

---

## 🔸 Policy Design

| Tier   | Frequency        | Retention     |
| ------ | ---------------- | ------------- |
| Tier-1 | Daily + Enhanced | 7D / 4W / 12M |
| Tier-2 | Daily            | 30D           |
| Tier-3 | Daily            | 7D            |

---

# 🔹 4. Azure Backup – Automation

## 🔸 Terraform Flow

```
Vault → Policy → VM Protection
```

✔ Automate using:

* Terraform
* GitHub Actions / Azure DevOps
* Alerts for failures

---

# 🔹 5. Restore Scenarios

| Type         | Use Case          |
| ------------ | ----------------- |
| New VM       | Full recovery     |
| Disk restore | Partial           |
| File restore | Accidental delete |
| Cross-region | DR scenario       |

✔ Always document **restore runbook**

---

# 🔹 6. Azure Site Recovery (ASR)

## 🔸 What ASR Solves

* Region failure
* Data center outage
* DR compliance
* Application recovery

---

## 🔸 A2A Flow

```
Primary VM
   ↓
Replication
   ↓
Target Region
   ↓
Failover VM
```

---

# 🔹 7. ASR Core Components

| Component      | Purpose                |
| -------------- | ---------------------- |
| Recovery Vault | Control plane          |
| Fabric         | Region abstraction     |
| Container      | Grouping               |
| Policy         | RPO rules              |
| RPI            | Protected VM           |
| Recovery Plan  | Failover orchestration |

---

# 🔹 8. ASR Operational Facts

✔ Continuous replication
✔ Crash + App consistent recovery
✔ Retention configurable

---

## 🔸 Failover Types

| Type      | Use        |
| --------- | ---------- |
| Test      | Validation |
| Planned   | Migration  |
| Unplanned | Disaster   |

---

# 🔹 9. ASR CLI – Step-by-Step (Ordered)

## 🔸 Step 1: Install Extension

```bash
az extension add --name site-recovery
```

---

## 🔸 Step 2: Create Resource Groups

```bash
az group create --name rg-primary --location eastus
az group create --name rg-dr --location westus2
```

---

## 🔸 Step 3: Create VM (IMPORTANT – NON NVMe)

```bash
az vm create \
  --resource-group rg-primary \
  --name vm-primary \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys
```

---

## 🔸 Step 4: Create Recovery Vault

```bash
az resource create \
  --resource-group rg-dr \
  --name asr-vault \
  --resource-type Microsoft.RecoveryServices/vaults
```

---

## 🔸 Step 5: Set Context

```bash
az recoveryservices vault set-context \
  --name asr-vault \
  --resource-group rg-dr
```

---

## 🔸 Step 6: Create DR Network

```bash
az network vnet create \
  --resource-group rg-dr \
  --name vnet-dr
```

---

## 🔸 Step 7: Create Policy

```bash
az site-recovery policy create \
  --name A2A-policy \
  --provider-type A2A
```

---

## 🔸 Step 8–11: Enable Replication

✔ Discover VM
✔ Map containers
✔ Enable replication

---

## 🔸 Step 12: Monitor

```bash
az recoveryservices vault replication-protected-item list
```

---

# 🔹 10. Failover Workflow (VERY IMPORTANT)

## 🔸 Sequence

```
Test Failover
   ↓
Failover
   ↓
Commit
   ↓
Re-protect
   ↓
Failback
```

---

## 🔸 Commands

### Test Failover

```bash
az site-recovery recovery-plan test-failover
```

### Failover

```bash
az site-recovery recovery-plan failover
```

### Commit

```bash
az site-recovery recovery-plan failover-commit
```

---

# 🔹 11. NVMe Issue (YOUR ERROR)

## ❌ Problem

```
ASR does not support NVMe disk controller
```

---

## ✔ Fix

### 1. Check disk

```bash
lsblk
```

### 2. Use supported VM size

✔ Recommended:

* Standard_D2s_v3
* Standard_B2s

❌ Avoid:

* Dv5 / Ev5

---

# 🔹 12. Best Practices (Architect Level)

✔ Use region pairs
✔ Separate DR network
✔ Monitor replication
✔ Test failover regularly
✔ Enable immutability
✔ Use RBAC separation

---

# 🔹 13. Security Checklist

✔ Soft delete
✔ Immutable vault
✔ Key Vault access
✔ Audit logs
✔ Backup validation
✔ DR drill evidence

---

# 🔹 14. Operational Excellence

| Check   | Evidence        |
| ------- | --------------- |
| Backup  | Logs            |
| Restore | Test proof      |
| DR      | Failover report |
| RPO     | Replication     |
| RTO     | Recovery plan   |

---

# 🔹 15. Final Architecture

```
Application
 ├─ ASR (Availability)
 │   ├─ Failover
 │   ├─ Recovery Plan
 │
 └─ Backup (Data Protection)
     ├─ Immutable backups
     ├─ Long retention
```

---

# 🔥 Final Architect Summary

👉 **Backup = Data Protection**
👉 **ASR = Disaster Recovery**
👉 **Production = Both Mandatory**

---
