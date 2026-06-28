# ⚡ AWS Lambda — Cheat Sheet (SAA-C03)

> **What it is:** A serverless, event-driven compute service that runs your code in response to events and fully manages the servers for you — you upload code, it runs and scales automatically, and you pay only while it runs.

---

## 1. The Big Picture (memorize these first)

| Trait | Detail | Plain-English meaning |
|-------|--------|-----------------------|
| **Serverless** | No servers to provision/patch | You only manage code; AWS owns the machine |
| **Event-driven** | Runs in response to an event | Something happens → your function fires |
| **Stateless** | No affinity to any server | Don't store data in the function; it forgets between runs |
| **Auto-scales horizontally** | Spins up as many instances as needed, from zero | Traffic spike = more copies automatically, no config |
| **Pay-per-use** | Billed per request + per **1 ms** of duration | No idle cost — quiet function = ~$0 |
| **Runs on** | Firecracker **microVMs** | Tiny, fast, secure mini-VMs that boot in ms |
| **Compliance** | SOC, HIPAA, PCI, ISO | Safe for regulated workloads |

**Easy way to remember Lambda's identity:** *"Code that wakes up on an event, does one job, and goes back to sleep — and you're billed by the millisecond."*

---

## 2. Hard Limits 🔒 (classic exam fodder — know the exact numbers)

| Limit | Value | Trap / note |
|-------|-------|-------------|
| **Timeout (max)** | **15 minutes** | #1 trap: long jobs (>15 min) ❌ need Fargate/Batch/EC2, not Lambda |
| **Memory** | **128 MB – 10 GB** | CPU, network, I/O scale *with* memory (more memory = more CPU) |
| **/tmp ephemeral storage** | **512 MB – 10 GB** | Temporary scratch disk, wiped between cold starts |
| **Deployment package (ZIP)** | 50 MB zipped (direct), 250 MB unzipped | Too big? → use **Layers** or **S3** |
| **Deployment package (container image)** | up to **10 GB** | Use ECR-hosted image for big dependencies |
| **Async payload** | up to **256 KB** | Asynchronous invoke event size cap |
| **Event source mapping batch payload** | **6 MB** | One of the triggers that flushes a batch (below) |

> **Memory ↔ CPU link is a favorite trick:** You can't set CPU directly. To make a function faster, **raise the memory** — CPU rises proportionally.

**Supported languages (natively):** Node.js, Python, Java, C#/.NET, Go, Ruby, PowerShell.
**Architectures:** x86 and **Graviton2** (ARM, cheaper/efficient).

---

## 3. Invocation Types — Synchronous vs Asynchronous ⭐ (HIGH-VALUE, heavily tested)

This is the single most testable Lambda concept. Memorize which service uses which.

| | **Synchronous** | **Asynchronous** |
|---|---|---|
| **Behavior** | Caller waits for the result | Event dropped in an internal queue; returns immediately |
| **Response** | Returns the function's result | Returns **202 Accepted** right away |
| **What 202 means** | n/a | "I queued it" — does **NOT** mean it succeeded |
| **Use case** | Real-time request/response | Background work: batch, video encoding, order processing |
| **Payload cap** | — | **256 KB** |

**Who invokes Lambda SYNCHRONOUSLY** (caller waits):
- Amazon **API Gateway**
- **Application Load Balancer (ALB)**
- Amazon **Cognito**
- Amazon **Data Firehose**
- Amazon **CloudFront (Lambda@Edge)**

**Who invokes Lambda ASYNCHRONOUSLY** (fire-and-forget):
- Amazon **S3**
- Amazon **CloudWatch Logs**
- Amazon **EventBridge**
- **AWS Config**, **CloudFormation**, **CodeCommit**
- API Gateway *can* be async by setting `Event` in the `X-Amz-Invocation-Type` header (non-proxy integration)

> **Memory hook:** *"If a human/client is waiting for an answer → synchronous (API GW, ALB, CloudFront). If a system event just happened and nobody's waiting → asynchronous (S3, EventBridge, CloudWatch Logs)."*

> 🪤 **Trap:** API Gateway is the tricky one — it's **synchronous by default** but can be made **asynchronous** via the invocation-type header. If a question mentions that header, the answer flips to async.

---

## 4. Event Source Mapping (polling-based triggers) ⭐

**Plain English:** A built-in poller that *reads from a queue/stream for you* and then invokes your function synchronously with a batch of records. You don't write the polling loop.

- Used for **stream/queue** sources (not S3/EventBridge — those push directly).
- Supports **event filtering** → process only relevant events = fewer invocations = **saves money**.
- A batch is sent to your function when **any one** of these is hit:
  - **Batch size** reached, **OR**
  - **Maximum batching window** reached, **OR**
  - **Total payload = 6 MB**

**Services that use Event Source Mapping:**
- Amazon **Kinesis**
- Amazon **DynamoDB** (Streams)
- Amazon **SQS**
- Amazon **MQ**
- Amazon **MSK** (Managed Kafka) + self-managed Apache Kafka

> **Easy way to remember:** *Event Source Mapping = "Lambda pulls" (streams/queues). Direct triggers like S3/EventBridge = "they push."*

---

## 5. Concurrency Management ⭐ (commonly confused pair)

**Concurrency** = number of function instances running *at the same time*. Each in-flight request = 1 concurrency unit.

| Type | What it does | When to pick | Analogy |
|------|--------------|--------------|---------|
| **Reserved Concurrency** | **Guarantees** a function a slice of concurrency **and caps** its max | Protect a critical function, or stop one function from starving others | "Reserved parking spots — always yours, but you can't exceed them" |
| **Provisioned Concurrency** | Pre-initializes instances so they're **warm** and ready | Need **consistent low latency**, avoid cold starts (e.g. predictable spikes) | "Engines kept running so there's no startup delay" |

> 🪤 **Trap — don't mix these up:**
> - **Reserved** = a *quota* (guarantee + ceiling), about **capacity allocation**. Free.
> - **Provisioned** = pre-**warmed** instances, about **killing cold-start latency**. **Costs extra.**
> - If the question says *"reduce/eliminate cold start latency"* → **Provisioned Concurrency** (or SnapStart for Java/Python/.NET).
> - If it says *"ensure a function always has capacity"* / *"limit a function's max concurrency"* → **Reserved Concurrency.**

---

## 6. Components of a Lambda Application

| Component | What it is | Plain English |
|-----------|-----------|----------------|
| **Function** | Your code that runs | The actual program |
| **Execution environment** | Secure isolated microVM | The sandbox your code runs in |
| **Runtime** | Language layer between Lambda & code | The translator that relays events/responses |
| **Environment variables** | Key-value config (DB strings, API keys) | Settings injected at runtime; **always encrypted at rest** |
| **Layers** | ZIP of libs/custom runtime/deps | Shared "add-on packs" to keep deployment package small |
| **Event source** | Service that triggers the function | The thing that pushes the "go" button |
| **Downstream resources** | Services the function calls | What the function talks to after it fires |
| **Log streams** | Custom logging into CloudWatch | Your own print statements for debugging |

### Versions & Aliases (know the ARN formats — tested verbatim)
- **Version** = immutable snapshot of the function at a point in time. Publishing appends a number:
  `arn:aws:lambda:us-east-2:123456789123:function:my-function:1`
- **Alias** = a friendly **pointer** to a version (e.g. `prod`, `dev`). Lets you shift traffic without changing callers:
  `arn:aws:lambda:us-east-2:123456789123:function:my-function:MyAlias`

> **Analogy:** *Version = a saved git commit (frozen). Alias = a branch name/pointer you can move to a different commit.*

### Layers vs big packages — deploying external dependencies
If your code needs external libs/SDKs:
1. Put dependencies in your app folder → 2. ZIP it → 3. Upload to Lambda (directly or via **S3**).
**Better practice:** move heavy/unchanging deps into a **Layer** so your function package stays small and modular.

---

## 7. Lambda Function URL

**Plain English:** A built-in **HTTPS endpoint** for your function — call it directly over the web, **no API Gateway needed**.

- Format: `https://<url-id>.lambda-url.<region>.on.aws`
- **Static** — doesn't change once created.
- **Dual-stack** — supports IPv4 **and** IPv6.
- Invoke via browser, curl, Postman, any HTTP client.
- **Public internet only** — ❌ cannot be reached via PrivateLink/VPC endpoints.
- Uses **resource-based policies**; supports **CORS** to whitelist origins.
- Applies only to a function **alias** or the **`$LATEST`** unpublished version — **not** to other specific versions.

**Two auth types:**
| Auth type | Meaning |
|-----------|---------|
| **AWS_IAM** | Only IAM users/roles with `lambda:InvokeFunctionUrl` permission can call it |
| **NONE** | Anyone with the URL can invoke it (public, no AWS account needed) |

> 🪤 **Trap:** Function URL = quick public HTTPS. If a question needs **throttling, API keys, request validation, usage plans, or private access** → that's **API Gateway**, not a Function URL.

---

## 8. Lambda in a VPC

- By default Lambda runs in an AWS-managed VPC (secure, but no access to your private resources).
- To reach private resources (RDS, EC2, ElastiCache), give Lambda **VPC subnet IDs + security group IDs**.
- Lambda then creates **Elastic Network Interfaces (ENIs)** to connect into your VPC securely.

> **Easy way to remember:** *"Lambda needs subnet + security group to step inside your VPC; it builds an ENI as its doorway."*

---

## 9. Lambda@Edge ⭐ (vs SnapStart vs Managed Instances — don't confuse)

**Plain English:** Run Lambda functions **at CloudFront edge locations**, close to the viewer, to customize content delivery — no servers to manage.

Four CloudFront trigger points (memorize the order):
1. **Viewer request** — after CloudFront gets the request from the user
2. **Origin request** — before CloudFront forwards to the origin
3. **Origin response** — after CloudFront gets the origin's response
4. **Viewer response** — before CloudFront sends back to the user

> **Hook:** *Viewer = nearest the user; Origin = nearest the backend. Request goes in (Viewer→Origin), Response comes out (Origin→Viewer).*

---

## 10. Lambda SnapStart

**Plain English:** Speeds up cold starts by booting once, taking a **snapshot** of the initialized environment, then **resuming from that snapshot** for future invocations — no extra resources needed.

- Originally **Java**; source notes support for **Java, Python, and .NET**.
- Reduces cold-start latency **without** paying for Provisioned Concurrency.

> 🪤 **Confusion alert — three cold-start fixes:**
> - **Provisioned Concurrency** = keep instances warm (any runtime, costs $).
> - **SnapStart** = resume from a snapshot (Java/Python/.NET, no provisioning).
> - **Managed Instances** = pre-provisioned EC2 stays active (see below).

---

## 11. Lambda Managed Instances 🆕 (likely too new for the exam — bonus)

**Plain English:** Run Lambda functions on **specific EC2 instances** (incl. **GPUs**, **Graviton4**) that AWS still manages for you — bridges serverless simplicity with EC2 hardware control.

- **Capacity Provider** = you define VPC config, EC2 instance types, scaling policies.
- **Multiconcurrency** = one instance handles **multiple requests at once** (vs standard Lambda's one-request-per-environment).
- AWS handles lifecycle, OS patching, load balancing, auto-scaling; you pick the hardware.
- **Why:** access GPUs (AI/ML inference), high-memory; **no cold starts** (instances stay active); good for **predictable, high-volume** workloads.
- **Pricing:** pay EC2 rates (Savings Plans/RIs eligible) + ~**15% management fee** + standard request charges ($0.20 / 1M). **No per-ms duration charge** — you pay for instance uptime.

---

## 12. Pricing & Free Tier 💰

**You're billed on:**
- **Number of requests**
- **Compute duration** (billed in **1 ms** increments)
- **Memory configuration × duration** (GB-seconds)

**Free tier (per month):**
- **1 million** free requests
- **400,000 GB-seconds** of compute

**Extra charges apply to:** Provisioned Concurrency, SnapStart, ECR storage for container images, VPC networking data transfer.

> **Mental model:** *Cost ≈ (how often it runs) × (how long each run takes) × (how much memory you gave it).*

---

## 13. Lambda vs Other Compute (common exam tie-breakers)

| If the scenario says... | Pick |
|--------------------------|------|
| Short event-driven code, <15 min, scale to zero | **Lambda** |
| Job runs **longer than 15 min** | **Fargate / ECS / EC2 / Batch** |
| Long-running containers, full control | **ECS/EKS on EC2 or Fargate** |
| Public HTTPS endpoint, no throttling/keys needed | **Lambda Function URL** |
| Need throttling, API keys, usage plans, caching | **API Gateway + Lambda** |
| Customize content at the edge | **Lambda@Edge (CloudFront)** |
| Eliminate cold starts (any language, $$) | **Provisioned Concurrency** |
| Reduce cold starts (Java/Python/.NET, free) | **SnapStart** |
| GPU / heavy ML inference on Lambda | **Managed Instances** |

---

## 🕒 30-Second Final Recall

- **Lambda = serverless, event-driven, stateless, auto-scaling; pay per request + per 1 ms.**
- **Max timeout = 15 min.** Memory **128 MB–10 GB** (CPU scales with memory). /tmp **512 MB–10 GB**. Container image up to **10 GB**.
- **Sync** = caller waits (**API GW, ALB, Cognito, Firehose, CloudFront/Lambda@Edge**). **Async** = 202 immediately, payload ≤ **256 KB** (**S3, EventBridge, CloudWatch Logs, Config, CFN**).
- **Event Source Mapping** = Lambda *polls* streams/queues (**Kinesis, DynamoDB, SQS, MQ, MSK/Kafka**); batch flushes at size **OR** window **OR** **6 MB**.
- **Reserved Concurrency** = guarantee + cap (capacity). **Provisioned Concurrency** = warm instances (kill cold starts, costs extra).
- **SnapStart** = snapshot-resume cold-start fix for **Java/Python/.NET**, free.
- **Function URL** = direct public HTTPS, auth **AWS_IAM** or **NONE**; public internet only (no PrivateLink). Need keys/throttling → **API Gateway**.
- **VPC access** needs **subnet IDs + security group IDs** → Lambda makes **ENIs**.
- **Lambda@Edge** = 4 hooks: viewer request → origin request → origin response → viewer response.
- **Versions** = immutable snapshots; **Aliases** = movable pointers to versions.
- **Free tier:** 1M requests + 400,000 GB-seconds/month.
