CDK Basic 

## AWS CDK Core Concepts + Practical S3 Bucket in TypeScript

---

### 1. Core CDK Concepts

| Concept | What it is |
|---|---|
| **App** | Entry point of your CDK program. Holds one or more Stacks. |
| **Stack** | A deployment unit — maps 1:1 to a CloudFormation stack. All resources must live inside a Stack. |
| **Construct** | The building block. Can be a single resource (S3 bucket) or a composite (API + Lambda + permissions). |
| **L1 / L2 / L3** | L1 = raw CloudFormation resource (`CfnBucket`), L2 = high-level with defaults & methods (`s3.Bucket`), L3 = pattern combining multiple resources. |
| **Synth** | `cdk synth` walks the construct tree → emits a CloudFormation template into `cdk.out/`. |
| **Deploy** | `cdk deploy` uploads assets to the bootstrap bucket, then calls CloudFormation to create/update the stack. |
| **Tokens** | Placeholder values resolved at synth time (e.g. `cdk.Aws.ACCOUNT_ID`, a bucket's auto-generated ARN). |
| **Removal Policy** | What happens to the resource when the stack is destroyed: `RETAIN` (keep), `DESTROY` (delete), `SNAPSHOT`. |
| **Aspects** | Cross-cutting concerns applied to a construct tree (e.g. tagging all resources, permissions boundaries). |
| **Bootstrap** | `cdk bootstrap` creates a toolkit stack (S3 bucket + ECR repo + roles) in a target account/region so `cdk deploy` can upload assets. |

**Lifecycle:**

```
Your TypeScript Code
       │
       ▼  (cdk synth)
Construct Tree → Cloud Assembly (cdk.out/)
       │              └── template.json (CloudFormation)
       │              └── assets/ (zips, images)
       ▼  (cdk deploy)
CloudFormation Stack → AWS Resources (S3, Lambda, VPC…)
```

---

### 2. Project Setup

```bash
npm install -g aws-cdk
cdk init app --language typescript
cd my-cdk-app
```

This generates:

```
my-cdk-app/
├── bin/
│   └── my-cdk-app.ts        ← Entry point (App + Stack instantiation)
├── lib/
│   └── my-cdk-app-stack.ts  ← Stack definition
├── cdk.json                 ← App config (entry, output, context)
├── package.json
└── tsconfig.json
```

---

### 3. Full Practical S3 Bucket Implementation

#### `bin/my-cdk-app.ts` (Entry Point)

```typescript
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { S3DataStack } from '../lib/s3-data-stack';

const app = new cdk.App();

new S3DataStack(app, 'S3DataStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
  // Custom context values
  data: {
    environment: app.node.tryGetContext('environment') || 'dev',
    retentionDays: 90,
  },
});
```

#### `lib/s3-data-stack.ts` (The Stack)

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as s3notifications from 'aws-cdk-lib/aws-s3-notifications';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as kms from 'aws-cdk-lib/aws-kms';
import * as logs from 'aws-cdk-lib/aws-logs';

// ─── Props interface (makes the stack reusable) ───────────────────────
interface S3DataStackProps extends cdk.StackProps {
  environment: string;
  retentionDays: number;
}

// ─── The Stack ─────────────────────────────────────────────────────────
export class S3DataStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: S3DataStackProps) {
    super(scope, id, props);

    const isProd = props.environment === 'production';

    // ── KMS Key (for production encryption) ──────────────────────────
    const kmsKey = new kms.Key(this, 'BucketKey', {
      alias: `alias/data-bucket-${props.environment}`,
      enableKeyRotation: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // ── Main Data Bucket ─────────────────────────────────────────────
    const dataBucket = new s3.Bucket(this, 'DataBucket', {
      bucketName: isProd
        ? `mycompany-data-${cdk.Aws.ACCOUNT_ID}-${cdk.Aws.REGION}`
        : undefined, // auto-name for dev (avoids conflicts)

      // Security
      encryption: isProd
        ? s3.BucketEncryption.KMS
        : s3.BucketEncryption.S3_MANAGED,
      encryptionKey: isProd ? kmsKey : undefined,
      bucketKeyEnabled: isProd,          // reduces KMS API cost
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      enforceSSL: true,
      objectOwnership: s3.ObjectOwnership.BUCKET_OWNER_ENFORCED,

      // Durability
      versioned: true,
      serverAccessLogsBucket: this.createAccessLogsBucket(),
      serverAccessLogsPrefix: 'access/',

      // Lifecycle
      lifecycleRules: [
        {
          id: 'TransitionToIAAfter30Days',
          enabled: true,
          transitions: [
            {
              after: cdk.Duration.days(30),
              storageClass: s3.StorageClass.INFREQUENT_ACCESS,
            },
            {
              after: cdk.Duration.days(90),
              storageClass: s3.StorageClass.GLACIER,
            },
          ],
        },
        {
          id: 'ExpireNonCurrentVersions',
          enabled: true,
          noncurrentVersionExpiration: cdk.Duration.days(props.retentionDays),
          noncurrentVersionsToRetain: 3,
        },
      ],

      // Cleanup behaviour
      removalPolicy: isProd
        ? cdk.RemovalPolicy.RETAIN
        : cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: !isProd,  // CDK auto-creates a Lambda to empty bucket on destroy
    });

    // ── Expose the bucket as a CfnOutput (visible in CloudFormation) ─
    new cdk.CfnOutput(this, 'BucketName', {
      value: dataBucket.bucketName,
      exportName: `${this.stackName}-bucket-name`,
    });

    new cdk.CfnOutput(this, 'BucketArn', {
      value: dataBucket.bucketArn,
    });

    // ── SNS notification on new object upload ────────────────────────
    const topic = new sns.Topic(this, 'UploadNotificationTopic', {
      displayName: 'S3 Upload Notifications',
    });

    dataBucket.addEventNotification(
      s3.EventType.OBJECT_CREATED,
      new s3notifications.SnsDestination(topic),
      { prefix: 'uploads/' }   // only objects under /uploads/
    );

    // ── Grant a specific IAM role read access ────────────────────────
    const readerRole = new iam.Role(this, 'ReaderRole', {
      assumedBy: new iam.AccountRootPrincipal(),
    });
    dataBucket.grantRead(readerRole);

    // ── Add a bucket policy (only allow from a specific VPC) ────────
    // (Example: restrict to VPC endpoint)
    dataBucket.addToResourcePolicy(new iam.PolicyStatement({
      effect: iam.Effect.DENY,
      principals: [new iam.AnyPrincipal()],
      actions: ['s3:*'],
      resources: [dataBucket.bucketArn, dataBucket.arnForObjects('*')],
      condition: {
        Bool: { 'aws:ViaAWSService': 'false' },
      },
    }));
  }

  // ── Helper: separate bucket for access logs ────────────────────────
  private createAccessLogsBucket(): s3.IBucket {
    return new s3.Bucket(this, 'AccessLogsBucket', {
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      enforceSSL: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      lifecycleRules: [
        { expiration: cdk.Duration.days(365) },
      ],
    });
  }
}
```

---

### 4. Key Concepts Demonstrated

| Concept | Where it appears |
|---|---|
| **Props interface** | `S3DataStackProps` — makes the stack parameterizable |
| **Tokens** | `cdk.Aws.ACCOUNT_ID`, `cdk.Aws.REGION` — resolved at synth time |
| **L2 construct** | `s3.Bucket` (not `s3.CfnBucket`) — gives you `grantRead()`, `addEventNotification()` |
| **Removal Policy** | `RETAIN` for prod, `DESTROY` for dev |
| **autoDeleteObjects** | CDK auto-synthesizes a Lambda + custom resource to empty the bucket before deletion |
| **CfnOutput** | Exports values visible in CloudFormation / usable by other stacks |
| **IAM grants** | `dataBucket.grantRead(role)` — auto-generates the policy |
| **Resource policy** | `addToResourcePolicy()` — adds a statement to the bucket's policy |
| **Event notifications** | `addEventNotification` → SNS topic (L2 wiring, no manual Lambda) |
| **Lifecycle rules** | Storage class transitions + noncurrent version expiry |
| **KMS encryption** | Customer-managed key with rotation, `bucketKeyEnabled` for cost |
| **Nested construct (helper method)** | `createAccessLogsBucket()` — DRY, still in the same stack |

---

### 5. CDK CLI Workflow

```bash
# 1. Bootstrap (once per account+region)
cdk bootstrap aws://123456789012/ap-south-1

# 2. Synthesize (validate & inspect the template)
cdk synth

# 3. Diff against deployed stack
cdk diff

# 4. Deploy
cdk deploy

# 5. Pass context values
cdk deploy --context environment=production --context retentionDays=180

# 6. Tear down
cdk destroy
```

---

### 6. What `cdk synth` Actually Produces

Running `cdk synth` on the stack above generates (simplified):

```
cdk.out/
├── S3DataStack.template.json   ← CloudFormation template
├── S3DataStack.assets.json
├── asset.abc123/               ← auto-created empty-bucket Lambda zip
└── manifest.json
```

The template contains resources like:
- `AWS::S3::Bucket` (data + logs)
- `AWS::KMS::Key`
- `AWS::SNS::Topic`
- `AWS::IAM::Role` + policies
- `AWS::Lambda::Function` (for auto-delete)
- `AWS::CloudFormation::CustomResource` (orchestrates auto-delete)

---

### 7. Testing (Jest)

```typescript
// test/s3-data-stack.test.ts
import * as cdk from 'aws-cdk-lib';
import { Template } from 'aws-cdk-lib/assertions';
import { S3DataStack } from '../lib/s3-data-stack';

test('production bucket has versioning + KMS', () => {
  const stack = new S3DataStack(new cdk.App(), 'Test', {
    environment: 'production',
    retentionDays: 90,
  });

  const template = Template.fromStack(stack);

  template.hasResourceProperties('AWS::S3::Bucket', {
    VersioningConfiguration: { Status: 'Enabled' },
    BucketEncryption: {
      ServerSideEncryptionConfiguration: [
        {
          ServerSideEncryptionByDefault: {
            SSEAlgorithm: 'aws:kms',
          },
        },
      ],
    },
  });
});
```

Run with `npx jest`. This catches regressions before `cdk deploy`.

---

### TL;DR

| You write | CDK does |
|---|---|
| `new s3.Bucket(this, 'X', { ... })` | Emits `AWS::S3::Bucket` + all child resources (policy, lifecycle, notifications, IAM) into the CFN template |
| `bucket.grantRead(role)` | Auto-generates the IAM policy statement |
| `cdk synth` | Walks the tree, resolves tokens, writes `template.json` |
| `cdk deploy` | Uploads assets → calls CloudFormation → resources exist in AWS |

The entire power of CDK is that **you write TypeScript that *describes* infrastructure**, and the toolkit handles all the CloudFormation boilerplate, dependency ordering, and deployment mechanics for you.

=================================================
Interview questions 
## Scenario-Based Interview Q&A: CDK Resource Import & Multi-Account Management

---

## Part A: Importing Existing Resources into CDK (TypeScript)

---

### Q1: "We have an S3 bucket created manually in production. We want to bring it under CDK management. Walk me through the process."

**Answer:**

There are **three mechanisms** to bring existing resources under CDK, depending on the goal:

| Mechanism | Goal | Lifecycle Control |
|---|---|---|
| `from*` methods | **Reference** an existing resource (read-only) | None — CDK won't manage it |
| `cdk import` | **Adopt** the resource into the stack | Full — CDK manages updates/deletes |
| `--import-existing-resources` | **Adopt** in CI/CD (non-interactive) | Full — automated |

**Step-by-step for `cdk import` (full adoption):**

```typescript
// lib/prod-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class ProdStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1. Add the construct matching the EXISTING resource's state
    //    Must set removalPolicy: RETAIN to avoid accidental deletion
    new s3.Bucket(this, 'ExistingBucket', {
      bucketName: 'mycompany-prod-data-2024',  // must match physical name
      versioned: true,                          // must match current state
      removalPolicy: cdk.RemovalPolicy.RETAIN,  // CRITICAL
    });
  }
}
```

```bash
# 2. Verify no other pending changes
cdk diff

# 3. Run the import (interactive — prompts for physical name)
cdk import ProdStack
# CLI asks: "What is the name of the S3 bucket to import?"
# You type: mycompany-prod-data-2024
```

**For CI/CD (non-interactive):**

```bash
# Option A: Generate a mapping file first
cdk import ProdStack --record-resource-mapping mapping.json
# Edit mapping.json to fill in physical IDs
cdk import ProdStack --resource-mapping mapping.json

# Option B: Use deploy with flag (matches by custom physical name)
cdk deploy ProdStack --import-existing-resources
```

**Key interview points to mention:**
- The construct must **exactly model** the resource's current state (no drift)
- `removalPolicy: RETAIN` is mandatory — otherwise `cdk destroy` deletes the resource
- Only **additions** are allowed in the diff at import time; no other changes
- `cdk import` uses CloudFormation's `ResourcesToImport` API under the hood

---

### Q2: "We need to reference a VPC that was created by a different team's stack. How do you handle this?"

**Answer:**

This is a **reference** (not adoption) scenario — use `from*` methods:

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as kms from 'aws-cdk-lib/aws-kms';

// ── Same account, same region ──────────────────────────────────────────

// Option 1: Look up by attribute (synth-time lookup, requires env)
const vpc = ec2.Vpc.fromLookup(this, 'ExistingVpc', {
  vpcName: 'shared-vpc',   // or isDefault: true
});

// Option 2: Import by VPC ID (no lookup needed)
const vpc = ec2.Vpc.fromVpcAttributes(this, 'ImportedVpc', {
  vpcId: 'vpc-0abc123def',
  availabilityZones: ['us-east-1a', 'us-east-1b'],
  privateSubnetIds: ['subnet-111', 'subnet-222'],
  privateSubnetNames: ['private-a', 'private-b'],
});

// ── Cross-account ──────────────────────────────────────────────────────

// Option 3: Import by ARN (no CloudFormation export needed)
const bucket = s3.Bucket.fromBucketName(this, 'OtherAccountBucket',
  'other-account-bucket-name'
);

const role = iam.Role.fromRoleArn(this, 'OtherAccountRole',
  'arn:aws:iam::999988887777:role/MyRole'
);

const key = kms.Key.fromKeyArn(this, 'OtherAccountKey',
  'arn:aws:kms:us-east-1:999988887777:key/abc-123'
);
```

**Critical rule:** `from*` methods create a **read-only reference**. CDK will NOT update or delete the resource. You can still call `grantRead()`, `grantWrite()` on the imported object to generate IAM policies.

---

### Q3: "What's the difference between `cdk import` and `cdk deploy --import-existing-resources`? When would you use each?"

**Answer:**

| Aspect | `cdk import` | `cdk deploy --import-existing-resources` |
|---|---|---|
| **Mode** | Interactive (prompts for names) | Non-interactive (matches by physical name) |
| **Matching** | You tell it which resource | Auto-matches template resources to unmanaged resources by **custom physical name** |
| **Use case** | One-off migration, manual work | CI/CD pipelines, automated adoption |
| **Prerequisite** | Resource must NOT be in any CFN stack | Resource must NOT be in any CFN stack AND must have a custom name |
| **Can mix with new resources?** | No — only additions allowed | Yes — imports matches, creates the rest |

**When to use `--import-existing-resources`:**
```bash
# In a CI pipeline, first-time adoption
cdk deploy MyStack --import-existing-resources
```
This is ideal when you've written the CDK code with explicit `bucketName`, `tableName`, etc. that match the existing resources — CDK will import them instead of trying to create new ones.

---

### Q4: "We have a legacy CloudFormation template with 50 resources. How do we bring it into CDK?"

**Answer:**

Use **`CfnInclude`** to import the entire template:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as cfninc from 'aws-cdk-lib/cloudformation-include';
import { Construct } from 'constructs';

export class LegacyStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const template = new cfninc.CfnInclude(this, 'LegacyTemplate', {
      templateFile: 'legacy-template.json',  // path to existing CFN template
    });

    // Access individual resources as L1 constructs
    const bucket = template.getResource('MyBucket') as cdk.aws_s3.CfnBucket;
    console.log(bucket.bucketName);

    // You can now add L2 constructs that reference these
    // or gradually refactor L1 → L2 over time
  }
}
```

**Migration strategy (interview gold):**
1. **Phase 1**: `CfnInclude` the whole template → deploy as-is (zero risk)
2. **Phase 2**: Gradually extract resources into L2 constructs, removing them from the template
3. **Phase 3**: Delete `CfnInclude` once all resources are native CDK

---

## Part B: Multi-Account CDK App Management

---

### Q5: "We have 3 AWS accounts (Dev: 111111111111, Staging: 222222222222, Prod: 333333333333). How should we structure the CDK app?"

**Answer:**

Use **`cdk.Stage`** (formerly `cdk.Environment`) to group stacks per environment:

```typescript
// lib/app-stage.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';

export class AppStageProps extends cdk.StageProps {
  // Custom props for environment-specific config
  readonly bucketPrefix: string;
  readonly enableVersioning: boolean;
}

export class AppStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props: AppStageProps) {
    super(scope, id, props);

    // Create stacks INSIDE the stage
    new NetworkStack(this, 'Network', {
      vpcCidr: '10.0.0.0/16',
      environment: id,
    });

    new DataStack(this, 'Data', {
      bucketPrefix: props.bucketPrefix,
      versioned: props.enableVersioning,
    });

    new ComputeStack(this, 'Compute', {
      instanceType: id === 'Prod' ? 'm5.2xlarge' : 't3.medium',
    });
  }
}
```

```typescript
// bin/my-app.ts
import * as cdk from 'aws-cdk-lib';
import { AppStage } from '../lib/app-stage';

const app = new cdk.App();

// Each Stage targets a different account
new AppStage(app, 'Dev', {
  env: { account: '111111111111', region: 'ap-south-1' },
  bucketPrefix: 'myapp-dev',
  enableVersioning: false,
});

new AppStage(app, 'Staging', {
  env: { account: '222222222222', region: 'ap-south-1' },
  bucketPrefix: 'myapp-staging',
  enableVersioning: true,
});

new AppStage(app, 'Prod', {
  env: { account: '333333333333', region: 'ap-south-1' },
  bucketPrefix: 'myapp-prod',
  enableVersioning: true,
});
```

**Deployment:**
```bash
# Deploy only the Dev stage
cdk deploy Dev/*

# Deploy only Prod
cdk deploy Prod/*

# Deploy everything
cdk deploy --all
```

**Key principles:**
- **One Stage per account** — the Stage's `env` is inherited by all child stacks
- **Parameterize** the stack via props (never hardcode account IDs inside stacks)
- **Never use `this.account` in conditional logic** — it's a Token at synth time and comparisons silently fail

---

### Q6: "How do you handle cross-account references? E.g., the Prod stack needs to read from a shared S3 bucket in the Dev account."

**Answer:**

**Critical rule:** CDK's built-in cross-stack references (`CfnOutput` + `Fn.importValue`) **only work within the same account and region**. For cross-account, you must use **string-based references**:

```typescript
// ── Approach 1: Pass ARN/ID as a string prop ──────────────────────────

interface DataStackProps extends cdk.StackProps {
  sharedBucketArn: string;  // concrete string, not a CDK object
}

export class DataStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: DataStackProps) {
    super(scope, id, props);

    // Import by name/ARN — no CloudFormation export needed
    const sharedBucket = s3.Bucket.fromBucketName(
      this, 'SharedBucket',
      props.sharedBucketArn.split('/').pop()!  // extract name from ARN
    );

    // Or if you know the name directly:
    // const sharedBucket = s3.Bucket.fromBucketName(this, 'SharedBucket', 'myapp-dev-shared');

    // Grant cross-account access
    sharedBucket.grantRead(
      new iam.AccountPrincipal(this.account)  // current account
    );
  }
}

// In the stage:
new DataStack(this, 'Data', {
  sharedBucketArn: 'arn:aws:s3:::myapp-dev-shared',
});
```

```typescript
// ── Approach 2: SSM Parameter Store (recommended for polyrepo) ────────

import * as ssm from 'aws-cdk-lib/aws-ssm';

// Producer stack (in Dev account) writes the value
new ssm.StringParameter(this, 'SharedBucketParam', {
  parameterName: `/shared/myapp/bucket-name`,
  stringValue: myBucket.bucketName,
});

// Consumer stack (in Prod account) reads it at deploy time
const bucketName = ssm.StringParameter.valueForStringAttribute(
  this, 'SharedBucketParam', '/shared/myapp/bucket-name'
);
const sharedBucket = s3.Bucket.fromBucketName(this, 'Shared', bucketName);
```

**Comparison:**

| Pattern | Scope | Survives Stack Delete? | Best For |
|---|---|---|---|
| Direct props (string ARN/ID) | Cross-account | ✅ | Same-repo, known values |
| `CfnOutput` + `Fn.importValue` | Same account/region only | ❌ | Same-account stack references |
| SSM Parameter Store | Cross-account, cross-region | ✅ | Polyrepo, dynamic values |
| Secrets Manager | Cross-account (sensitive) | ✅ | Credentials, keys |

---

### Q7: "How do you bootstrap multiple accounts for CDK deployment?"

**Answer:**

```bash
# 1. Bootstrap each target account (once)
cdk bootstrap aws://111111111111/ap-south-1
cdk bootstrap aws://222222222222/ap-south-1
cdk bootstrap aws://333333333333/ap-south-1

# 2. For cross-account deployments (central pipeline → target accounts):
#    Bootstrap with --trust pointing to the deployment account
cdk bootstrap aws://333333333333/ap-south-1 \
  --trust 444444444444 \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/PowerUserAccess
```

**CI/CD pattern (central pipeline deploys to all accounts):**

```
┌─────────────────────────────────────────────────────────────┐
│  Central Account (444444444444)                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  CodePipeline / GitHub Actions                        │  │
│  │  1. cdk synth                                        │  │
│  │  2. cdk deploy Dev/*   (trust → 111111111111)       │  │
│  │  3. cdk deploy Staging/* (trust → 222222222222)     │  │
│  │  4. cdk deploy Prod/*    (trust → 333333333333)     │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │  Dev     │        │ Staging  │        │  Prod    │
   │ 1111...  │        │ 2222...  │        │ 3333...  │
   │ (trust   │        │ (trust   │        │ (trust   │
   │  4444...)│        │  4444...)│        │  4444...)│
   └──────────┘        └──────────┘        └──────────┘
```

**Credential flow:**
```bash
# Developer authenticates via IAM Identity Center (SSO)
aws sso login --profile myorg-dev

# CDK uses the profile to assume bootstrap roles in target accounts
cdk deploy Prod/* --profile myorg-dev
```

**Bootstrap via StackSets (for many accounts):**
```bash
# Bootstrap all accounts in an OU at once
cdk bootstrap \
  --toolkit-stack-name cdk-toolkit \
  aws://ou-abc123:ap-south-1 \
  --trust 444444444444
```

---

### Q8: "What's the recommended project structure for a multi-account CDK app?"

**Answer:**

```
my-organization/
├── bin/
│   └── app.ts                 ← Entry point (creates Stages)
├── lib/
│   ├── stages/
│   │   ├── app-stage.ts       ← Stage class (groups stacks per env)
│   │   └── stage-props.ts     ← Shared stage props
│   ├── stacks/
│   │   ├── network-stack.ts   ← VPC, subnets, security groups
│   │   ├── data-stack.ts      ← S3, RDS, DynamoDB
│   │   ├── compute-stack.ts   ← ECS, Lambda, EC2
│   │   └── monitoring-stack.ts← CloudWatch, ALB
│   ├── constructs/
│   │   ├── secured-bucket.ts  ← Reusable L3 construct
│   │   └── api-gateway.ts     ← Reusable L3 construct
│   └── config/
│       ├── dev.config.ts      ← Dev-specific parameters
│       ├── staging.config.ts  ← Staging-specific parameters
│       └── prod.config.ts     ← Prod-specific parameters
├── test/
│   ├── stacks/
│   └── constructs/
├── cdk.json
├── package.json
└── tsconfig.json
```

**Config-driven approach:**

```typescript
// lib/config/prod.config.ts
export const prodConfig = {
  environment: 'production',
  vpcCidr: '10.0.0.0/16',
  instanceType: 'm5.2xlarge',
  enableVersioning: true,
  removalPolicy: 'retain',
  monitoring: {
    alarmThreshold: 95,
    notificationEmail: 'ops@company.com',
  },
};
```

```typescript
// bin/app.ts
import { prodConfig } from '../lib/config/prod.config';

new AppStage(app, 'Prod', {
  env: { account: '333333333333', region: 'ap-south-1' },
  ...prodConfig,
});
```

---

### Q9: "What are the common pitfalls in multi-account CDK setups?"

**Answer:**

| Pitfall | Why it breaks | Fix |
|---|---|---|
| Comparing `this.account` in `if` statements | Returns a **Token** at synth time; `===` always fails | Pass environment as a **string prop** |
| Passing a CDK object (e.g., `IVpc`) cross-account | CDK creates a CFN Export/Import which is **regional + account-scoped** | Pass the **string ID/ARN** and use `from*` methods |
| Forgetting `env` on a stack | CDK generates an **environment-agnostic** template; `Vpc.fromLookup` and `cdk.Aws.ACCOUNT_ID` won't resolve | Always set `env: { account, region }` |
| Stage `env` vs Stack `env` conflict | If a child stack defines a different `env`, it deploys to a different account than the Stage — often an accidental bug | Keep consistent: either Stage defines it, or every Stack defines it |
| Not bootstrapping target accounts | `cdk deploy` fails with "No CDK toolkit" | Run `cdk bootstrap aws://<account>/<region>` first |
| Using `--trust` incorrectly | Trust is the account that **deploys** (pipeline account), not the target | `--trust <deployer-account-id>` |
| Stateful resources in same stack as stateless | Destroying the stack for a Lambda update also deletes the RDS | **Separate stacks by lifecycle** — stateful resources in their own stack with `terminationProtection` |

---

### Q10: "How would you implement a promotion workflow (Dev → Staging → Prod) in a CDK pipeline?"

**Answer:**

```typescript
// lib/pipeline-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as pipeline from 'aws-cdk-lib/aws-codepipeline';
import * as actions from 'aws-cdk-lib/aws-codepipeline-actions';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Artifact bucket for the pipeline
    const artifactBucket = new s3.Bucket(this, 'ArtifactBucket', {
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    const sourceOutput = new pipeline.Artifact();
    const cdkOut = new pipeline.Artifact();

    const pipeline = new pipeline.Pipeline(this, 'PromotionPipeline', {
      pipelineName: 'MyApp-Promotion',
      artifactBucket,
      crossAccountKeys: true,  // REQUIRED for multi-account CDK deploys
      stages: [
        // ── Source ──────────────────────────────────────────────────
        {
          stageName: 'Source',
          actions: [
            new actions.CodeStarConnectionsSourceAction({
              actionName: 'GitSource',
              connectionArn: 'arn:aws:codestar-connections:ap-south-1:444444444444:connection/xxx',
              repository: 'myorg/my-app',
              output: sourceOutput,
            }),
          ],
        },
        // ── Build + Synth ───────────────────────────────────────────
        {
          stageName: 'Build',
          actions: [
            new actions.CodeBuildAction({
              actionName: 'Synth',
              project: new cdk.aws_codebuild.Project(this, 'BuildProject', {
                buildSpec: cdk.aws_codebuild.BuildSpec.fromObject({
                  version: '0.2',
                  phases: {
                    build: {
                      commands: ['npm ci', 'npm run build', 'npx cdk synth'],
                    },
                  },
                  artifacts: { files: ['cdk.out/**'] },
                }),
              }),
              input: sourceOutput,
              outputs: [cdkOut],
            }),
          ],
        },
        // ── Deploy to Dev (auto) ────────────────────────────────────
        {
          stageName: 'DeployDev',
          actions: [
            new actions.CdkDeployAction({
              actionName: 'DeployDev',
              project: new cdk.aws_codebuild.Project(this, 'DeployDevProject', {
                buildSpec: cdk.aws_codebuild.BuildSpec.fromObject({
                  version: '0.2',
                  phases: {
                    build: { commands: ['npx cdk deploy Dev/* --require-approval never'] },
                  },
                }),
              }),
              input: cdkOut,
            }),
          ],
        },
        // ── Deploy to Staging (manual approval) ─────────────────────
        {
          stageName: 'ApproveStaging',
          actions: [
            new actions.ManualApprovalAction({
              actionName: 'Approve',
              additionalInformation: 'Review Dev results before promoting to Staging',
            }),
          ],
        },
        {
          stageName: 'DeployStaging',
          actions: [
            new actions.CdkDeployAction({
              actionName: 'DeployStaging',
              project: new cdk.aws_codebuild.Project(this, 'DeployStagingProject', {
                buildSpec: cdk.aws_codebuild.BuildSpec.fromObject({
                  version: '0.2',
                  phases: {
                    build: { commands: ['npx cdk deploy Staging/* --require-approval never'] },
                  },
                }),
              }),
              input: cdkOut,
            }),
          ],
        },
        // ── Deploy to Prod (manual approval) ────────────────────────
        {
          stageName: 'ApproveProd',
          actions: [
            new actions.ManualApprovalAction({
              actionName: 'Approve',
              additionalInformation: 'Final approval for production deployment',
            }),
          ],
        },
        {
          stageName: 'DeployProd',
          actions: [
            new actions.CdkDeployAction({
              actionName: 'DeployProd',
              project: new cdk.aws_codebuild.Project(this, 'DeployProdProject', {
                buildSpec: cdk.aws_codebuild.BuildSpec.fromObject({
                  version: '0.2',
                  phases: {
                    build: { commands: ['npx cdk deploy Prod/* --require-approval never'] },
                  },
                }),
              }),
              input: cdkOut,
            }),
          ],
        },
      ],
    });
  }
}
```

**Key points:**
- `crossAccountKeys: true` is **mandatory** for multi-account CDK pipelines (creates a KMS key each account can use)
- Each deploy step runs in a CodeBuild project with credentials for the **target account**
- Manual approval gates between environments enforce the promotion workflow
- The same `cdk.out` artifact is promoted — no re-synth at each stage (deterministic)

---

## Quick Reference: Interview Cheat Sheet

| Topic | Key Command / API |
|---|---|
| Reference existing (no lifecycle) | `s3.Bucket.fromBucketName()`, `ec2.Vpc.fromVpcAttributes()` |
| Adopt into stack (interactive) | `cdk import <Stack>` |
| Adopt into stack (CI) | `cdk deploy --import-existing-resources` |
| Import CFN template | `new cfninc.CfnInclude(this, 'T', { templateFile })` |
| Multi-env structure | `cdk.Stage` with `env: { account, region }` |
| Cross-account reference | Pass **string ARN/ID** + `from*` method |
| Cross-account shared value | SSM Parameter Store |
| Bootstrap target | `cdk bootstrap aws://<acct>/<region> --trust <deployer>` |
| Pipeline cross-account | `crossAccountKeys: true` in `Pipeline` |

