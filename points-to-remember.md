# 📌 Azure Site Recovery — Points to Remember

### 1️⃣ Purpose

* Provides **Disaster Recovery (DR)** and **Business Continuity** during outages.
* Replicates workloads to **another Azure region or datacenter**.
* Enables **failover of applications, VMs, and servers** during disasters.

---

### 2️⃣ Supported Workloads

Azure Site Recovery can protect:

* **Azure Virtual Machines**
* **VMware Virtual Machines**
* **Hyper-V Virtual Machines**
* **Physical Windows/Linux servers**

---

### 3️⃣ Replication Types

ASR supports **continuous replication** of workloads.

Important options:

* **Crash-consistent recovery points**
* **Application-consistent recovery points**

Recovery points can be stored for **up to 15 days**.

---

### 4️⃣ Replication Frequency

Replication intervals vary depending on platform:

* **Hyper-V → as low as 30 seconds**
* **VMware / Physical servers → near continuous replication**

---

### 5️⃣ Failover Capabilities

ASR supports multiple failover operations:

* **Test Failover**
* **Planned Failover**
* **Unplanned Failover**

These help organizations **simulate disaster recovery scenarios safely**.

---

### 6️⃣ Test Failover (Very Important)

Allows testing DR without impacting production.

Benefits:

* No downtime
* Validates DR strategy
* Ensures recovery plans work

---

### 7️⃣ Recovery Plans

Recovery plans allow orchestration of multi-tier applications.

Capabilities:

* Control **VM startup order**
* Execute **Azure Automation runbooks**
* Automate failover workflows

Example:

```
Database VM → Application VM → Web VM
```

---

### 8️⃣ Centralized Monitoring

ASR provides:

* Replication job monitoring
* Alerts and notifications
* Replication health status

Integrated with:

* **Azure Monitor**
* **Log Analytics**

---

### 9️⃣ Security Feature

Supports **Private Link** for secure replication.

This ensures:

* Replication traffic stays inside **private network**
* No exposure to public internet

---

### 🔟 Pricing Model

Billing is based on:

* **Number of protected instances**

Important point:

* **First 31 days are FREE per instance**

---

# ⚡ Key Exam / Interview Points

* Azure Site Recovery = **Disaster Recovery service**
* Supports **VMware, Hyper-V, Azure VMs, Physical Servers**
* Provides **continuous replication**
* Recovery plans automate **multi-tier failover**
* Supports **test failover without downtime**
* Replication frequency **as low as 30 seconds**
* Pricing based on **protected instances**

---

✅ **One-Line Definition (Exam Ready)**

> Azure Site Recovery is a disaster recovery service that replicates workloads to another Azure region or datacenter to ensure business continuity during outages.
