# Terraform interview Q&A (AWS-based), end to end

Version-specific features change quickly (state locking, ephemeral values, testing, licensing), so confirm against the current Terraform and AWS provider docs before quoting version numbers. Answers are model answers. Add one real example from your own work to the ones marked ⭐.

---

## Part 1: Fundamentals

**Q1. What is Terraform and how does it work?**
A declarative IaC tool. Terraform core reads your configuration, builds a dependency graph, compares desired state against the state file and the real infrastructure (via provider APIs), and produces a plan of creates, updates and deletes. Providers (such as `hashicorp/aws`) translate resources into API calls.

**Q2. Walk me through the core workflow.**
`init` (downloads providers and modules, configures the backend), `validate` (syntax), `plan` (preview), `apply` (execute), `destroy`. In CI I save the plan with `-out` and apply that exact file after approval, so what was reviewed is what runs.

**Q3. Declarative vs imperative, and what does idempotent mean here?**
You describe the end state, not the steps. Re-running `apply` with no changes does nothing. Terraform figures out the order from references between resources.

**Q4. `count` vs `for_each`?**
`count` is index-based, so removing item 0 from a list shifts every later index and can destroy and recreate resources. `for_each` keys resources by a map key or set value, so adding or removing one item affects only that item. Prefer `for_each` for anything where identity matters. Both need values known at plan time.

**Q5. How do dependencies work?**
Implicit: referencing `aws_vpc.main.id` makes Terraform create the VPC first. Explicit: `depends_on`, used only when the dependency isn't visible through references (for example an IAM policy that must exist before an instance starts). Overusing `depends_on` slows plans and hides design problems.

**Q6. Resource vs data source?**
A resource is created and managed by Terraform. A data source reads existing information (for example `aws_availability_zones`, `aws_caller_identity`, an AMI lookup, or `aws_ssm_parameter`). Data sources run during plan, and if their inputs aren't known yet, they are deferred to apply time.

**Q7. How do variable values get resolved, and what is the precedence?**
Lowest to highest: defaults, environment variables (`TF_VAR_name`), `terraform.tfvars` and `terraform.tfvars.json`, `*.auto.tfvars` (alphabetical), then `-var` and `-var-file` on the command line, in the order given (later wins).

**Q8. Variables, locals and outputs: when do you use each?**
Variables are inputs, locals are computed values reused inside a module, outputs expose values to the parent module or the CLI. Use `validation` blocks for input rules and `sensitive = true` to hide values from CLI output.

**Q9. What are `lifecycle` rules?**
`create_before_destroy` (replace without a gap), `prevent_destroy` (guard critical resources), `ignore_changes` (ignore drift on attributes managed elsewhere, such as an ASG's `desired_capacity`), `replace_triggered_by`, and `precondition` and `postcondition` for custom checks.

**Q10. Name some functions and expressions you use daily.**
`merge`, `lookup`, `try`, `coalesce`, `flatten`, `zipmap`, `cidrsubnet`, `templatefile`, `jsonencode`, `for` expressions, conditional expressions (`cond ? a : b`), and `dynamic` blocks.

**Q11. What are provisioners, and should you use them?**
`local-exec` and `remote-exec` run commands during create or destroy. HashiCorp treats them as a last resort. Prefer user data, cloud-init, configuration tools (Ansible), or purpose-built resources, because provisioners aren't tracked in state and fail in awkward ways.

---

## Part 2: State management

**Q12. What is the state file and why does it matter?** ⭐
It maps your configuration to real resource IDs, stores attributes and dependency metadata, and enables fast planning. It can contain **secrets in plain text**, so it must be encrypted and access-controlled.

**Q13. How do you configure remote state on AWS?**
An S3 bucket with encryption, versioning, Block Public Access and a restrictive bucket policy, separate state keys per environment and layer, and locking. Locking historically used a DynamoDB table. Newer Terraform versions support native S3 locking with `use_lockfile`, and DynamoDB locking is being deprecated, so check which your version supports.

```hcl
terraform {
  required_version = ">= 1.10"

  backend "s3" {
    bucket       = "acme-tfstate-prod"
    key          = "network/terraform.tfstate"
    region       = "eu-west-1"
    encrypt      = true
    use_lockfile = true      # older versions: dynamodb_table = "tf-locks"
  }
}
```

**Q14. Can the backend block use variables?**
No. The backend is configured before variables are evaluated. Use partial configuration with `terraform init -backend-config=...` files or flags per environment.

**Q15. What happens if two people run `apply` at once?**
Without locking, the state can be corrupted. With locking, the second run waits or fails with a lock error. If a crashed run leaves a stale lock, `terraform force-unlock <ID>` clears it, after you confirm no run is active.

**Q16. The state file was corrupted or deleted. What do you do?** ⭐
Restore the previous version from S3 versioning. If there is no backup, rebuild state with `import` (import blocks, then `plan` until clean). This is why versioning on the state bucket and a separate, protected bucket policy are mandatory.

**Q17. Useful `terraform state` commands?**
`state list`, `state show`, `state mv` (rename or move resources without recreating them), `state rm` (stop managing a resource without destroying it), `state pull` and `push`. Take a backup before any manual state change. In modern versions, prefer `moved` and `removed` blocks, which are reviewable in code.

**Q18. How do you refactor without destroying resources?**
A `moved` block records the rename or move, and plan shows it as a move instead of a replace.

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

**Q19. How do you import existing AWS resources?** ⭐
Use an `import` block (Terraform 1.5 and later) with `plan -generate-config-out=generated.tf` to draft the configuration, then clean it up and iterate until the plan shows no changes. The older `terraform import` CLI command works one resource at a time and doesn't write the code for you.

```hcl
import {
  to = aws_security_group.legacy
  id = "sg-0abc123def456"
}
```

**Q20. Workspaces vs separate directories for environments?**
Workspaces share one configuration with different state files, which is easy to misuse (same backend, same credentials, little visibility of which workspace you're in). For dev, test and prod, I prefer separate directories or root modules with separate state, accounts and credentials. Workspaces suit short-lived copies of the same stack, such as per-branch test environments.

**Q21. How do you split state to limit blast radius?**
By layer (network, data, cluster, apps) and environment, so a change to an app doesn't plan against the whole account. Layers share values through outputs, `terraform_remote_state` or SSM parameters. Trade-offs: more pipelines and an explicit apply order.

**Q22. What is drift, and how do you handle it?** ⭐
Real infrastructure differing from state, usually through console changes. Detect with a scheduled `plan` using `-detailed-exitcode` (0 no changes, 1 error, 2 changes) and alert on 2. Remediate by re-applying, or codify the change if legitimate. Prevent with IAM and SCP restrictions on console changes. `plan -refresh-only` updates state to match reality without changing infrastructure.

---

## Part 3: Modules and structure

**Q23. What makes a good module?**
Single purpose, typed variables with validation, sensible defaults, documented outputs, no hard-coded regions or account IDs, versioned releases, and examples and tests. Avoid thin wrappers around a single resource, and avoid giant "do everything" modules with dozens of toggles.

**Q24. How do you version and consume modules?**
Publish through a Git tag or a private registry with semantic versions, and pin consumers: `source = "git::https://...//vpc?ref=v1.4.2"` or `version = "~> 1.4"`. Upgrade dev first, then promote.

**Q25. Root module vs child module, and how does data flow?**
The root module is where you run Terraform. Child modules receive inputs through arguments and return values through outputs. A child can't read the parent's variables directly.

**Q26. How do you structure a repo for dev, staging and prod on AWS?**
```
infra/
├── modules/            # vpc, eks, rds, alb, iam-roles ...
└── live/
    ├── dev/{network,data,eks,apps}/
    ├── staging/...
    └── prod/...
```
Each environment lives in its own AWS account with its own state key. Same modules, different `tfvars`.

**Q27. When would you use Terragrunt?**
To reduce repetition in a large `live/` tree (backend config, provider blocks, dependency ordering between stacks). The trade-off is another tool and abstraction layer for the team to learn. I'd use it when repetition is genuinely painful, not by default.

**Q28. Public registry modules vs writing your own?**
Well-maintained community modules (such as `terraform-aws-modules`) save time for VPC, EKS and RDS, but you must pin versions, read the code, and accept their upgrade pace. Write your own where your standards differ, or where a module hides too much.

---

## Part 4: Providers and multi-account

**Q29. How do you manage provider versions?**
Constraints in `required_providers` (for example `~> 5.0`), and commit `.terraform.lock.hcl` so every machine and the pipeline use identical provider builds. Upgrade deliberately with `terraform init -upgrade`, and read the provider changelog first, since major versions change resource behavior.

**Q30. How do you deploy to multiple regions or accounts?**
Provider aliases, optionally with `assume_role`.

```hcl
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias  = "dr"
  region = "eu-central-1"
}

module "dr_bucket" {
  source    = "./modules/bucket"
  providers = { aws = aws.dr }
}
```
For many accounts, I prefer one pipeline per account with a role in that account, rather than one giant configuration touching all of them.

**Q31. How should Terraform authenticate to AWS?**
No static keys. Pipelines use OIDC to assume a role (GitHub Actions, GitLab CI), and humans use SSO profiles. Roles are scoped per environment and per repo or branch.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}

data "aws_iam_policy_document" "trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:myorg/infra:ref:refs/heads/main"]
    }
  }
}
```
Older provider versions required a thumbprint list for this resource, so check the current docs.

**Q32. What is `default_tags`?**
A provider-level block that applies tags to every taggable resource automatically, for cost allocation and ownership. Combine it with SCPs or tag policies to enforce required tags.

---

## Part 5: AWS resources in Terraform

### Networking

**Q33. Build a 3-AZ VPC with public and private subnets.**
Derive AZs from a data source and carve CIDRs with `cidrsubnet`.

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

resource "aws_subnet" "private" {
  for_each          = { for i, az in local.azs : az => i }
  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, each.value)
}
```
Public subnets use a different offset. Add an internet gateway, route tables and associations, one NAT gateway per AZ for production (a single shared NAT for non-prod saves cost), and gateway endpoints for S3 and DynamoDB.

**Q34. Security groups: inline rules vs separate rule resources?**
Mixing inline `ingress` or `egress` blocks with separate rule resources on the same group causes conflicts and perpetual diffs. Pick one style. Separate rule resources are easier to compose across modules. The AWS provider has newer per-rule resources, so check current recommendations.

**Q35. How do you avoid hard-coding AZs, AMIs and account IDs?**
Data sources: `aws_availability_zones`, `aws_caller_identity`, `aws_region`, and an SSM parameter lookup for the latest AMI (`aws_ssm_parameter` for the Amazon Linux path), or `aws_ami` with filters.

### Compute and load balancing

**Q36. Create an Auto Scaling Group behind an ALB.**
`aws_launch_template` (AMI, instance profile, security group, IMDSv2 required, user data), `aws_autoscaling_group` with `vpc_zone_identifier` for private subnets, `target_group_arns`, `health_check_type = "ELB"`, and `instance_refresh` for rolling AMI updates. Then `aws_lb`, `aws_lb_target_group` with a health check path, and `aws_lb_listener` (443 with an ACM certificate, 80 redirecting to 443). Use a target tracking scaling policy.

**Q37. How do you handle the ASG's desired capacity drifting?**
Autoscaling changes it at runtime, so use `lifecycle { ignore_changes = [desired_capacity] }`, otherwise every plan tries to reset it.

**Q38. Issue an ACM certificate with DNS validation.**
`aws_acm_certificate`, then `aws_route53_record` with `for_each` over `domain_validation_options`, then `aws_acm_certificate_validation` to wait for issuance. This is a classic example of why `for_each` keys must be known at plan time.

### IAM and security

**Q39. How do you write IAM policies in Terraform?**
Prefer `data "aws_iam_policy_document"` over raw JSON strings, since it validates structure and composes well. Keep trust policies and permission policies separate. Avoid wildcard actions and resources, and use conditions where possible.

**Q40. EC2 instance role: what resources do you need?**
`aws_iam_role` (trust policy for `ec2.amazonaws.com`), `aws_iam_policy` plus attachment (or inline policy), and `aws_iam_instance_profile`, which is what the instance actually references. In the console the profile is created automatically, but in Terraform it is explicit.

**Q41. KMS and Secrets Manager?**
`aws_kms_key` with rotation enabled and an explicit key policy, an alias, and `aws_secretsmanager_secret` encrypted with that key. Don't put secret values in Terraform code or `tfvars`.

### Storage and data

**Q42. Create a secure S3 bucket.**
In AWS provider v4 and later, bucket settings are separate resources: `aws_s3_bucket`, `aws_s3_bucket_public_access_block` (all four settings true), `aws_s3_bucket_versioning`, `aws_s3_bucket_server_side_encryption_configuration` (KMS where needed), `aws_s3_bucket_lifecycle_configuration`, and a bucket policy denying non-TLS requests (`aws:SecureTransport` false). Add replication for DR. Wrap all of this into a module so every bucket meets the standard.

**Q43. How do you handle RDS passwords safely?** ⭐
Don't hard-code them. Use `manage_master_user_password = true`, which has RDS create and rotate the secret in Secrets Manager, so the password never appears in your code. If you generate one with `random_password`, remember it is stored in state, so the state must be encrypted and tightly controlled. Newer Terraform versions add ephemeral values and write-only attributes that keep secrets out of state, so check which providers support them.

**Q44. What are the important RDS settings to get right?**
`multi_az`, `db_subnet_group_name`, parameter groups, `storage_encrypted` with a KMS key, `backup_retention_period`, `deletion_protection = true` in production, `skip_final_snapshot = false` with a snapshot identifier, and `apply_immediately` (without it, many changes wait for the maintenance window). Some parameter changes force replacement, so always read the plan.

**Q45. DynamoDB with autoscaling or on-demand?**
`aws_dynamodb_table` with `billing_mode = "PAY_PER_REQUEST"` for unpredictable traffic, or provisioned mode with `aws_appautoscaling_*` resources. Add point-in-time recovery, encryption, TTL and global table replicas where needed. With provisioned capacity, `ignore_changes` on read and write capacity avoids fighting autoscaling.

### Containers and serverless

**Q46. How do you provision EKS with Terraform?** ⭐
Either the community EKS module or your own: cluster, IAM roles, managed node groups or Karpenter, an OIDC provider for IRSA (or EKS Pod Identity), access entries, add-ons, and logging. The classic trap is configuring the `kubernetes` and `helm` providers from the cluster created in the same configuration, which can fail on first apply because the cluster doesn't exist at plan time. The robust pattern is separate layers: one state for the cluster, another for in-cluster resources that reads the cluster data source.

**Q47. How do you deploy a Lambda function behind API Gateway?**
`archive_file` (or a build artifact in S3), `aws_lambda_function` with `source_code_hash` so code changes trigger updates, an execution role, `aws_lambda_permission` allowing API Gateway to invoke it, and `aws_apigatewayv2_api`, `integration`, `route`, `stage` for an HTTP API. For REST APIs, a deployment needs `triggers` (a hash of the API definition) so changes redeploy.

**Q48. ECS on Fargate?**
`aws_ecs_cluster`, `aws_ecs_task_definition` (execution role to pull images and write logs, task role for application permissions), `aws_ecs_service` with `network_configuration`, load balancer wiring, deployment circuit breaker, and autoscaling. Task definition changes create a new revision, so image tags should be immutable or pinned by digest.

### DNS, CDN and observability

**Q49. Route 53 and CloudFront?**
`aws_route53_zone`, alias records pointing to the ALB or CloudFront (aliases work at the apex and are free), health checks and failover or weighted routing where needed. CloudFront needs its ACM certificate in `us-east-1`, which is a common multi-provider alias example.

**Q50. CloudWatch alarms and log retention in code?**
`aws_cloudwatch_log_group` with `retention_in_days` (otherwise logs are kept forever, which costs money), `aws_cloudwatch_metric_alarm` with SNS actions, and dashboards as code. Create log groups explicitly, so Lambda and ECS don't create unmanaged groups with infinite retention.

---

## Part 6: Advanced language features

**Q51. When do you use `dynamic` blocks, and what are the risks?**
To generate repeated nested blocks from a variable (security group rules, listener rules). Overuse makes code hard to read, so use them where the number of blocks is truly variable.

```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.port
    to_port     = ingress.value.port
    protocol    = "tcp"
    cidr_blocks = ingress.value.cidrs
  }
}
```

**Q52. Why do you see "value depends on resource attributes that cannot be determined until apply" with `for_each`?**
Keys must be known during plan. If keys come from a computed value (for example an ID created in the same run), Terraform can't build the graph. Fix by keying on static values (names) rather than generated IDs, or by splitting into two applies or layers.

**Q53. How do you create conditional resources?**
`count = var.enabled ? 1 : 0`, then reference with `[0]` or a splat. With `for_each`, use an empty set or map. Keep conditionals shallow, because heavy conditional logic makes modules brittle.

**Q54. What are `precondition` and `postcondition`?**
Custom validation attached to resources, data sources or outputs. For example, assert that a chosen subnet is private, or that an AMI is from an approved owner, with a clear error message at plan time.

**Q55. How do you test Terraform?**
`fmt` and `validate`, `tflint`, security and policy scanning (Checkov or Trivy config scanning), the built-in `terraform test` framework (1.6 and later), and Terratest for real deployments in a sandbox account. For modules, run examples in CI and destroy afterwards.

---

## Part 7: CI/CD, security and governance

**Q56. Describe a Terraform CI/CD pipeline.** ⭐
On a pull request: `fmt -check`, `validate`, tflint, Checkov, `plan` posted as a comment (with Infracost for cost impact). On merge: apply to dev automatically, then staging, then prod behind a manual approval, applying the saved plan file. Authenticate with OIDC, use one role per environment, keep a concurrency lock so only one apply runs per stack, and run a scheduled drift-detection plan.

**Q57. Atlantis, Terraform Cloud (HCP Terraform) or plain CI?**
Atlantis automates plan and apply through pull request comments and is self-hosted. HCP Terraform offers hosted state, runs, policy (Sentinel or OPA), private registry and RBAC, with pricing and feature-tier changes to check. Plain GitHub Actions or GitLab CI gives the most control with the most DIY work. Choose by team size, compliance needs and operating appetite.

**Q58. How do you enforce policy as code?**
Static checks in CI (Checkov, Trivy, OPA/Conftest against the plan JSON), Sentinel or OPA in HCP Terraform, and AWS-side guardrails (SCPs, AWS Config rules, tag policies) as the safety net for anything created outside Terraform.

**Q59. How do you keep secrets out of Terraform?**
Don't pass secrets through variables or `tfvars` in Git. Use managed secret features (RDS-managed passwords, Secrets Manager references), mark variables and outputs `sensitive`, remember that sensitive values still land in state, and lock down the state bucket (KMS encryption, tight IAM, access logging). Use ephemeral values where the provider supports them.

**Q60. How do you give a team self-service without losing control?**
Approved, versioned modules in a private registry, a pipeline that runs checks and plans, policy as code, scoped IAM roles per team and environment, and CODEOWNERS on the platform modules. Developers write small `live/` files, and the platform team owns the modules.

**Q61. Terraform license change and OpenTofu?**
HashiCorp moved Terraform from MPL to the Business Source License in 2023. OpenTofu is the open-source fork under the Linux Foundation. Both are largely compatible, though features have diverged over time, so check current compatibility before choosing. Mention that you weigh the license against your organization's policy.

---

## Part 8: Troubleshooting

**Q62. `Error acquiring the state lock`.**
Another run is active or a previous run crashed. Check the lock info (who and when). If no run is active, `force-unlock` with the ID. Then find out why runs overlapped (missing pipeline concurrency control).

**Q63. `Error: Cycle`.**
Two resources reference each other. The classic AWS case is security groups that reference each other inline. Fix by splitting out the rules into separate resources. Use `terraform graph` to see the loop.

**Q64. `AccessDenied` during apply.**
Check which principal is making the call (`aws_caller_identity`), the permissions boundary and SCPs, the resource policy (for example KMS key policy or bucket policy), and whether the provider is using the intended role (aliases). Read CloudTrail for the denied API call.

**Q65. IAM role created but the next resource fails with "invalid principal" or "not authorized".**
IAM is eventually consistent. A short propagation delay can break the next step. Use `depends_on` on the policy attachment, retry, or `time_sleep` as a last resort.

**Q66. "Resource already exists."**
It was created outside Terraform or by a previous failed run. Import it, or remove the stray resource if it is genuinely unwanted.

**Q67. A plan shows a resource will be destroyed and recreated. What do you do?** ⭐
Read why (`# forces replacement` attribute). If it is unexpected, check recent provider upgrades, renamed arguments, or a `count` index shift. Use `moved` blocks for refactors, `ignore_changes` for externally managed attributes, and `create_before_destroy` to avoid downtime when the replacement is intended. Never apply a plan you haven't understood.

**Q68. Plans are slow with hundreds of resources.**
Split state by layer, avoid unnecessary data sources, tune `-parallelism`, use `-refresh=false` carefully when you know infrastructure didn't change (not as a habit), and avoid `-target` except in emergencies, since it creates partial applies and hidden drift.

**Q69. API throttling errors from AWS.**
Reduce parallelism, increase provider `max_retries`, split large stacks, and avoid creating thousands of resources of one type in a single run.

**Q70. The provider upgrade changed behavior.**
Pin and lock versions, read the upgrade guide, test in dev, and keep module version constraints so a provider major upgrade is a planned change.

---

## Part 9: Scenario questions

1. **"Design the Terraform for a new AWS account landing zone."** Organizations with OUs, SCPs, centralized logging and security accounts, baseline module (CloudTrail, Config, GuardDuty, IAM Access Analyzer), network layer with Transit Gateway, per-account pipelines with OIDC roles, and an account vending process.
2. **"Someone changed a security group in the console during an incident."** Detect it with drift detection, decide whether to codify or revert, record the reason, and tighten console access afterwards.
3. **"A teammate ran `terraform destroy` against production."** Prevention: separate prod credentials and approvals, `prevent_destroy`, deletion protection on data stores, and restricted IAM. Recovery: restore from snapshots and state versions.
4. **"Migrate 200 hand-made resources into Terraform."** Inventory by layer, import blocks with generated config, iterate to a clean plan, lock down console changes with SCPs, and migrate layer by layer starting with networking.
5. **"The RDS password appears in the state file."** Rotate it, move to RDS-managed secrets, tighten state access, and review who has read the state.
6. **"Deploy the same stack in two regions for DR."** Provider aliases or a region variable per root module, region-specific data lookups, replicated data stores (Aurora Global or DynamoDB global tables, S3 replication), and Route 53 failover. Keep the DR region's state separate.

---

## Part 10: Hands-on coding challenges to practise

1. Write a module for a secure S3 bucket (public access block, versioning, encryption, TLS-only policy, lifecycle), with variables, validation and outputs.
2. Build a 3-AZ VPC using `for_each` and `cidrsubnet`, with NAT per AZ and gateway endpoints.
3. Create an ASG behind an ALB with an ACM certificate and Route 53 alias.
4. Import a hand-made security group with an `import` block, then clean up the generated code.
5. Create IAM roles for several teams from a map variable using `for_each`.
6. Configure the GitHub OIDC role and use it from a workflow.
7. Deploy a resource to two regions using provider aliases.
8. Refactor a flat configuration into a module using `moved` blocks, with a plan that shows no destroys.

---

## Rapid-fire round

| Question | Short answer |
|---|---|
| Store state remotely on AWS? | S3 with encryption, versioning and locking |
| Rename a resource without recreating it? | `moved` block |
| Stop managing a resource without destroying it? | `state rm` or a `removed` block |
| Replace a specific resource? | `apply -replace=ADDRESS` (the older `taint` is deprecated) |
| Detect drift in CI? | `plan -detailed-exitcode` on a schedule |
| Hide values from output? | `sensitive = true` (still stored in state) |
| Backend with variables? | Not possible, use `-backend-config` |
| Apply to one resource only? | `-target`, but only in emergencies |
| `count` or `for_each` for stable identity? | `for_each` |
| Tag every resource automatically? | Provider `default_tags` |
| Lock provider builds? | Commit `.terraform.lock.hcl` |
| Why `create_before_destroy`? | Replace without downtime |
| Check Terraform code security? | Checkov or Trivy config scanning in CI |

---

## Tips for the interview

- **Draw the layout first:** modules, state per layer, accounts, pipeline. Most senior questions are about structure and safety, not syntax.
- **Always mention safety:** locking, plan review, approvals, no static keys, `prevent_destroy` on data stores.
- **Know the plan output.** Be able to explain what `~`, `+`, `-`, `-/+` and "forces replacement" mean.
- **Bring one real story** for state recovery, a risky refactor, an import project and a drift incident.
- **Say what you would check in the docs** rather than guessing version-specific details.

I can run a mock interview where I ask these one at a time and grade your answers, expand any section into a worked lab with full code, or build a complete sample repo layout (modules, live environments and a GitHub Actions pipeline) for you to practise on.
