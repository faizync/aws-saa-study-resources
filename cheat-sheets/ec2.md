# 🖥️ AWS EC2 — SAA-C03 Exam Cheat Sheet

---

## ⚡ Key Highlights
- Virtual server (Linux / Windows / Mac-based) in the cloud
- Runs on **AWS Nitro System** → offloads compute to dedicated hardware → near bare-metal performance
- **Nitro Hypervisor** outperforms legacy Xen Hypervisor
- Instance limits per region: **On-Demand** (vCPU-based), **20 Reserved**, **Spot** (dynamic limit)

---

## 🧩 EC2 Features — Quick Reference

| Feature | What It Is |
|---|---|
| **AMI** | Reusable template (OS + apps) |
| **Instance Types** | t/m = General, c = Compute, r/x/z = Memory, d/h/i = Storage, f/g/p/trn/inf = Accelerated |
| **Key Pairs** | Secure login credentials |
| **Security Groups** | Virtual firewall (instance level) |
| **Elastic IP** | Static public IPv4 address |
| **Instance Store** | Temporary storage — deleted on STOP/TERMINATE |
| **EBS** | Persistent block storage |
| **Tags** | Metadata (key-value) for resources |
| **VPC** | Logically isolated virtual network |
| **User Data** | Script run at instance boot (max 16 KB) |
| **EC2 Instance Connect** | SSH (Linux) / RDP (Windows) without managing SSH keys |
| **EC2 Attestation** | Uses NitroTPM to verify instance identity & software integrity |

---

## 🔄 Instance States

| State | Billed? | Storage Preserved? | Notes |
|---|---|---|---|
| **Start/Running** | ✅ Yes | ✅ EBS | Normal operation |
| **Stop** | ❌ No | ✅ EBS / ❌ Instance Store | Can resize, change kernel, attach EBS |
| **Hibernate** | ❌ No (EBS + EIP only) | ✅ RAM saved to EBS | Root EBS must be **encrypted** |
| **Terminate** | ❌ No | ❌ Root EBS (deleted) / ✅ Other EBS | Cannot restart — permanent |

- Enable **termination protection** to prevent accidental termination
- Enable **stop protection** to prevent accidental stopping
- Instance Store-backed → can only **start** or **terminate** (no stop!)
- EBS-backed → can **stop, start, terminate, hibernate**

---

## 💿 Root Device Volumes

| Feature | EBS-backed | Instance Store-backed |
|---|---|---|
| **Boot time** | < 1 minute | < 5 minutes |
| **Root size limit** | 64 TiB | 10 GiB |
| **Data persistence** | Survives stop | Deleted on termination |
| **Stopped state** | ✅ Supported | ❌ Not supported |
| **Modifications** | Type, kernel, RAM disk, user data (while stopped) | Fixed for life |
| **Charges** | Instance + EBS + snapshot (S3) | Instance + S3 |

- Can **replace root volume** of running instance (from: initial state, snapshot, or AMI)
- EBS root volume **deleted by default** on termination
- Can now launch **encrypted EBS instance directly from unencrypted AMI**

---

## 🖼️ AMI

- Contains: **root volume template + launch permissions + block device mapping**
- Backed by **EBS** (snapshot-based) or **Instance Store** (S3-based)
- **Regional** — tied to region; can be **copied across regions**
- **Recycle Bin** — restore accidentally deleted AMIs; set lock retention rules
- Public AMIs **deprecated after 2 years** from creation date
- AMIs from Amazon/verified partners are marked as **verified provider** in console
- **UEFI Secure Boot** — ensures instance only boots cryptographically signed software
- Supports **IMDSv2** configuration

---

## 🏗️ EC2 Image Builder
- Fully managed service to **automate AMI creation, management & deployment**
- Supports Console, CLI, API
- Can set up **pipelines** for automated updates and system patching
- Images created are **owned by you**

---

## 💰 Pricing Models

| Model | Key Facts |
|---|---|
| **On-Demand** | Pay per second, no commitment |
| **Reserved (Standard)** | Up to **60% off**, 1 or 3 years, can sell in RI Marketplace |
| **Reserved (Convertible)** | Up to **54% off**, can exchange for different attributes, **cannot** sell in RI Marketplace |
| **Savings Plans** | Up to **72% off**, commit to $/hour usage, covers EC2 + Fargate + Lambda |
| **Spot** | Up to **90% off**, can be interrupted by AWS |
| **Dedicated Host** | Pay per physical host; BYOL support |
| **Dedicated Instance** | Pay per hour on single-tenant hardware |
| **Capacity Reservation** | Reserve capacity in specific AZ, no term commitment |

### Reserved Instances — Regional vs Zonal

| Feature | Regional RI | Zonal RI |
|---|---|---|
| Capacity reservation | ❌ No | ✅ Yes |
| AZ flexibility | ✅ Any AZ in region | ❌ Specific AZ only |
| Instance size flexibility | ✅ Within family | ❌ Specific size only |
| Queue a purchase | ✅ Yes | ❌ No |

### Spot Instances
- **Spot Blocks** — defined duration, designed not to be interrupted
- **Spot Fleet** — collection of Spot + optional On-Demand instances
- **Spot Capacity Pool** — same instance type + OS + AZ + network platform
- **Rebalance Recommendation** — signal that instance is at elevated risk of interruption

#### Spot Allocation Strategies
- **LowestPrice** — from cheapest pool (default)
- **Diversified** — spread across all pools
- **CapacityOptimized** — from pool with most available capacity
- **InstancePoolsToUseCount** — spread across N pools (used with LowestPrice)

### Capacity Reservations
- No 1 or 3-year commitment needed
- Specify: AZ, instance count, instance type, tenancy, platform/OS
- Apply Savings Plans or Regional RIs for discounts
- ✅ Can be created in **placement groups**
- ❌ Cannot be used with **Dedicated Hosts**

---

## 🔐 Security

- Use **IAM policies & roles** to control access
- **Security Groups** = virtual firewall (instance level):
  - **Stateful** — response traffic automatically allowed
  - **Allow rules only** — no explicit deny rules
  - Evaluates **ALL rules from ALL attached security groups**
  - Default: allows all **outbound**, no **inbound**
  - Default SG: allows all inbound from same SG + all outbound
  - Rules apply immediately to all associated instances
- **Disable password-based SSH logins** → use key pairs instead
- Key pair formats: **.pem** and **.ppk**
- Can **query public key and creation date** of key pairs
- **EC2 Instance Connect & Serial Console** → support **ED25519** keys
- **VPC Traffic Mirroring** → copy/replicate network traffic to security appliances for inspection, threat monitoring, troubleshooting

---

## 🌐 Networking

| Feature | Key Detail |
|---|---|
| **Elastic IP (EIP)** | Static IPv4, region-specific, **5 per region** default limit |
| **EIP Transfer** | Transfer between accounts in **same region**, 7-day acceptance window, tags reset, no charge |
| **Primary ENI (eth0)** | Auto-created, **cannot be detached** |
| **Secondary ENIs** | Can be attached/detached (hot/warm/cold), max count varies by instance type |
| **Bastion Host** | Jump box for SSH/RDP into private VPC instances |

- Instance in **public subnet** → gets public + private IP
- Instance in **private subnet** → private IP only (need EIP for internet)
- ENI attributes (private IP, EIP, MAC address) **follow the ENI** when moved
- Attaching multiple ENIs does **NOT double bandwidth**
- Can attach ENI from different subnet if **same AZ**
- Default ENIs terminated with instance termination

### Enhanced Networking (SR-IOV)
- **Higher bandwidth, higher PPS, lower latency**
- Bypasses hypervisor → talks directly to NIC hardware
- **No additional charge**
- Two mechanisms:
  - **ENA (Elastic Network Adapter)** → up to **100 Gbps** (all Nitro instances)
  - **Intel 82599 VF** → up to **10 Gbps** (older: C3, C4, D2, I2, M4, R3)
- **T2 instances do NOT support Enhanced Networking**

### Elastic Fabric Adapter (EFA)
- Network device for **AI, ML, and HPC** workloads
- **OS-bypass** via **Libfabric API** → skips kernel, talks directly to hardware
- Uses **SRD (Scalable Reliable Datagram)** protocol (not TCP)
- Lower & more consistent latency, higher throughput than TCP
- EFA = **ENA + OS-bypass** (superset of ENA)
- Works with: **NCCL/NIXL** (ML/AI), **Open MPI / Intel MPI** (HPC)
- EFA traffic **cannot cross AZs or VPCs**
- **No additional charge**
- ❌ Not supported on AWS Outposts
- Always pair with **Cluster Placement Group** for max performance

### Load Balancer Types (ELB)

| Type | Layer | Protocol | Use Case |
|---|---|---|---|
| **ALB** | 7 | HTTP/HTTPS | Web apps, microservices, path-based routing |
| **NLB** | 4 | TCP/UDP/TLS | Low latency, gaming, IoT; supports **static IP** |
| **GLB** | 3 | IP | Firewalls, DPI, 3rd party security appliances |
| **CLB** | 4+7 | HTTP/HTTPS/TCP | Legacy only |

- ALB does **NOT** preserve client IP or support static IPs
- NLB **preserves** client IP and supports **static/Elastic IPs**
- ASG auto-registers new instances with attached ELB
- No charge for ASG itself — pay only for EC2 instances launched

---

## 📊 Monitoring

- **CloudWatch** — default metrics every **5 minutes**; enable **Detailed Monitoring** for **1-minute** intervals
- **System Status Checks** — AWS infrastructure (requires AWS to fix)
- **Instance Status Checks** — software/network config (requires YOU to fix)
- **CloudWatch Alarms** — watch metric, trigger action on threshold
- **CloudWatch Events** — automate responses to system events
- **CloudWatch Logs** — collect logs from EC2, CloudTrail, other sources

#### What to Monitor
- Via EC2 metrics: CPU, Network, Disk performance, Disk reads/writes
- Via **CloudWatch agent**: Memory, disk swap, disk space, page file, logs

---

## 📋 Instance Metadata & User Data

- Metadata URL: `http://169.254.169.254/latest/meta-data/`
- User Data URL: `http://169.254.169.254/latest/user-data`
- **NOT** protected by cryptographic methods
- User data types: **shell scripts** and **cloud-init directives**
- User data limit: **16 KB**
- Modifying user data while stopped → **NOT executed on restart**
- Instance **tags** accessible from metadata
- ASG instances → metadata includes **target lifecycle state**

---

## 📦 Placement Groups

| Type | Goal | AZ Span | Limit | Use Case |
|---|---|---|---|---|
| **Cluster** | Max performance / lowest latency | ❌ Single AZ only | No hard limit | HPC, ML, Big Data (tightly coupled) |
| **Spread** | Max fault isolation | ✅ Multiple AZs | **7 instances per AZ** | Small group of critical instances |
| **Partition** | Fault isolation at scale | ✅ Multiple AZs | **7 partitions per AZ** | Hadoop, Kafka, Cassandra |

### Placement Group Rules
- Name must be **unique within AWS account per region**
- **Cannot merge** placement groups
- Instance can be in **only one** placement group at a time
- **Dedicated Hosts (tenancy=host)** → ❌ **cannot** be launched in placement groups
- **Dedicated Instances** → ❌ cannot use **Spread** PG; Partition PG limited to **2 partitions**
- Cannot launch **Spot Instances configured to stop/hibernate on interruption** in PGs
- **No charge** for creating placement groups
- Cluster PG: up to **10 Gbps single-flow** (vs 5 Gbps outside PG)
- Internet traffic from Cluster PG limited to **5 Gbps**
- Max **500 placement groups per region**

---

## 💾 Storage

| Storage | Type | Persistence | Notes |
|---|---|---|---|
| **EBS** | Block | ✅ Persistent | AZ-specific; snapshots to S3 |
| **Instance Store** | Block | ❌ Temporary | Deleted on stop/terminate |
| **EFS** | File | ✅ Persistent | Shared across multiple instances |
| **FSx for Windows** | File | ✅ Persistent | Windows Server-based |
| **FSx for Lustre** | File | ✅ Persistent | High-performance file system |
| **FSx for NetApp ONTAP** | File | ✅ Persistent | NetApp ONTAP-based |
| **FSx for OpenZFS** | File | ✅ Persistent | OpenZFS-based |
| **S3** | Object | ✅ Persistent | EBS snapshots, Instance Store AMIs |

### Torn Write Prevention (TWP)
- Block storage feature for **I/O-intensive relational databases** (MySQL, MariaDB)
- Eliminates need for **doublewrite buffer** → writes are all-or-nothing at hardware level
- Performance gains: up to **+30% TPS**, up to **-50% write latency**
- Supports block sizes: **4 KiB, 8 KiB, 16 KiB**
- Works on **all EBS volumes on Nitro instances**; **enabled by default**
- Must configure MySQL/MariaDB to disable doublewrite buffer to benefit
- Also available on **RDS** as **"RDS Optimized Writes"**
- **No additional cost**

---

## 🗂️ EC2 Resources — Regional vs Global vs AZ

| Resource | Scope |
|---|---|
| AWS Account | Global |
| Key Pairs | Regional (or upload to make global) |
| AMIs | Regional (can copy across regions) |
| Elastic IP Addresses | Regional |
| Security Groups | Regional |
| EBS Snapshots | Regional (can copy across regions) |
| EC2 Resource IDs | Regional |
| **EBS Volumes** | **Availability Zone** |
| **Instances** | **Availability Zone** (ID is regional) |

---

## 🐳 ECS Extensions (Bonus)

### ECS Anywhere
- Run ECS tasks on **on-premises** infrastructure
- Install **SSM Agent + ECS Container Agent** → recognized as "External Instances"
- Use cases: data sovereignty, edge computing, maximizing on-prem investments

### ECS Service Connect
- Simplifies **service-to-service communication** for containerized apps (EC2 + Fargate)
- No need for separate proxy (e.g. App Mesh)
- Features: **Service Discovery** (reliable names), **Traffic Engineering** (auto retries, connection draining), **Observability** (metrics in ECS console)

---

## 🧠 EC2 Hibernation
- Saves **in-memory (RAM) state** to root EBS volume, then shuts down
- Requirements: **encrypted AMI + encrypted root EBS volume**
- Available for: On-Demand and Reserved Instances (M3/M4/M5, C3/C4/C5, R3/R4/R5, Amazon Linux, Ubuntu 18.04 LTS)
- While hibernated: pay only for **EBS + Elastic IPs** (no hourly instance charge)
- Resume using: **start-instances** command (same as normal start)

---

## 🎯 Quick Exam Decision Guide

| Scenario | Answer |
|---|---|
| BYOL (per-socket/per-core licensing) | **Dedicated Host** |
| Physical isolation, compliance | **Dedicated Host or Dedicated Instance** |
| Lowest cost, fault-tolerant workload | **Spot Instances** |
| Predictable workload, 1-3 years | **Reserved Instances** |
| Flexible commitment across services | **Savings Plans** |
| HPC / ML distributed training | **EFA + Cluster Placement Group** |
| Lowest latency between instances | **Cluster Placement Group** |
| Hadoop / Kafka / Cassandra resilience | **Partition Placement Group** |
| Few critical instances, max isolation | **Spread Placement Group** |
| Replicate/inspect network traffic | **VPC Traffic Mirroring** |
| Web app HTTP routing, microservices | **ALB** |
| Ultra-low latency, static IP, gaming | **NLB** |
| 3rd party firewall / deep packet inspection | **GLB** |
| MySQL/MariaDB performance boost | **Torn Write Prevention** |
| Instance fails, keep same IP instantly | **Move Secondary ENI** to standby |
| SSH without managing keys | **EC2 Instance Connect** |
| Connect to instance without open ports | **Session Manager (SSM)** |
| Basic network metadata logging | **VPC Flow Logs** |
| Full packet content inspection | **VPC Traffic Mirroring** |

---

*Last verified against AWS official docs — June 2026*
