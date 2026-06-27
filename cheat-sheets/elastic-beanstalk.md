# AWS Elastic Beanstalk — Exam Cheat Sheet

> Quick-read summary for the SAA-C03 exam. Read top to bottom the night before.

---

## 1. What It Is (the one-liner)

**Elastic Beanstalk (EB) lets you deploy and run an app without managing the servers/infrastructure yourself.** You just upload your code; AWS handles the rest.

It automatically handles:
- **Capacity provisioning** (launching EC2 instances)
- **Load balancing** (distributing traffic)
- **Auto Scaling** (adding/removing instances based on load)
- **Health monitoring** (checking if your app is alive)

**Service model:** Platform-as-a-Service (**PaaS**).

> Easy way to remember: *You bring the code, EB brings the whole environment.*
>
> Trade-off vs. raw EC2: EB is faster and easier, but gives you less fine-grained control. Trade-off vs. fully serverless (Lambda): EB still runs actual EC2 servers under the hood — you just don't have to babysit them.

**Pricing:** EB itself is **free**. You only pay for the underlying resources it spins up (EC2, ELB, RDS, S3, etc.).

**Default domain format:** `subdomain.region.elasticbeanstalk.com`

---

## 2. Supported Platforms

| Category | What EB supports |
|---|---|
| **Languages** | Go, Java (incl. Corretto), .NET, Node.js, PHP, Python, Ruby |
| **Web containers** | Tomcat, Passenger, Puma |
| **Docker** | Single-container **and** Multi-container |

> A **Platform** in EB = OS + language runtime + web/app server + EB components, all bundled together.

---

## 3. Core Concepts (know these cold)

| Term | Plain-English meaning |
|---|---|
| **Application** | The top-level container. A logical "folder" holding all your environments, versions, and configs. |
| **Application Version** | One specific labeled build of your deployable code. It points to a code file stored in an **S3 object**. One app can have many versions; each version is unique. |
| **Environment** | An app version actually **deployed onto AWS resources**. Each environment runs **only one version at a time**, but you can run the same/different versions in many environments. |
| **Environment Tier** | Decides what kind of resources EB sets up — see table below. |
| **Environment Configuration** | The collection of settings/parameters that define how an environment behaves. |
| **Saved Configuration** | A reusable starting template for creating new environment configs. |
| **Platform** | OS + runtime + web server + EB components combined. |

### Environment Tiers (important distinction)

| Tier | When to use | What it runs |
|---|---|---|
| **Web Server** tier | App that responds to **HTTP requests** (a website/API) | Web server environment |
| **Worker** tier | App that **pulls tasks from a queue** (background jobs) | Worker environment that reads from an **Amazon SQS** queue |

> Mental model: **Web tier = answers visitors. Worker tier = chews through a to-do queue in the background.**

### Version Limits
There's a limit on how many application versions you can keep. To avoid hitting it, apply an **Application Version Lifecycle Policy** to auto-delete versions that are old, or once the total exceeds a set number.

---

## 4. Environment Types

| Type | Description |
|---|---|
| **Load-Balancing, Auto Scaling** | Automatically launches more instances as load increases. Best for production. |
| **Single-Instance** | One EC2 instance with an **Elastic IP**. No load balancer. Cheaper, good for dev/test. |

---

## 5. What's Inside an Environment

- **EC2 instances** configured to run your app on your chosen platform.
- **Auto Scaling group** — guarantees at least 1 instance is always running (single-instance), or scales within a min/max range (load-balanced).
- **Elastic Load Balancer (ELB)** — created automatically **when you enable load balancing**, to spread traffic across instances.
- **Amazon RDS integration** — EB can attach a database (MySQL, PostgreSQL, Oracle, or SQL Server). EB then auto-injects connection info (hostname, port, username, password, DB name) into your app as **environment properties**.
- **Environment Properties** — key/value settings used to pass secrets, endpoints, debug flags, etc. Let you run the same app across dev/test/staging/prod with different values.
- **Amazon SNS** — can be configured to notify you of important events.
- **Subdomain** — your environment gets a unique `*.elasticbeanstalk.com` subdomain.
- **Shared Application Load Balancer** — one ALB can serve multiple apps across multiple EB environments **within the same VPC**.
- **Environment Links** — named references that connect component environments together.
- **Rebuild window** — you can rebuild a **terminated** environment within **6 weeks** using the same name, ID, and config.
- **Managed Platform Updates** — EB can auto-apply OS / runtime / web-server patches during a scheduled **maintenance window**, keeping you secure with no manual work.

---

## 6. Deployment Policies (high exam value)

How EB rolls out a new version. Pick based on speed vs. safety vs. cost.

| Policy | How it works | Downtime? | Notes |
|---|---|---|---|
| **All at once** | Updates **every** instance at the same time | **Yes** — brief outage | Fastest, riskiest |
| **Rolling** | Updates instances in **batches** | No full outage, but reduced capacity during rollout | Some instances on old version while others update |
| **Rolling with additional batch** | Like rolling, but **first launches a new batch** of instances | No capacity loss | Keeps full capacity the whole time |
| **Immutable** | Deploys to a **brand-new set** of instances; swaps over if healthy | No | Safest; easy rollback (just keep old instances) |
| **Traffic splitting** | Deploys to a separate fleet, sends a **% of traffic** to it | No | Used for **canary deployments** — test new version on a slice of users before full rollout |

> Quick memory hook:
> - Need it fast & don't care about a blip → **All at once**
> - Save capacity → **Rolling with additional batch**
> - Safest / easy rollback → **Immutable**
> - Test on real users first → **Traffic splitting (canary)**

---

## 7. Monitoring

- **Monitoring console** — shows environment status and app health at a glance.
- **Health checks** — EB reports environment health based on how your app responds to the health check.
- **Enhanced Health Reporting** — opt-in feature; gathers **extra** info about environment resources for a fuller health picture and faster issue identification.
- **Alarms** — create alarms on metrics (latency, CPU, etc.) to catch problems early.
- **Logs** — EC2 instances generate logs you can view to troubleshoot.

### Where do files & logs go? (classic exam question)
- **Application files → always stored in Amazon S3.**
- **Server log files → optionally stored in S3 *or* CloudWatch Logs.**
  - You can configure EB to copy logs to S3 every hour, or stream them live to **CloudWatch Logs**.
  - ❌ Not directly to Glacier (only via an S3 lifecycle policy). ❌ Not CloudTrail (that's for API auditing).

### Environment Pages (what each tab shows)

| Page | Shows |
|---|---|
| **Configuration** | Resources provisioned; lets you tweak some of them |
| **Health** | Status + detailed health of the EC2 instances |
| **Monitoring** | Stats like average latency & CPU utilization; create alarms here |
| **Events** | Info / error messages from services the environment uses |
| **Tags** | Key-value tags applied to environment resources; manage them here |

---

## 8. Security — IAM Roles (know the two roles)

When you create an environment, EB asks for **two IAM roles**:

| Role | Applied to | Purpose |
|---|---|---|
| **Service Role** (`AWSElasticBeanstalkService`) | EB the service | Lets Beanstalk manage resources **on your behalf** (Auto Scaling, ELB, etc.) |
| **Instance Profile** (`aws-elasticbeanstalk-ec2-role`) | The EC2 **instances** | Lets instances pull app versions from S3, upload logs to S3, and often use **X-Ray** and **SSM** |

Plus:
- **User Policies** — let *users* create and manage EB apps and environments.

> Common error: *"The instance profile ... does not exist."* → usually because your IAM user lacks permission to **create roles**, or the role exists but has **insufficient/outdated permissions**.

---

## 9. Best-Practice Notes (don't skip — these show up)

- **Database placement:** You *can* let EB provision an RDS database inside the environment, BUT for **production**, provision RDS **separately (outside EB)** and connect via a connection string. Why? If the EB environment is terminated, an internal DB gets **deleted with it** — an external DB survives.
- **Managed Platform Updates** keep your OS/runtime patched automatically during a maintenance window.
- **Shared ALB** saves money by serving multiple EB environments in one VPC.

---

## 10. 30-Second Final Recall

- EB = **PaaS**, upload code → AWS runs it. **EB is free, pay for resources.**
- Two environment **tiers**: **Web** (HTTP) vs **Worker** (SQS queue).
- Two environment **types**: **Single-instance** (Elastic IP) vs **Load-balanced (Auto Scaling)**.
- **App files always in S3**; **logs in S3 or CloudWatch Logs**.
- Deployment policies safest→fastest: **Immutable / Traffic-splitting → Rolling (w/ batch) → Rolling → All at once.**
- Two IAM roles: **Service Role** (EB acts for you) + **Instance Profile** (EC2 instances).
- **Production DB → keep it outside EB** so it isn't deleted on termination.
- Rebuild a terminated environment within **6 weeks**.
