============================================================================================================================
can you please explain below aws compoent in deep 

EC2, S3, IAM, VPC
Lambda
ECS/EKS
ECR
API Gateway
Route 53
ALB/NLB
CloudWatch and CloudTrail
Secrets Manager and KMS
EventBridge, SNS and SQS
Step Function
============================================================================================================================

# AWS Core Services: In-Depth Guide

## 1. EC2, S3, IAM, VPC

### EC2 (Elastic Compute Cloud)
Virtual machines running on the Nitro hypervisor.

**Key concepts**
- **Instance families:** General purpose (M, T), compute optimized (C), memory optimized (R, X), storage optimized (I, D), accelerated (P, G, Inf, Trn). The "g" suffix means Graviton (ARM), which is typically cheaper per unit of performance.
- **Burstable (T-series):** CPU credit model, good for low-average-load workloads.
- **AMI:** A template (root volume snapshot, launch permissions, block device mapping). Build golden AMIs with EC2 Image Builder or Packer.
- **Launch templates:** Versioned definitions of AMI, instance type, SGs, user data, and IAM profile. Auto Scaling groups use them (launch configurations are deprecated).
- **User data:** A bootstrap script that runs at first boot.

**Storage**
- **EBS:** Network-attached block storage, AZ-scoped. Types: gp3 (default, baseline IOPS and throughput configurable independently), io2 (high IOPS databases), st1/sc1 (HDD throughput/cold). Snapshots are incremental and stored in S3 (managed by AWS).
- **Instance store:** Physically attached, very fast, but ephemeral (lost on stop or terminate).
- **EFS/FSx:** Shared file systems.

**Purchasing options**
- On-Demand, Savings Plans (Compute or EC2 Instance), Reserved Instances, Spot (up to ~90% discount, 2-minute interruption notice), Dedicated Hosts/Instances (licensing and compliance), Capacity Reservations.

**Networking and placement**
- ENI (private IP, SGs, MAC), Elastic IP, Enhanced Networking (ENA), EFA for HPC.
- **Placement groups:** Cluster (low latency, same rack), Spread (max 7 per AZ, isolated hardware), Partition (large distributed systems such as Kafka and HDFS).

**Security and operations**
- Use **IMDSv2** (token-based metadata) to mitigate SSRF attacks. Instance profiles supply role credentials. Prefer **SSM Session Manager** over SSH and bastions.
- **Auto Scaling:** Target tracking, step, scheduled, and predictive scaling. Lifecycle hooks, warm pools, and instance refresh for rolling AMI updates.

**Architect's view:** Use immutable infrastructure (bake AMIs, replace instead of patching), spread across AZs, mix On-Demand and Spot in ASGs, and manage everything through IaC.

---

### S3 (Simple Storage Service)
Object storage with 11 nines of durability. It is **strongly consistent** for reads, writes, and lists since December 2020.

**Core model:** Buckets are global-namespace and regional. Objects (up to 5 TB) are addressed by key. The "folder" structure is just key prefixes. Request rates scale per prefix (3,500 writes and 5,500 reads per second per prefix).

**Storage classes**
| Class | Use |
|---|---|
| Standard | Frequent access |
| Intelligent-Tiering | Unknown or changing patterns (automatic tiering) |
| Standard-IA / One Zone-IA | Infrequent access |
| Glacier Instant / Flexible / Deep Archive | Archive, with retrieval times from milliseconds to hours |

**Data management**
- **Versioning:** Protects against overwrites and deletes (delete markers). Combine with MFA Delete for critical data.
- **Lifecycle rules:** Transition and expire objects, clean up incomplete multipart uploads and old versions.
- **Replication:** CRR/SRR, requires versioning, replicates only new objects by default (Batch Replication for existing ones). Replication Time Control offers a 15-minute SLA.
- **Object Lock:** WORM (governance or compliance mode) for regulatory retention.
- **Event notifications:** To Lambda, SQS, SNS, or EventBridge (EventBridge is the most flexible).

**Security**
- **Block Public Access** (account and bucket level), **bucket policies**, IAM policies, **ACLs** (disabled by default now via Object Ownership "bucket owner enforced"), **Access Points**, and VPC endpoint policies.
- **Encryption:** SSE-S3 (default on for all new objects), SSE-KMS (audit trail and key control, watch for KMS request costs; use S3 Bucket Keys), SSE-C, and client-side. Enforce TLS with a `aws:SecureTransport` deny.
- **Presigned URLs** for temporary access.

**Performance:** Multipart upload (recommended above 100 MB), Transfer Acceleration, byte-range fetches, S3 Select/Athena for querying in place.

**Common interview trap:** "Why can't I delete a bucket?" Check versions, delete markers, and Object Lock.

---

### IAM (Identity and Access Management)
The global authorization layer for everything in AWS.

**Identities and policies**
- **Users, groups, roles.** Roles provide temporary credentials through **STS** (`AssumeRole`, `AssumeRoleWithWebIdentity`, `AssumeRoleWithSAML`).
- **Policy types:** Identity-based, resource-based (S3, SQS, KMS, Lambda), permission boundaries, SCPs, session policies, and VPC endpoint policies.
- **Role anatomy:** A **trust policy** (who can assume) and **permission policies** (what they can do).

**Policy evaluation logic (a favorite question)**
1. Explicit **Deny** always wins.
2. Check SCPs (must allow), then resource policies, permission boundaries, and session policies.
3. Otherwise, an explicit **Allow** is needed. Default is implicit deny.

Same-account resource policies can grant access without an identity policy allow. Cross-account access needs both sides to allow.

**Advanced features**
- **Condition keys:** `aws:SourceVpce`, `aws:PrincipalOrgID`, `aws:MultiFactorAuthPresent`, `aws:RequestedRegion`, and tags.
- **ABAC:** Access via tags on principals and resources, which scales better than per-resource policies.
- **IAM Identity Center:** Centralized workforce SSO with permission sets across accounts.
- **Federation for workloads:** OIDC for GitHub Actions and EKS (IRSA, Pod Identity), with no static keys.
- **IAM Access Analyzer:** Finds external and unused access, validates policies, and generates least-privilege policies from CloudTrail activity.
- **Credential report, last-accessed data** for cleanup.

**Architect's view:** No IAM users for humans, no long-lived keys, SCPs as guardrails, permission boundaries for delegated administration, and least privilege iterated from actual usage.

---

### VPC (Virtual Private Cloud)
Your logically isolated network, regional in scope, with subnets bound to a single AZ.

**Building blocks**
- **CIDR:** From /16 to /28. Plan non-overlapping ranges up front, since overlap breaks peering and TGW routing. Secondary CIDRs and IPv6 are supported.
- **Subnets:** Public (route to an Internet Gateway), private (route via NAT), isolated (no internet route). AWS reserves 5 IPs per subnet.
- **Route tables:** Longest-prefix match decides routing. The "local" route is implicit.
- **Internet Gateway** (horizontally scaled, no bandwidth limit), **NAT Gateway** (AZ-scoped, so deploy one per AZ for resilience, and watch data processing costs), and an **Egress-only IGW** for IPv6.
- **Security groups** (stateful, allow-only, ENI-level) and **NACLs** (stateless, ordered rules, subnet-level).

**Connectivity**
- **Gateway endpoints** (S3, DynamoDB; free, route table entries) vs **Interface endpoints** (PrivateLink ENIs; hourly and data cost). Endpoint policies restrict what can be accessed.
- **VPC Peering** (non-transitive), **Transit Gateway** (hub, transitive, route table segmentation), **PrivateLink** (service exposure, handles overlapping CIDRs).
- **Site-to-Site VPN**, **Direct Connect** (dedicated, private, consistent latency; use a VPN backup or a second DX).
- **Observability:** VPC Flow Logs, Traffic Mirroring, Reachability Analyzer.

**Typical 3-tier layout:** Public subnets (ALB, NAT), private app subnets, and isolated data subnets across 3 AZs.

---

## 2. Lambda

Serverless compute that runs code in response to events, billed per request and per GB-second (1 ms granularity).

**Execution model**
- Code runs in a Firecracker microVM execution environment. The first invocation incurs a **cold start** (download code, start runtime, run init code). Later invocations reuse the warm environment, so initialize SDK clients and connections **outside the handler**.
- **Limits:** 15-minute timeout, 128 MB to 10 GB memory (CPU scales proportionally with memory), 10 GB ephemeral `/tmp`, 6 MB synchronous payload, 256 KB asynchronous payload, 250 MB unzipped package (or 10 GB container image).

**Invocation types**
| Type | Examples | Retry behavior |
|---|---|---|
| Synchronous | API Gateway, ALB, SDK | Caller handles errors |
| Asynchronous | S3, SNS, EventBridge | 2 automatic retries, then DLQ or on-failure destination |
| Poll-based (event source mapping) | SQS, Kinesis, DynamoDB Streams | Lambda polls. For SQS, the message returns to the queue on failure |

**Concurrency**
- Account limit (default 1,000 per region, raisable). **Reserved concurrency** guarantees and caps a function. **Provisioned concurrency** pre-warms environments to eliminate cold starts, at a cost.
- If a Lambda reading SQS gets throttled, messages return after the visibility timeout. Set the queue visibility timeout to at least 6x the function timeout.
- Use **partial batch responses** (`ReportBatchItemFailures`) for SQS and streams so a single bad record doesn't retry the whole batch.

**Features**
- **Versions and aliases** (weighted aliases enable canary deployments with CodeDeploy), **layers**, **environment variables** (KMS-encrypted), **extensions**, **Function URLs**.
- **SnapStart** (Java, Python, .NET) reduces cold starts through snapshot restore.
- **VPC access:** Uses Hyperplane ENIs. A VPC Lambda needs a NAT or VPC endpoints to reach AWS APIs.
- **Destinations** for async success and failure routing.
- **Power Tuning:** Test memory settings, since more memory can mean lower total cost.

**Best practices:** Idempotent handlers (events can be delivered more than once), small packages, least-privilege execution role, structured logging with Powertools for AWS Lambda, and avoiding Lambda for long-running or constant high-throughput work where containers cost less.

---

## 3. ECS / EKS

### ECS (Elastic Container Service)
AWS-native container orchestrator with no control plane to manage.

- **Task definition:** The blueprint (images, CPU/memory, ports, env, logging, volumes, roles).
- **Task:** A running instantiation. **Service:** Maintains the desired count, integrates with ALB/NLB, and does deployments.
- **Cluster:** A logical grouping of capacity.
- **Launch types:** **Fargate** (serverless, per-task VM isolation, pay for vCPU/memory), **EC2** (you manage instances; better for GPU, cost at scale, and daemon-style needs), and **ECS Anywhere** for on-prem.
- **Two IAM roles (a classic question):** The **task execution role** lets the ECS agent pull from ECR, write logs, and fetch secrets. The **task role** is what your application code uses to call AWS APIs.
- **Networking:** `awsvpc` mode gives each task its own ENI and security group. **Service Connect** and Cloud Map handle service discovery.
- **Deployments:** Rolling (default) or blue/green via CodeDeploy, with deployment circuit breaker and automatic rollback.
- **Scaling:** Application Auto Scaling (target tracking on CPU, memory, or ALB requests per target) and capacity providers.

### EKS (Elastic Kubernetes Service)
Managed Kubernetes control plane (multi-AZ API servers and etcd), where you run the data plane.

**Compute options**
- **Managed node groups** (AWS handles AMI updates and draining), **self-managed nodes**, **Fargate profiles** (no nodes, with limitations such as no DaemonSets or GPUs), **Karpenter** (just-in-time node provisioning, bin-packing, Spot and consolidation), and EKS Auto Mode (AWS-managed compute, storage, and load balancing).

**Networking**
- **VPC CNI** assigns real VPC IPs to pods, so plan subnet sizing carefully (use prefix delegation or custom networking if IPs run short).
- **AWS Load Balancer Controller** creates ALBs (Ingress) and NLBs (Service type LoadBalancer).
- Network policies are supported by the VPC CNI. Cilium is an alternative CNI.

**Security**
- **Access:** EKS access entries (newer) or the aws-auth ConfigMap map IAM principals to Kubernetes RBAC.
- **Pod-level IAM:** **IRSA** (OIDC provider and service account annotation) or **EKS Pod Identity** (simpler, no OIDC trust per cluster).
- Secrets envelope encryption with KMS, Pod Security Standards, Kyverno or Gatekeeper, and GuardDuty EKS Runtime Monitoring.

**Operations**
- Add-ons (CoreDNS, kube-proxy, VPC CNI, EBS CSI), version upgrades (control plane first, then add-ons, then nodes, one minor version at a time), Helm and GitOps (Argo CD/Flux), and observability with Container Insights or Prometheus/Grafana.

**ECS vs EKS:** ECS is simpler and deeply integrated. EKS offers portability, a rich ecosystem (operators, service meshes, CRDs), and a standard API, at the cost of more operational complexity. The control plane costs a flat hourly fee.

---

## 4. ECR (Elastic Container Registry)

Managed, OCI-compliant container (and Helm/OCI artifact) registry.

- **Private registries** (per account and region, with IAM and repository policies) and **ECR Public**.
- **Authentication:** `aws ecr get-login-password`, which issues a 12-hour token. Nodes, tasks, and pipelines use IAM roles.
- **Image scanning:** *Basic* (Clair-based on push) and *Enhanced* (Amazon Inspector, continuous, with OS and language package CVEs). Findings route to Security Hub and EventBridge.
- **Tag immutability:** Prevents overwriting tags like `v1.2.3` (important for reproducible deployments and audit).
- **Lifecycle policies:** Expire untagged or old images to control storage cost.
- **Replication:** Cross-region and cross-account, for DR and latency.
- **Pull-through cache:** Caches upstream registries (Docker Hub, Quay, GitHub Container Registry) to avoid rate limits and add scanning.
- **Encryption:** At rest with AES-256 or KMS.
- **Image signing:** Via AWS Signer or Cosign, verified at admission (Kyverno or Gatekeeper).

**Best practice:** Deploy by immutable tag or digest (`@sha256:...`), scan on push, gate promotion on scan results, and use VPC endpoints (`ecr.api`, `ecr.dkr`, plus S3 gateway endpoint) for private clusters.

---

## 5. API Gateway

Managed service for creating, publishing, securing, and monitoring APIs.

**API types**
| Type | Notes |
|---|---|
| **REST API** | Full-featured: API keys and usage plans, request/response validation and transformation (VTL), caching, WAF, resource policies, private endpoints, canary stages |
| **HTTP API** | Cheaper (~70% less), lower latency, simpler. JWT authorizers, OIDC, Lambda and HTTP proxy. Fewer features |
| **WebSocket API** | Two-way real-time communication with route selection |

**Endpoint types:** Edge-optimized (CloudFront), Regional, and **Private** (accessible only through an interface VPC endpoint, important for internal enterprise integration).

**Integrations:** Lambda (proxy or custom), HTTP/HTTP proxy, AWS service integrations (direct to SQS, Step Functions, DynamoDB, with no Lambda needed), Mock, and VPC Link (to private ALB/NLB, via a VPC Link v2 for HTTP APIs).

**Security**
- **Authorization:** IAM (SigV4), Cognito user pools, Lambda authorizers (custom token or request), JWT (HTTP API), and mutual TLS with custom domains.
- **WAF** (REST), resource policies (IP and VPC restrictions), and request validation.

**Traffic management:** Throttling at account, stage, and method levels (token bucket: rate and burst), usage plans and API keys (metering, not security), and response caching (REST).

**Deployment:** Stages (dev/prod), stage variables, canary release settings, and custom domains with base path mappings. Define everything in OpenAPI via IaC.

**Limits to remember:** 29-second default integration timeout for REST APIs (can be raised for Regional APIs via a quota request), 10 MB payload, and 30 seconds for HTTP APIs. For long processing, go asynchronous: return 202 and use SQS or Step Functions.

**Observability:** Access logs (custom format), execution logs, CloudWatch metrics (4XX, 5XX, Latency, IntegrationLatency), and X-Ray tracing.

---

## 6. Route 53

Highly available DNS, domain registration, and health checking, with a 100% availability SLA.

**Hosted zones:** **Public** and **Private** (associated with VPCs; enable `enableDnsHostnames` and `enableDnsSupport`).

**Record types:** A, AAAA, CNAME (not allowed at the zone apex), MX, TXT, NS, SRV, and **Alias** (AWS-specific; works at the apex, free queries to AWS targets like ALB, CloudFront, S3 website, API Gateway; follows target IP changes automatically).

**Routing policies**
| Policy | Use |
|---|---|
| Simple | One resource |
| Weighted | Canary and A/B, gradual migration |
| Latency | Route to the lowest-latency region |
| Failover | Active-passive DR, tied to health checks |
| Geolocation | Route by user location (compliance, localization) |
| Geoproximity | Location plus a bias to shift traffic (via Traffic Flow) |
| Multivalue answer | Up to 8 healthy records, a basic client-side load balancer |
| IP-based | Route by client CIDR |

**Health checks:** Endpoint (HTTP/HTTPS/TCP), calculated (combining other checks), and CloudWatch alarm-based (the way to monitor private resources). Checkers run from public AWS locations, so private endpoints need an alarm-based check.

**Route 53 Resolver:** The VPC's DNS (.2 address). **Inbound endpoints** let on-prem resolve AWS names, **outbound endpoints** plus **forwarding rules** let AWS resolve on-prem names, and **DNS Firewall** blocks malicious domains. Rules can be shared across accounts with RAM.

**Other:** Application Recovery Controller (routing controls for multi-region failover), DNSSEC signing, query logging, and low TTLs before migrations or cutovers.

---

## 7. ALB / NLB

Elastic Load Balancing family.

### ALB (Application Load Balancer, Layer 7)
- Understands HTTP/HTTPS/gRPC and WebSocket. Content-based routing via **listener rules** (host, path, header, query string, source IP, HTTP method).
- Targets: Instances, IPs, Lambda, and ECS/EKS pods.
- Features: Weighted target groups (canary and blue/green), sticky sessions (cookie), **authentication** (Cognito/OIDC at the listener), redirects and fixed responses, WAF integration, SNI with multiple certificates, HTTP/2, and slow start.
- Preserves client IP in the `X-Forwarded-For` header. Cross-zone load balancing is on by default.
- Ideal for microservices, web apps, and container ingress.

### NLB (Network Load Balancer, Layer 4)
- TCP, UDP, TLS. Extremely high throughput and ultra-low latency, handles sudden spikes.
- **Static IP per AZ** and Elastic IP support, a key reason to choose it (partner allow-listing). Also the front end for **PrivateLink** services.
- Preserves source IP by default (for instance targets). Supports TLS termination and security groups (added in 2023).
- Cross-zone is off by default (extra data charges if enabled).
- Ideal for non-HTTP protocols, gaming, IoT, and very high performance.

### Also know
- **Gateway Load Balancer (GWLB):** Layer 3, for inserting third-party firewalls and inspection appliances (GENEVE).
- **Target groups and health checks:** Health check path, thresholds, and deregistration delay (connection draining).
- **Common combo:** NLB in front of an ALB (to get static IPs plus L7 routing), or API Gateway VPC Link to an NLB or ALB.
- **Access logs** to S3, **idle timeout** tuning (default 60s on ALB), and deletion protection.

---

## 8. CloudWatch and CloudTrail

These answer different questions: CloudWatch is "how is it performing?" and CloudTrail is "who did what?"

### CloudWatch
- **Metrics:** Namespaces, dimensions, resolution (standard 1-minute, high-resolution 1-second), and custom metrics (via API or Embedded Metric Format in logs). EC2 memory and disk require the **CloudWatch agent**.
- **Alarms:** Static threshold, anomaly detection, and composite alarms (reduce noise). Actions include SNS, Auto Scaling, EC2 recover, and Systems Manager.
- **Logs:** Log groups and streams, retention policies (set them, since the default is never expire), **Logs Insights** (query language), **metric filters**, **subscription filters** (stream to Lambda, Kinesis, OpenSearch, Firehose), and cross-account observability. Infrequent Access log class lowers cost.
- **Dashboards**, **Synthetics** (canaries that test endpoints and flows), **RUM**, **Evidently** (feature flags), **Application Signals** (SLOs for services), **Container Insights**, **Lambda Insights**, and **Contributor Insights**.
- **Events** is now **EventBridge**.
- **Tracing partner:** X-Ray (or OpenTelemetry via the ADOT collector) for distributed traces and service maps.

### CloudTrail
- Records **API activity** across the account.
- **Event types:** *Management events* (control plane, e.g., `RunInstances`, `CreateRole`; on by default, 90-day history in Event history), *Data events* (S3 object-level, Lambda invoke, DynamoDB item-level; high volume, opt-in), and *Insights events* (anomalous API call rates or error rates).
- **Trails:** Create an **organization trail** delivering to a central S3 bucket in the log-archive account, with SSE-KMS, **log file integrity validation** (digest files), and optional CloudWatch Logs integration for real-time alerting.
- **CloudTrail Lake:** Managed event data store with SQL querying.
- **Typical uses:** Security forensics, compliance audit, detecting root account use or console logins without MFA (via EventBridge or metric filters), and generating least-privilege policies with Access Analyzer.

**Related services:** AWS Config (resource configuration history and compliance rules) completes the audit picture. VPC Flow Logs covers network.

---

## 9. Secrets Manager and KMS

### KMS (Key Management Service)
- Managed cryptographic keys, backed by FIPS 140-3 validated HSMs. You never see plaintext key material.
- **Key types:** *AWS owned* (invisible), *AWS managed* (`aws/s3`; AWS controls policy and rotation), and **Customer managed keys (CMKs)** (you control the policy, rotation, and lifecycle). Symmetric (AES-256-GCM, the default) and asymmetric (RSA, ECC) keys are supported, plus HMAC keys.
- **Envelope encryption:** `Encrypt` accepts only 4 KB directly. For larger data, call `GenerateDataKey`, which returns a plaintext data key and the same key encrypted under the KMS key. You encrypt the data locally, discard the plaintext key, and store the encrypted data key with the data. AWS services do this for you.
- **Access control:** The **key policy** is the primary authority (unlike most resources, IAM alone is not enough unless the key policy delegates to IAM). It can be supplemented by IAM policies and **grants** (temporary delegated permissions used by services like EBS).
- **Encryption context:** Additional authenticated data that is logged in CloudTrail and can be used in policy conditions.
- **Rotation:** Automatic yearly (configurable) rotation of CMK backing material, keeping old versions so old ciphertext still decrypts. The key ID and ARN stay the same.
- **Multi-Region keys:** Same key ID and material replicated across regions, for global tables and DR decryption.
- **Other:** Key deletion has a 7 to 30 day waiting period (disable first), imported key material, and custom key stores (CloudHSM). Cross-account use requires both the key policy and the caller's IAM policy.
- **Watch for:** KMS API throttling and per-request cost on S3 SSE-KMS (use Bucket Keys), and every CloudTrail log showing key usage, which is great for audit.

### Secrets Manager
- Stores, retrieves, and **rotates** secrets (DB credentials, API keys, OAuth tokens). Encrypted with a KMS key.
- **Automatic rotation** via Lambda (built-in templates for RDS, Aurora, Redshift, DocumentDB), using single-user or alternating-users strategies. Versions have staging labels (`AWSCURRENT`, `AWSPREVIOUS`, `AWSPENDING`).
- **Access:** IAM and resource policies (including cross-account), and **multi-region replication**.
- **Consumption:** SDK/CLI, caching clients (reduce API calls and cost), ECS task definitions (`secrets` injecting env vars), EKS via External Secrets Operator or the Secrets Store CSI driver, and Lambda extension.
- **Pricing:** About $0.40 per secret per month plus API calls.

### Secrets Manager vs Parameter Store
| | Secrets Manager | Parameter Store |
|---|---|---|
| Rotation | Built-in | No (DIY) |
| Cost | Paid per secret | Standard tier free, Advanced paid |
| Size | 64 KB | 4 KB (8 KB advanced) |
| Best for | Credentials needing rotation | Config values, simple secrets, feature flags |

---

## 10. EventBridge, SNS and SQS

The three core messaging and event services, each with a distinct role.

### SQS (Simple Queue Service): a **queue** (pull model)
- **Standard:** Nearly unlimited throughput, at-least-once delivery, best-effort ordering. **FIFO:** Exactly-once processing (5-minute deduplication window), strict ordering per **MessageGroupId**, 300 TPS (3,000 with batching) per API, or far higher with high-throughput mode.
- **Key settings:** *Visibility timeout* (default 30s; time a message is hidden while being processed), *long polling* (`WaitTimeSeconds` up to 20, reduces empty receives and cost), *retention* (up to 14 days), *delay queues*, and max message size 256 KB (use the Extended Client with S3 for larger payloads).
- **Dead-letter queues (DLQ):** After `maxReceiveCount` failures, messages move to the DLQ. Use **redrive** to move them back after a fix. FIFO sources need FIFO DLQs.
- **Consumers:** Lambda (event source mapping), ECS/EKS workers (scale with KEDA or Application Auto Scaling on queue depth), and EC2.
- **Security:** SSE (SQS-owned or KMS), queue policies for cross-account and SNS or EventBridge to SQS delivery.
- **Purpose:** Decouple producers and consumers, buffer spikes (load leveling), and add retry resilience.

### SNS (Simple Notification Service): **pub/sub** (push model)
- Publishers send to **topics**, and SNS pushes to all subscribers: SQS, Lambda, HTTP/S, email, SMS, mobile push, and Firehose.
- **Fan-out pattern:** SNS topic → multiple SQS queues, each with its own consumer and retry behavior.
- **Subscription filter policies** let each subscriber receive only relevant messages (on attributes or message body).
- **FIFO topics** provide ordering and deduplication (FIFO SQS subscribers only).
- Features: Delivery retries with policies, subscription-level DLQs, raw message delivery, message archiving/replay on FIFO topics, and encryption with KMS.
- **No persistence:** If a subscriber isn't reachable and retries are exhausted, the message is lost without a DLQ. That is why SNS to SQS is the common durable combination.

### EventBridge: a serverless **event bus** with routing
- **Event buses:** Default (AWS service events), custom (your apps), and partner (SaaS). Cross-account and cross-region bus targets are supported.
- **Rules** match events by **event patterns** (rich content filtering: prefix, numeric, exists, anything-but) and route to 20+ target types (Lambda, SQS, SNS, Step Functions, Kinesis, API destinations, ECS tasks, another bus, and more). Input transformers reshape the payload.
- **Schema registry** (discovery and code bindings), **Archive and Replay** (reprocess historical events), **API destinations** (call external HTTP APIs with auth and rate limiting), **EventBridge Scheduler** (cron or one-time schedules at scale, replaces scheduled rules), and **Pipes** (point-to-point from a source such as SQS or DynamoDB Streams, with filter, enrich, and target).
- Delivery is at-least-once, with retry policy (up to 24 hours) and a DLQ per target.

### How to choose
| Need | Pick |
|---|---|
| Buffer and process work at the consumer's pace | **SQS** |
| Broadcast one message to many subscribers | **SNS** (fan-out with SQS) |
| Route events by content, react to AWS or SaaS events, loosely coupled event-driven architecture, schedules | **EventBridge** |
| Ordered streaming with replay and multiple readers | Kinesis or MSK (outside this list) |

**Design rules:** Make consumers **idempotent**, always configure **DLQs**, use SNS or EventBridge → SQS → Lambda for durable processing, and monitor `ApproximateAgeOfOldestMessage` and DLQ depth.

---

## 11. Step Functions

Serverless workflow orchestration. You define a **state machine** in **Amazon States Language (ASL)**, a JSON or YAML-like definition, designed visually in Workflow Studio.

**Workflow types**
| | Standard | Express |
|---|---|---|
| Duration | Up to 1 year | Up to 5 minutes |
| Semantics | Exactly-once execution | At-least-once (async) or at-most-once (sync) |
| Throughput | ~2,000 execution starts/sec | 100,000+/sec |
| History | Full, visible in console, 90 days | CloudWatch Logs only |
| Pricing | Per state transition | Per request and duration |
| Use | Long-running, auditable, human approval | High-volume event processing, streaming transforms |

**State types:** `Task` (do work), `Choice` (branching), `Parallel` (concurrent branches), `Map` (iterate over an array; **Distributed Map** processes millions of S3 objects at up to 10,000 parallel child executions), `Wait`, `Pass`, `Succeed`, `Fail`.

**Integrations**
- 220+ services via **AWS SDK integrations**, and *optimized* integrations for Lambda, ECS/Fargate, Batch, Glue, SNS, SQS, DynamoDB, EventBridge, and nested Step Functions.
- Integration patterns: **Request-Response** (fire and move on), **Run a Job (`.sync`)** (wait for completion, e.g., Batch or ECS task), and **Wait for Callback (`.waitForTaskToken`)** (pause until an external system or human responds, ideal for approvals).
- Direct integrations often remove the need for glue Lambdas, which means lower cost and less code.

**Error handling (heavily tested)**
- `Retry` with `ErrorEquals`, `IntervalSeconds`, `BackoffRate`, `MaxAttempts`, and jitter.
- `Catch` routes to a fallback state, enabling cleanup or **compensation (saga pattern)** to undo earlier steps when a later one fails.
- Timeouts (`TimeoutSeconds`, `HeartbeatSeconds`) prevent hung tasks.

**Data handling:** `InputPath`, `Parameters`, `ResultSelector`, `ResultPath`, `OutputPath`, and newer **JSONata** and variables support. The **payload limit is 256 KB** per state, so pass references (S3 keys) for large data.

**Security and operations:** An execution role with least privilege per state machine, X-Ray tracing, CloudWatch logging, EventBridge events for execution status changes, and **Redrive** to resume failed executions from the point of failure. Versions and aliases support gradual deployments.

**When to use it:** Multi-step workflows with branching, retries, parallelism, or human steps (order processing, EDI pipelines, ETL orchestration, ML pipelines, approval flows). **Don't** use it for a simple single-step function, or to implement sub-millisecond logic. Step Functions shines on orchestration, while EventBridge and SQS handle choreography.

**Orchestration vs choreography:** Step Functions centralizes control and visibility (orchestration). EventBridge and SNS let services react independently (choreography). Large systems often combine both.

---

## How These Services Fit Together

A typical integration flow: **Route 53** → **API Gateway** (private or public) → **Lambda** → **EventBridge** → **SQS** (with DLQ) → **Step Functions** → **ECS/EKS workers** (images in **ECR**) → **S3**, with everything in a **VPC** behind **ALB/NLB**, secured by **IAM** roles, **KMS**-encrypted, secrets in **Secrets Manager**, and monitored by **CloudWatch** with **CloudTrail** auditing.

If it would help, I can put this into a Word or PDF study guide, add architecture diagrams for any of these services, or write a quiz and flashcards so you can test yourself before the interview.
