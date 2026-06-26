# ⚡ AWS Lambda — SAA-C03 Cheat Sheet
> Verified against official AWS docs (docs.aws.amazon.com/lambda) — June 2026

---

## 🔑 What is Lambda?
- **Serverless, event-driven** compute — runs code in response to events, no server management
- You pay only for **requests + execution duration** (billed per 1ms)
- Functions are **stateless** — no affinity to any underlying server
- Scales **horizontally and automatically** — spins up as many instances as needed
- Code deployed as **ZIP packages** (up to 250 MB unzipped) or **container images** (up to 10 GB)

---

## 🧩 Key Components

| Component | What it does |
|-----------|-------------|
| **Function** | Your code + config — processes an event and returns a response |
| **Execution Environment** | Secure, isolated Firecracker micro-VM where your function runs |
| **Runtime** | Sits between Lambda service and your code — relays events and responses |
| **Environment Variables** | Key-value config (DB strings, API keys). Always **encrypted at rest**, optionally in transit |
| **Layers** | ZIP archives with shared libraries/runtimes — keeps deployment packages small |
| **Event Source** | AWS service or custom app that triggers the function |
| **Log Streams** | Auto-sent to CloudWatch; add custom logging with annotations |

---

## 💻 Supported Runtimes
Node.js, Python, Java, C# (.NET), Go, Ruby, PowerShell

> **Custom runtimes** also supported via the Lambda Runtime API (bring any language)

---

## ⚙️ Function Configuration Limits

| Setting | Limit |
|---------|-------|
| Memory | 128 MB – 10,240 MB (10 GB) |
| Ephemeral storage (/tmp) | 512 MB – 10,240 MB (10 GB) |
| Execution timeout | 15 minutes max |
| Deployment package (ZIP, unzipped) | 250 MB |
| Container image | 10 GB |
| Sync payload limit | 6 MB |
| Async payload limit | 256 KB |

- CPU, network, and I/O scale **proportionally to memory** — more memory = more CPU
- **Versions**: snapshot of function state, appended as `:version-number` to ARN
- **Aliases**: human-readable pointer to a version (e.g., `MyAlias`)

---

## 📞 Invocation Types

### Synchronous (Lambda waits and returns result)
Used when caller needs the response immediately.

Common triggers:
- Amazon API Gateway
- Application Load Balancer (ALB)
- Amazon Cognito
- Amazon CloudFront (Lambda@Edge)
- Amazon Data Firehose

### Asynchronous (Lambda queues the event, returns 202 immediately)
Used for background/long-latency work. Lambda retries on failure.
- Returns **202 Accepted** — does NOT mean the function succeeded
- Max payload: **256 KB**
- Used for: batch jobs, video encoding, order processing

Common triggers:
- Amazon S3
- Amazon SNS
- Amazon EventBridge
- Amazon CloudWatch Logs
- AWS CloudFormation, Config, CodeCommit

---

## 🔗 Event Source Mapping
Reads from a **queue or stream** and synchronously invokes Lambda. Lambda polls the source for you.

Function is invoked when:
- Batch size is reached
- Max batching window elapsed
- Total payload hits 6 MB

Supported sources:
- Amazon Kinesis Data Streams
- Amazon DynamoDB Streams
- Amazon SQS
- Amazon MQ
- Amazon MSK (Managed Streaming for Kafka)
- Self-managed Apache Kafka

> 💡 You can apply **event filtering** to only process relevant events — saves cost

---

## 🔄 Concurrency

| Type | What it does |
|------|-------------|
| **Reserved Concurrency** | Guarantees a minimum level for a function; caps its maximum — no other function can use this allocation |
| **Provisioned Concurrency** | Pre-initialises execution environments so there's zero cold start latency on invocation |

- Default account concurrency limit: **1,000 concurrent executions** (can be increased via support)
- Scaling rate: **1,000 new execution environments per 10 seconds** *(doubled from 500 in 2025)*

---

## 🌐 Lambda Function URL
- Creates a **dedicated HTTPS endpoint** for your function — no API Gateway needed
- Format: `https://<url-id>.lambda-url.<region>.on.aws`
- Static — doesn't change once created
- **Dual-stack** (IPv4 + IPv6)
- Can be invoked via browser, CURL, Postman, or any HTTP client

**Authentication types:**

| Type | Who can invoke |
|------|---------------|
| `AWS_IAM` | Only IAM users/roles with explicit permission |
| `NONE` | Anyone with the URL (public) |

- Can be applied to any **alias** or **$LATEST** version — NOT to other specific versions
- Accessible via **public internet only** — NOT via VPC Endpoints (PrivateLink)
- Supports CORS configuration to whitelist allowed origins

---

## 🔒 Lambda in a VPC
- By default Lambda runs in AWS-managed infrastructure (outside your VPC)
- You can enable VPC access by providing **subnet IDs** and **security group IDs**
- Lambda creates **Elastic Network Interfaces (ENIs)** to connect securely to private resources (RDS, EC2, etc.)
- Can mount **Amazon EFS** for shared persistent storage across functions

---

## 🌍 Lambda@Edge
- Runs Lambda functions **at CloudFront edge locations** — closer to the user
- Triggered at 4 CloudFront event points:

| Event | When it fires |
|-------|--------------|
| Viewer Request | After CloudFront receives request from viewer |
| Origin Request | Before CloudFront forwards request to origin |
| Origin Response | After CloudFront receives response from origin |
| Viewer Response | Before CloudFront forwards response to viewer |

> Use cases: A/B testing, URL rewrites, auth at the edge, HTTP header manipulation

---

## ⚡ Lambda SnapStart
- Speeds up **cold starts** by taking a snapshot of the initialized execution environment
- Restores from snapshot instead of re-initializing — much faster startup
- Originally Java only; **now available for Python (python3.12+) and .NET (dotnet8+)** *(expanded Nov 2024)*

---

## 💰 Pricing
- **Number of requests**: first 1M free/month, then $0.20 per 1M
- **Duration**: billed per 1ms of execution; first 400,000 GB-seconds free/month
- **⚠️ NEW (Aug 2025)**: The **INIT (cold start) phase is now billed** at the same rate as execution time — important cost consideration for Java/.NET workloads with heavy initialization
- Additional charges: Provisioned Concurrency, SnapStart, ECR image storage, VPC data transfer

---

## 📦 Deploying with External Dependencies
1. Place all external libraries **locally in your project folder**
2. **ZIP** the entire package (code + dependencies)
3. Upload to Lambda directly (console) or via **S3** (for large packages)
4. Alternatively, use **Lambda Layers** to separate shared dependencies

---

## 🆕 New Features & Updates (Not in Tutorial Dojo / Post-publication)

| Feature | Detail | Date |
|---------|--------|------|
| **SnapStart for Python & .NET** | Now supports `python3.12` and `dotnet8` — Tutorial Dojo only mentioned Java | Nov 2024 |
| **Node.js 22.x runtime** | New managed runtime `nodejs22.x` now available | Nov 2024 |
| **Python 3.13 runtime** | New managed runtime now available | Nov 2024 |
| **KMS encryption for .zip packages** | Customer-managed key (CMK) encryption for deployment packages | Nov 2024 |
| **Lambda Managed Instances** | Run Lambda on dedicated EC2 hardware (incl. GPUs, Graviton4) — AWS manages OS, patching, scaling. Supports **multiconcurrency** (multiple requests per instance). Pay for EC2 uptime + ~15% mgmt fee, no per-ms billing | Nov 2025 |
| **Lambda Durable Functions** | Build **stateful workflows** that persist state for up to **1 year** — like Step Functions but natively in Lambda | Dec 2025 |
| **Scaling rate doubled** | Now 1,000 new environments/10s (was 500) | 2025 |
| **INIT phase now billed** | Cold start initialization time billed at execution rates | Aug 2025 |
| **SQS Provisioned Mode ESM** | Pre-scale Lambda consumers before burst traffic hits | 2025 |
| **Amazon Linux 2 End of Life** | AL2 runtimes (python3.11, java17, nodejs16.x) must migrate to AL2023 by **June 30, 2026** | — |

---

## 🔁 Quick Recall: Sync vs Async vs Event Source Mapping

| | Sync | Async | Event Source Mapping |
|--|------|-------|---------------------|
| Caller waits? | ✅ Yes | ❌ No (202) | ❌ No (Lambda polls) |
| Retries? | No (caller handles) | Yes (Lambda retries) | Yes |
| Sources | API GW, ALB, Cognito | S3, SNS, EventBridge | SQS, Kinesis, DynamoDB |
| Payload limit | 6 MB | 256 KB | 6 MB (batch) |

---

## 🚨 Common Exam Traps
- Lambda@Edge ≠ Lambda in a Region — runs **at CloudFront edge**, not in your VPC
- Function URL auth type `NONE` = **publicly accessible** — not "unauthenticated IAM"
- Reserved concurrency **caps** the function — it's a ceiling AND a guarantee
- SnapStart is NOT the same as Provisioned Concurrency — SnapStart snaps init state; PC keeps warm environments alive
- Async invocation returns 202 — this is a **queue confirmation**, NOT a success confirmation
- Lambda can mount **EFS** (not EBS) — EBS is block storage for EC2, not Lambda
