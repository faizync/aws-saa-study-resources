# 🖥️ Amazon EC2 — SAA-C03 Cheat Sheet

> **What it is:** Resizable virtual servers (Linux/Windows/Mac) you rent in the cloud, with full control over CPU, memory, storage, and networking. *Think: "rent a computer by the second."*

> ⏱️ **Source freshness:** Based on Tutorials Dojo (last updated **Dec 20, 2025**). Core EC2 concepts are stable, so this is reliable for the exam. If you want me to double-check any specific limit or new feature against current AWS docs, just ask.

---

## 1. Foundations (Nitro, Instances, AMIs)

| Concept | Plain-English meaning |
|---|---|
| **AWS Nitro System** | The modern hardware platform under EC2. It **offloads** hypervisor/networking/storage jobs to dedicated chips → performance is **indistinguishable from bare metal**, faster than the old Xen hypervisor. *Mental model: a butler handles all the chores so the server can focus 100% on your workload.* |
| **Instance** | The running virtual server itself. |
| **AMI (Amazon Machine Image)** | A reusable **template** (OS + apps + settings) used to launch instances. *Think: a "save file" you boot copies from.* |
| **Instance type** | The hardware spec combo (CPU/mem/storage/network). E.g. `t3.micro`. |
| **EC2 Instance Connect** | Connect via SSH/RDP **without managing your own SSH keys**. |
| **EC2 Instance Attestation** | Uses NitroTPM to **cryptographically prove** the instance identity + software integrity (Attestable AMIs). *Shifts from "trust me" → "verify me."* |

**Key highlights:** 30+ Regions & Local Zones · processor choice (Intel / AMD / **Graviton**) · verified boot via **NitroTPM** + VPC isolation.

---

## 2. Instance Type Families (memorize the letters!)

| Category | Letters | Easy way to remember |
|---|---|---|
| **General Purpose** | `t`, `m` | **"T/M = Team Member"** — balanced all-rounders |
| **Compute Optimized** | `c` | **C = CPU** |
| **Memory Optimized** | `r`, `x`, `z` | **R = RAM** (x, z = extra-large memory) |
| **Storage Optimized** | `d`, `h`, `i` | **"DHI = Disk-Heavy I/O"** |
| **Accelerated Computing** | `f`, `g`, `p`, `trn` (Trainium), `inf` (Inferentia) | **GPU/ML**: `g`/`p` = graphics/parallel, `trn`/`inf` = AI train/infer |

> 🎯 **Exam hook:** Need fast node-to-node compute for HPC/ML? → accelerated/compute types **+ Cluster placement group + EFA** (see §10/§12).

---

## 3. Instance States

| State | What happens | Billing | Notes |
|---|---|---|---|
| **Start** | Runs normally | Charged while running | — |
| **Stop** | Normal shutdown; can restart anytime | **No usage charge** (still pay for EBS) | EBS volumes stay attached; **instance store data is LOST**. Can change type/kernel/RAM, attach/detach EBS, create AMI. |
| **Hibernate** | Writes RAM → file on **root EBS**, then shuts down | Pay only for EBS + Elastic IPs (no hourly) | **Root EBS + AMI must be encrypted.** Resumes where you left off. |
| **Terminate** | Permanent delete | None after | **Cannot restart.** Root volume deleted by default; other EBS preserved by default; instance store data lost. |

**Protection toggles:**
- **Termination protection** → blocks accidental terminate.
- **Stop protection** → blocks accidental stop.

> 🪤 **Trap:** Only **EBS-backed** instances can be **Stopped**. Instance Store-backed instances can only be **started or terminated** (no stop).

---

## 4. Storage Backing: Instance Store vs EBS-backed

| | **Instance Store-backed** | **EBS-backed** |
|---|---|---|
| Data persistence | **Dies with the instance** (stop/terminate/fail = gone) | Survives stop; root deleted on terminate by default, other EBS persists |
| Can be Stopped? | ❌ No (start/terminate only) | ✅ Yes |
| Boot time | < 5 min | **< 1 min** |
| Root device size limit | **10 GiB** | **64 TiB** |
| AMI stored in | S3 | EBS snapshot |
| Modify type/kernel/userdata? | ❌ Fixed for life | ✅ While stopped |
| Charges | Instance + AMI in S3 | Instance + EBS + AMI snapshot |

> **Easy way to remember:** *Instance store = scratchpad (fast, temporary). EBS = hard drive (persistent).*

---

## 5. Root Device Volumes & AMIs

**Root device volume** = the boot image of the instance. You can replace it on a running instance via: initial launch state, snapshot, or AMI.

**An AMI contains:**
1. Template for the root volume (OS, app server, apps)
2. **Launch permissions** (which AWS accounts can use it)
3. **Block device mapping** (which volumes attach at launch)

**AMI extras worth knowing:**
- You can launch **encrypted EBS-backed instances directly from unencrypted AMIs** now (no copy step needed).
- AMIs are **Regional** — you can **copy** them across regions (data transfer charge applies).
- **Recycle Bin** restores deleted AMIs; **lock retention rules** protect against deletion/modification.
- **Public AMIs deprecate after 2 years** from creation by default.
- AMI state changes emit **EventBridge** events → automate responses.
- **UEFI Secure Boot** = only boots software signed with crypto keys.
- Configure AMI to use **IMDSv2** (more secure metadata access).
- `LastLaunchedTime` shows when an AMI was last used.

### EC2 Image Builder
> Fully-managed service that **automates building, patching, and deploying AMIs** via pipelines. *Think: a CI/CD assembly line for golden images.* Images it creates are **owned by you**.

---

## 6. Pricing — Purchase Options (BIG exam topic)

| Option | Plain-English | When to pick |
|---|---|---|
| **On-Demand** | Pay by the second, no commitment | Short, unpredictable, dev/test |
| **Savings Plans** | Commit to **$/hour** for 1 or 3 yrs → up to **72% off**. Works for EC2, **Fargate, Lambda** | Steady overall usage, want flexibility across services |
| **Reserved (RI)** | Commit to a specific instance for 1/3 yrs → big discount | Steady, predictable, known instance family |
| **Spot** | Bid on spare capacity → up to **90% off**, but can be interrupted | Fault-tolerant, flexible (batch, CI, rendering) |
| **Dedicated Host** | Pay for a **whole physical server** | **BYOL** (per-socket/per-core/per-VM licenses) + compliance |
| **Dedicated Instance** | Single-tenant hardware, billed hourly | Isolation but no license-binding control |
| **On-Demand Capacity Reservation** | Reserve capacity in an AZ, **no 1/3-yr term** | Guarantee capacity (e.g., for a known event) without long commitment |

> 🪤 **Classic trap — Dedicated Host vs Dedicated Instance:**
> - **Dedicated Host** = you see/control the **physical sockets & cores** → required to **use existing server-bound (BYO) software licenses** + meet compliance.
> - **Dedicated Instance** = isolated hardware but **no visibility into sockets/cores** → can't do per-socket BYOL.
> *If a question mentions "existing/server-bound licenses" → **Dedicated Host**.*

### Reserved Instances: Standard vs Convertible

| | **Standard RI** | **Convertible RI** |
|---|---|---|
| Discount (1 yr / 3 yr) | **40% / 60%** (bigger) | **31% / 54%** (smaller) |
| Change AZ, instance size (Linux), networking type | ✅ | ✅ |
| Change **instance family, OS, tenancy, payment option** | ❌ | ✅ |
| Sell in **RI Marketplace** | ✅ | ❌ |

> **Remember:** **Standard = Stronger discount, Stiff** (limited changes). **Convertible = Changeable, Cheaper discount.**

### Reserved Instances: Regional vs Zonal

| | **Regional RI** | **Zonal RI** |
|---|---|---|
| **Reserve capacity** | ❌ No | ✅ **Yes** |
| AZ flexibility | ✅ Any AZ in the Region | ❌ Only the specified AZ |
| Instance size flexibility | ✅ Across the family | ❌ Specific type+size only |
| Queue a purchase | ✅ | ❌ |

> **Hook:** Need a **capacity guarantee** → **Zonal** RI (or a Capacity Reservation). Want **flexibility** → **Regional** RI.

---

## 7. Spot Instances Deep-Dive

| Term | Meaning |
|---|---|
| **Spot Instance** | Spare capacity at up to 90% off; can be interrupted when AWS needs it back, price exceeds your max, or demand rises. |
| **Spot block (defined duration)** | Designed **NOT to be interrupted** — runs continuously for a set duration. Good for finite jobs (batch, encoding, CI). |
| **Spot Fleet** | A collection of Spot (+ optional On-Demand) instances aiming for a **target capacity**; auto-maintains it if instances drop. |
| **Spot Capacity pool** | Set of unused instances with the **same type, OS, AZ, network platform**. |
| **Rebalance recommendation** | Signal that a running Spot instance is at **elevated risk of interruption** (act before it's gone). |

**Spot Allocation Strategies:**

| Strategy | Behavior |
|---|---|
| **LowestPrice** (default) | From the cheapest pool |
| **Diversified** | Spread across **all** pools (resilience) |
| **CapacityOptimized** | From pool with **optimal capacity** (fewest interruptions) |
| **InstancePoolsToUseCount** | Spread across **N pools you specify** (only valid with lowestPrice) |

**Spot vs On-Demand quick contrast:** Spot price **fluctuates** with demand & auto-retries launch when capacity returns; On-Demand price is **static** and throws an **ICE (Insufficient Capacity Error)** if no capacity. You control when On-Demand stops/terminates; AWS may interrupt Spot.

---

## 8. Security

- Control access with **IAM** (policies + roles). Restrict to trusted hosts/networks.
- **Security Group = virtual firewall** for instances:
  - **Stateful** (return traffic auto-allowed).
  - Rules are **permissive only** — you **cannot create deny rules**.
  - **All outbound allowed by default.**
  - AWS **evaluates all rules from all SGs** attached to an instance.
  - New rules apply to **all associated instances immediately**.
  - No SG specified at launch → gets the VPC's **default SG** (allows traffic from instances in the same default SG + all outbound).
- **Disable password logins** on custom AMIs (passwords get cracked).
- **Traffic Mirroring**: copy an instance's network traffic to security/monitoring appliances for inspection.
- Key pairs: choose format **.pem** (OpenSSH) or **.ppk** (PuTTY); **ED25519** keys supported for Instance Connect & Serial Console; can query public key + creation date.

---

## 9. Networking

| Item | Key facts |
|---|---|
| **Elastic IP (EIP)** | Static **IPv4** you can **remap** to another instance to mask failure. **Region-specific.** Default **5 per region**. **Transferable** between accounts. |
| **EIP charging trap** 🪤 | AWS charges a **small hourly fee when an EIP is NOT associated** with a running instance (or attached to a stopped instance / unattached ENI), and for **extra** EIPs on an instance. *Idle = you pay.* |
| **Default IPs** | Private subnet → private IP only. Public subnet → public + private IP. |
| **ENI (Elastic Network Interface)** | Virtual network card. **Primary ENI (eth0) cannot be detached.** Can attach extra ENIs (max varies by type). Can attach an ENI to an instance in a **different subnet but same AZ**. |
| **Bastion host / jump box** | An instance used to securely SSH/RDP into private VPC instances. |
| **Enhanced Networking** | Higher bandwidth, higher PPS, lower latency via **SR-IOV**. Used in placement groups. |
| **Elastic Fabric Adapter (EFA)** | Special device for **HPC & ML** — ultra-low latency, high throughput for **inter-instance** communication. *Think: the racing tire for tightly-coupled clusters.* |

> 🪤 **EFA vs Enhanced Networking:** Both boost networking, but **EFA** is the one for **HPC/ML tightly-coupled** workloads (OS-bypass). Enhanced Networking (SR-IOV) is the general high-perf baseline.

---

## 10. Placement Groups

| Type | What it does | Use when | Memory hook |
|---|---|---|---|
| **Cluster** | Packs instances **close together in ONE AZ** → lowest latency, highest throughput | HPC, tightly-coupled node-to-node | **"Cluster = Close & fast"** |
| **Spread** | Spreads instances onto **separate hardware** | Small # of **critical** instances kept apart | **"Spread = Safe/separate"** (max **7 per AZ per group**; can span multiple AZs) |
| **Partition** | Groups into **logical partitions** on different hardware racks | Large distributed/replicated apps (HDFS, HBase, Cassandra) | **"Partition = big data, fault isolation"** |

> 🎯 **Exam answer:** *Low-latency for tightly-coupled HPC across instances* → **Cluster placement group in a single AZ** (NOT spread, NOT multi-AZ/multi-region).

**Placement group rules:** name unique per account per region · **can't merge** groups · an instance is in **one group at a time** · **`host` tenancy can't** be in a placement group.

---

## 11. Monitoring

- **What to monitor with built-in EC2 metrics:** CPU, network, disk performance, disk reads/writes.
- **Needs a CloudWatch agent (NOT default):** **memory utilization, disk swap, disk space, page file, logs.** *Trap: memory & disk-space are NOT in default metrics.*
- **Status checks:**
  - **System Status Check** → AWS-side problem (needs AWS to fix).
  - **Instance Status Check** → your instance's software/network config (needs **you** to fix).
- **CloudWatch:** Alarms (act on a metric vs threshold) · Events (auto-respond to system events) · Logs (collect log files).
- **Default metric interval = 5 minutes.** Enable **detailed monitoring = 1 minute.**

---

## 12. Instance Metadata & User Data

| | Metadata | User Data |
|---|---|---|
| What | Data **about** the instance (config/management) | A **boot script** that runs at launch (shell or cloud-init) |
| Endpoint | `http://169.254.169.254/latest/meta-data/` | `http://169.254.169.254/latest/user-data` |
| Limit | — | **16 KB** |

- **Neither is encrypted** — don't put secrets in user data.
- 🪤 **Trap:** If you **stop → edit user data → start**, the updated user data **does NOT re-run** on start (only runs at first launch).
- Instance **tags** and (with Auto Scaling) the **target lifecycle state** are accessible from metadata.

---

## 13. Storage Options Summary

| Storage | Use it for |
|---|---|
| **EBS** | Durable block storage for an instance; frequent granular updates. Snapshot → stored in **S3**, restore to a new volume/instance. |
| **Instance Store** | Temporary block storage; **dies with the instance**. |
| **EFS** | Scalable **shared file** storage mountable by **many instances** at once (Linux). |
| **FSx** | Windows File Server (SMB), **Lustre** (HPC), NetApp ONTAP, OpenZFS. |
| **S3** | Object storage; holds EBS snapshots & instance-store AMIs. |

> **Torn write prevention** (block storage feature) boosts I/O-intensive relational DB performance without sacrificing resiliency.

---

## 14. Resource Scope (Global / Regional / AZ)

| Resource | Scope |
|---|---|
| AWS account | **Global** |
| Key pairs | Global *or* Regional (created per region; upload to each region to "globalize") |
| AMIs, EIPs, Security Groups, EBS **snapshots**, resource IDs, resource names | **Regional** |
| **EBS volumes**, **Instances** | **Availability Zone** (instance **ID** is regional) |

> **Hook:** *Volumes & instances live in an AZ; almost everything else is regional; only the account is global.* You can **copy** AMIs and snapshots across regions.

---

## 15. ECS Extensions on EC2 (know the concept)

- **ECS Anywhere** — Run ECS tasks on **on-premises** servers using the same ECS control plane. Install **SSM Agent + ECS Container Agent**; AWS treats them as **"External Instances."** Use: data sovereignty, edge/local processing, reuse existing on-prem hardware.
- **ECS Service Connect** — Simplifies **service-to-service** comms, traffic management, and observability (EC2 **and** Fargate) **without** running a separate proxy/App Mesh. Gives service discovery (stable names like `http://payment-service`), auto-retries/connection draining, and built-in traffic metrics.

---

## 16. Misc. facts worth a point

- Default limits: **On-Demand** by **vCPU-based limit**, **20 Reserved Instances**, **Spot** by dynamic regional limit.
- **Host Recovery** auto-restarts instances on a new host after **Dedicated Host** hardware failure.
- **EC2 Hibernation** supported on freshly launched **M3/M4/M5, C3/C4/C5, R3/R4/R5** running **Amazon Linux & Ubuntu 18.04 LTS**; requires **encrypted EBS-backed** instance.
- You can allow **automatic connection of EC2 instances to an RDS database**.
- Capacity Reservations: can be in **placement groups**, **can't** be used with **Dedicated Hosts**, monitored in CloudWatch; pair with **Savings Plans / regional RIs** for billing discounts.
- Cross-region data transfer is billed as **"Data Transfer Out"** (source) + **"Data Transfer In"** (destination).

---

## ⚡ 30-Second Final Recall

- **Nitro** = near bare-metal perf (offloads to dedicated hardware).
- **Only EBS-backed instances can Stop;** instance store = start/terminate only & data dies.
- **Hibernate** = RAM→encrypted root EBS; needs encrypted AMI + root.
- **Boot:** EBS < 1 min, instance store < 5 min. **Root size:** EBS 64 TiB, instance store 10 GiB.
- **Pricing:** Savings Plans up to **72%**, Spot up to **90%**, RI Standard 1/3yr = **40/60%**, Convertible = **31/54%**.
- **Standard RI** = bigger discount + sellable; **Convertible** = changeable family/OS.
- **Zonal RI / Capacity Reservation = reserve capacity;** Regional RI = flexible, no capacity reservation.
- **Dedicated HOST** = BYO server-bound licenses + see sockets/cores (Dedicated *Instance* can't).
- **Cluster PG** = low-latency single-AZ HPC; **Spread** = critical apart (max 7/AZ); **Partition** = big distributed data.
- **EFA** = HPC/ML tightly-coupled networking; **Enhanced Networking** = SR-IOV baseline.
- **Security Groups:** stateful, allow-only (no deny), all outbound open by default.
- **EIP:** static IPv4, 5/region, **charged when idle/unassociated.**
- **Metadata** at `169.254.169.254`; **user data ≤ 16 KB**, runs at launch only (not on stop→start).
- **Default monitoring = 5 min**, detailed = 1 min; **memory & disk-space need the CloudWatch agent.**
- **Volumes & instances = AZ-scoped;** AMIs/snapshots/SGs/EIPs = Regional; account = Global.
