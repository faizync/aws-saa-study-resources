# 🌱 AWS Elastic Beanstalk — SAA-C03 Cheat Sheet

**What it is (one line):** A **Platform-as-a-Service (PaaS)** that lets you upload your app code and AWS auto-provisions and manages the infrastructure for you — capacity provisioning, load balancing, auto scaling, and health monitoring.

> **Easy way to remember:** *"I bring the code, AWS brings the servers."* You give Beanstalk a `.zip` of your app; it builds the EC2 + ELB + Auto Scaling stack underneath and keeps it healthy. You still own/see the resources (they're plain EC2, ELB, etc.) — Beanstalk just orchestrates them.

**Where it sits on the abstraction ladder:** More hands-off than EC2 (you manage servers) but more hands-on than Lambda (fully serverless). Beanstalk = *managed servers you can still touch*.

---

## 🎯 The Two Distinctions Examiners Love to Confuse

These two pairs look similar and are classic traps. Burn them in:

| Concept | Question it answers | Options |
|---|---|---|
| **Environment TIER** | *What kind of work does the app do?* | **Web Server** (handles HTTP requests) vs **Worker** (pulls jobs from an SQS queue) |
| **Environment TYPE** | *How many instances / scaling behavior?* | **Single-Instance** (1 EC2 + Elastic IP) vs **Load-Balanced + Auto Scaling** (many EC2 behind an ELB) |

> **Mental model:** *Web tier = answers visitors at the front desk. Worker tier = chews through a to-do (SQS) queue in the back room.*
> **Don't mix it up:** "Tier" = web vs worker (the **job**). "Type" = single vs load-balanced (the **size/scaling**).

### Environment Tiers (in detail)
| Tier | What it does | Plain-English |
|---|---|---|
| **Web Server Environment** | Serves HTTP requests; sits behind an ELB | The public-facing website/API |
| **Worker Environment** | Pulls tasks from an **Amazon SQS queue** and processes them | Background job runner — good for long/async tasks |

### Environment Types (in detail)
| Type | What it gives you | Use when |
|---|---|---|
| **Single-Instance** | One EC2 instance + an **Elastic IP** (no load balancer) | Dev/test, low traffic, cost-sensitive |
| **Load-Balancing + Auto Scaling** | ELB + Auto Scaling group that adds/removes instances as load changes | Production, variable/high traffic, high availability |

---

## 🚀 Deployment Policies — THE most tested topic

When you deploy a **new application version**, Beanstalk gives you several rollout strategies. Know the trade-offs cold.

| Policy | How it works (source) | Downtime? | Extra instances/cost? | Rollback speed | Best for |
|---|---|---|---|---|---|
| **All at once** | New version to **all instances at the same time** | **Yes** — briefly out of service | No extra | Slow (must redeploy) | Fastest deploy; dev where downtime is OK |
| **Rolling** | Deploys in **batches** (some instances at a time) | No full outage, but **reduced capacity** during deploy | No extra (reuses instances) | Slow-ish | When small capacity dip is acceptable |
| **Rolling with additional batch** | **Launches a new batch of instances first**, then deploys in batches | No, and **full capacity kept** | Small temporary cost (the extra batch) | Slow-ish | Prod where you can't lose capacity |
| **Immutable** | Deploys to a **brand-new set of instances**; old ones swapped out | No | **Highest** (temporarily doubles instances) | **Fast** (just terminate the new ones) | Safest standard deploy for prod |
| **Traffic splitting** | New version on a **separate fleet**; forwards a **configurable % of traffic** to it | No | Higher (extra fleet) | Fast | **Canary testing** — validate health before full rollout |

> **Order safest → fastest (memory hook):**
> **Immutable → Rolling-with-additional-batch → Rolling → All-at-once**
> (Safer/more available = more instances/cost & slower. Faster/cheaper = riskier.)

> **Quick picks:**
> - *Need zero downtime + easy rollback?* → **Immutable**
> - *Need to keep full capacity while deploying?* → **Rolling with additional batch**
> - *Want to test the new version on a small % of real traffic (canary)?* → **Traffic splitting**
> - *Don't care about a brief outage, want it done now?* → **All at once**

🪤 **Trap:** "Canary" / "configurable percentage of traffic" → **Traffic splitting** (not Immutable). "Doubles instances then swaps" → **Immutable**.

---

## 🧩 Core Concepts (vocabulary)

| Term | Meaning | Plain-English / analogy |
|---|---|---|
| **Application** | Logical collection of EB components (environments, versions, configs) | Like a **folder** that holds everything for one app |
| **Application Version** | A specific, labeled iteration of deployable code; **points to an S3 object** containing the code. Each version is unique | A **tagged release** sitting in S3 |
| **Environment** | A version **deployed onto AWS resources**. Runs **only ONE application version at a time** (but you can run the same/different versions across many environments) | The actual **running copy** (e.g., dev, staging, prod) |
| **Environment Tier** | Whether resources support **HTTP requests (web server)** or **SQS queue tasks (worker)** | See tiers table above |
| **Environment Configuration** | Parameters/settings that define how an environment & its resources behave | The **settings dial** for an environment |
| **Saved Configuration** | A reusable starting point for creating new environment configs | A **template** of settings |
| **Platform** | Combination of **OS + language runtime + web/app server + EB components** | The **runtime stack** your code runs on |

🪤 **Trap:** An **Application Version** lives in **S3**. An **Environment** runs **one** version at a time.

**Application version limit:** There's a cap on how many versions you can keep. Apply an **application version lifecycle policy** to auto-delete old versions (by age, or once total exceeds a set number) so you don't hit the limit.

---

## ⚙️ What's Inside an Environment (Configuration)

- **EC2 instances** configured to run your web app on your chosen **platform**.
- **Auto Scaling group** — guarantees **≥1 instance** in a single-instance env; allows an **instance range** in a load-balanced env.
- **Elastic Load Balancer** — created automatically when you **enable load balancing** to distribute traffic across instances.
- **Amazon RDS integration** — can add a DB: **MySQL, PostgreSQL, Oracle, or SQL Server**. Beanstalk injects connection info into your app via **environment properties** (DB hostname, port, username, password, db name).
- **Environment properties** — pass **secrets, endpoints, debug settings**, etc. Lets you run the same app across **dev / test / staging / prod**.
- **Amazon SNS** — configure to **notify you of important events** affecting your app.
- **Subdomain** on `elasticbeanstalk.com` — you pick a unique subdomain for the environment.
- **Shared Application Load Balancer** — one ALB can serve **multiple apps across multiple EB environments in the same VPC** (cost-saving).
- **Environment links** — named references that connect component environments together.

### 🔑 Domain name format
```
subdomain.region.elasticbeanstalk.com
```

### 🔁 Rebuilding & Updates
- **Rebuild terminated environments** within **six (6) weeks** of termination with the **same name, ID, and configuration**.
- **Managed Platform Updates** — Beanstalk can **automatically apply platform updates** (OS, runtime, web server patches) during a **scheduled maintenance window**, keeping the env secure with no manual work.

### 🪤 Database Best Practice (high-value exam fact)
> **For production, provision RDS *externally* (outside Beanstalk) and connect via connection strings.**
> **Why:** If the RDS DB is created *inside* the Beanstalk environment, terminating the environment **deletes the database too**. Decoupling it protects your data.

---

## 📄 Environment Pages (console)

| Page | Shows / used for |
|---|---|
| **Configuration** | Resources provisioned for the env; also lets you configure some of them |
| **Health** | Status & detailed health of the **EC2 instances** running your app |
| **Monitoring** | Statistics (e.g., **average latency, CPU utilization**); create **alarms** here |
| **Events** | Informational/error messages from services the env uses |
| **Tags** | Key-value pairs applied to env resources; manage them here |

---

## 📊 Monitoring

- **Monitoring console** — environment status + app health at a glance.
- EB reports a **web server environment's health** based on **how the app responds to the health check**.
- **Enhanced health reporting** *(enable it)* — gathers **additional info** about env resources for a **better, more detailed health picture** and faster issue identification (helps catch things before the app goes down).
- **Alarms** on metrics → spot and mitigate problems early.
- EC2 instances generate **logs** you can view to troubleshoot app/config issues.

> **Plain-English:** Basic health = "did the health check pass?". **Enhanced** health = "here's *why* it's red, with deeper instance/OS metrics."

---

## 🔐 Security — IAM Roles (know the two roles!)

When creating an environment, Beanstalk asks for **two IAM roles**:

| Role | Applied to | What it allows | Plain-English |
|---|---|---|---|
| **Service Role** (`AWSElasticBeanstalkService`) | Beanstalk service itself | Manage resources on your behalf — **Auto Scaling, ELB**, etc. | Lets **Beanstalk** drive AWS for you |
| **Instance Profile** | The **EC2 instances** in the env | Retrieve **app versions from S3**, **upload logs to S3**, often **X-Ray** and **Amazon SSM** permissions | Lets **your servers** reach the AWS services they need |
| **User Policies** | IAM users | Create & manage EB applications and environments | Lets **people** operate Beanstalk |

🪤 **Trap (mined from real Q):** Error *"The instance profile aws-elasticbeanstalk-ec2-role ... does not exist."* Common causes:
1. The **EB CLI couldn't create it** because your **IAM role lacks permission to create roles**.
2. The **IAM role exists but has insufficient permissions** that Beanstalk needs.
> Takeaway: instance-profile problems are about **missing role or missing permissions**, not the wrong platform.

---

## 💰 Pricing

- **No additional charge for Elastic Beanstalk itself.**
- You pay **only for the underlying AWS resources** your app consumes (EC2, ELB, RDS, S3, etc.).

> **Remember:** Beanstalk = **free orchestration layer**; the bill is just the resources beneath it.

---

## 🛠️ Platform Support (verbatim — gets tested)

| Category | Supported |
|---|---|
| **Languages** | Go, **Java (incl. Corretto)**, .NET, Node.js, PHP, Python, Ruby |
| **Web containers** | Tomcat, Passenger, Puma |
| **Docker** | Single Container **and** Multi-container |

---

## 📦 Where Do Files & Logs Go? (mined exam fact)

- **Application files** → stored in **Amazon S3**.
- **Server log files** → optionally stored in **S3** *or* **CloudWatch Logs**.

🪤 **Trap:** It's **S3 or CloudWatch Logs** — **not** CloudTrail, **not** Glacier-only, **not** EBS-only.

---

## ⚡ 30-Second Final Recall

- **What:** PaaS — upload code, AWS builds & manages EC2/ELB/Auto Scaling. **You still see/own the resources.**
- **Tier vs Type:** Tier = **Web** (HTTP) vs **Worker** (SQS queue). Type = **Single-Instance** vs **Load-Balanced+Auto Scaling**.
- **Deploy policies safest→fastest:** **Immutable → Rolling+additional batch → Rolling → All at once.** **Traffic splitting = canary** (% of traffic).
  - *Zero downtime + easy rollback* → **Immutable**. *Keep full capacity* → **Rolling with additional batch**. *Brief outage OK* → **All at once**.
- **Application** = folder. **Application Version** = labeled code in **S3**. **Environment** runs **one** version at a time.
- **Prod DB best practice:** provision **RDS outside** Beanstalk (else terminating env deletes the DB).
- **Two roles:** **Service Role** (Beanstalk manages ASG/ELB) + **Instance Profile** (EC2 → S3, logs, X-Ray, SSM).
- **Managed Platform Updates** = auto OS/runtime/web-server patching in a maintenance window.
- **Rebuild** terminated env within **6 weeks** (same name/ID/config).
- **Files in S3; logs in S3 or CloudWatch Logs.**
- **Domain:** `subdomain.region.elasticbeanstalk.com`.
- **Pricing:** Beanstalk is **free**; pay only for underlying resources.
- **Enhanced health reporting** = deeper health detail (enable it).
- **Shared ALB** can serve multiple EB environments in the **same VPC**.

---
*Source: Tutorials Dojo — AWS Elastic Beanstalk Cheat Sheet (last updated Nov 21, 2025). Service/role names kept exact for verbatim exam recall.*
