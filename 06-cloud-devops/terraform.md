# 🏗️ Infrastructure as Code — Terraform

> Terraform's entire behaviour follows from one thing: **the state file**. Understand what state is, why it exists, and how it gets out of sync, and every confusing Terraform moment resolves.

**Prerequisite:** [AWS](aws.md)

---

## 📑 Contents

1. [The Mental Model](#1-the-mental-model)
2. [State — the Heart of It](#2-state--the-heart-of-it)
3. [Core Language](#3-core-language)
4. [Modules](#4-modules)
5. [Multi-Environment Patterns](#5-multi-environment-patterns)
6. [Secrets](#6-secrets)
7. [Importing Existing Infrastructure](#7-importing-existing-infrastructure)
8. [Team Workflow](#8-team-workflow)
9. [Testing & Policy](#9-testing--policy)
10. [Common Failure Modes](#10-common-failure-modes)
11. [Alternatives](#11-alternatives)
12. [Interview Section](#12-interview-section)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. The Mental Model

```
   ⭐ TERRAFORM RECONCILES THREE THINGS

   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │   CONFIG     │    │    STATE     │    │    REALITY   │
   │  (your .tf)  │    │ (what TF     │    │ (what's      │
   │              │    │  thinks      │    │  actually in │
   │  DESIRED     │    │  exists)     │    │  the cloud)  │
   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
          │                   │                   │
          └───────────────────┴───────────────────┘
                              │
                         terraform plan
                              │
                              ▼
                    "here's the diff, and
                     here's what I'll do"

   ⭐ EVERY WEIRD TERRAFORM PROBLEM IS THESE THREE DISAGREEING.
     • Config ≠ State  → a normal plan; apply it
     • State ≠ Reality → DRIFT (someone changed it manually)
     • State lost      → Terraform wants to recreate everything ⚠️
```

```
   THE EXECUTION FLOW

   terraform init      download providers, configure the backend
        ▼
   terraform plan      refresh state from reality → build a
        │              dependency graph → compute the diff
        ▼
   terraform apply     execute the graph, respecting dependencies
        │              and parallelism
        ▼
   state updated
```

---

## 2. State — the Heart of It

#### 💬 Why state exists at all

```
   Terraform needs to answer: "is this resource mine, and what
   did it look like last time?"

   It cannot answer that from the cloud API alone, because:
     • It must know which resources it MANAGES (vs ones a human
       created) — otherwise `destroy` would delete everything
     • It must map your logical name (aws_instance.web) to the
       real ID (i-0abc123)
     • It must track dependencies to order create/destroy
     • Refreshing everything from the API on every run would be
       impossibly slow at scale

   ⭐ SO: state is a JSON file mapping your config to real
     resource IDs, plus a cache of their last-known attributes.
```

```
   ⚠️⚠️ STATE CONTAINS SECRETS IN PLAINTEXT

   Database passwords, generated keys, certificate private keys —
   anything a resource returns as an attribute lands in state
   unencrypted.

   ⭐ CONSEQUENCES:
     • NEVER commit state to git
     • Encrypt the backend at rest
     • Restrict who can read the state bucket as tightly as
       you'd restrict production database access
     • Treat a leaked state file as a credential compromise
```

### Remote state — non-negotiable for teams

```hcl
terraform {
  required_version = ">= 1.9"

  backend "s3" {
    bucket       = "acme-tfstate"
    key          = "prod/network/terraform.tfstate"   # ⭐ one key per
    region       = "us-east-1"                        #   component
    encrypt      = true
    use_lockfile = true            # ⭐ S3 native locking (TF 1.10+),
                                   #   replaces the DynamoDB table
  }

  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.60" }
  }
}
```

```
   ⭐ WHY LOCKING MATTERS
     Two engineers running apply simultaneously against the same
     state produces corrupted state and duplicated or orphaned
     resources. The lock makes the second one wait.

   ⚠️ If someone Ctrl-C's mid-apply, the lock can be left behind:
     terraform force-unlock <LOCK_ID>
     ⭐ But VERIFY nobody is actually running first — force-unlocking
       during a live apply is how you get real corruption.
```

### State splitting — the scaling decision

```
   ⚠️ ONE GIANT STATE FILE IS THE MOST COMMON TERRAFORM MISTAKE

   Symptoms:
     • `plan` takes 15+ minutes (it refreshes everything)
     • Every change locks the whole organization's infrastructure
     • ⭐ A mistake anywhere can destroy anything
     • Teams block each other constantly

   ⭐ SPLIT BY BLAST RADIUS AND CHANGE FREQUENCY

   ┌──────────────────────────────────────────────────────────────┐
   │ networking/     VPC, subnets, TGW — changes rarely,          │
   │                 breaks everything if wrong                   │
   │ data/           RDS, S3, ElastiCache — ⭐ stateful, most       │
   │                 dangerous to touch                           │
   │ platform/       EKS cluster, IAM roles, shared services      │
   │ apps/<service>/ per-service resources — changes constantly   │
   └──────────────────────────────────────────────────────────────┘

   ⭐ RULE OF THUMB: things with different change frequencies or
     different blast radii belong in different states.
```

```hcl
# Reading another state's outputs — the composition mechanism
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "acme-tfstate"
    key    = "prod/network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
}
```

⚠️ This creates a **dependency between states**. Destroying the network state while apps reference it breaks them. Prefer data sources with tags or SSM parameters for looser coupling in large organizations.

---

## 3. Core Language

```hcl
# ── VARIABLES ─────────────────────────────────────────────────
variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {                          # ⭐ fail fast on bad input
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}

variable "instance_config" {
  type = object({                       # ⭐ typed objects beat
    type     = string                   #   loose maps
    count    = number
    monitoring = optional(bool, true)   # optional with a default
  })
}

# ── LOCALS ────────────────────────────────────────────────────
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {                       # ⭐ tag everything, always
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.team
  }
}

# ── RESOURCES ─────────────────────────────────────────────────
resource "aws_instance" "app" {
  ami           = data.aws_ami.al2023.id
  instance_type = var.instance_config.type

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-app" })

  lifecycle {
    create_before_destroy = true        # ⭐ avoid a gap during replacement
    prevent_destroy       = true        # ⭐ guard for stateful resources
    ignore_changes        = [ami]       # ⚠️ use sparingly — hides drift
  }
}
```

### for_each vs count — an important distinction

```hcl
# ❌ count — index-based, and therefore FRAGILE
resource "aws_instance" "app" {
  count = 3
  # aws_instance.app[0], [1], [2]
}
# ⚠️ Removing the MIDDLE item shifts every subsequent index,
#   so Terraform destroys and recreates them all.

# ✅ for_each — key-based and STABLE
resource "aws_instance" "app" {
  for_each      = toset(["web", "api", "worker"])
  instance_type = "t3.micro"
  tags          = { Name = each.key }
  # aws_instance.app["web"], ["api"], ["worker"]
}
# ⭐ Removing "api" affects ONLY "api". Nothing else moves.
```

```
   ⭐ USE for_each BY DEFAULT. Use count only for a simple
     on/off toggle:

   count = var.enable_monitoring ? 1 : 0
```

### Meta-arguments and expressions

```hcl
# depends_on — ⭐ only when the dependency is IMPLICIT
resource "aws_instance" "app" {
  depends_on = [aws_iam_role_policy.app]   # needed at boot, not referenced
}
# ⚠️ Terraform infers dependencies from references automatically.
#   Explicit depends_on is a code smell unless genuinely hidden.

# Dynamic blocks for repeated nested blocks
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.port
    to_port     = ingress.value.port
    protocol    = "tcp"
    cidr_blocks = ingress.value.cidrs
  }
}

# Useful functions
try(var.optional_value, "default")
coalesce(var.a, var.b, "fallback")
lookup(var.map, "key", "default")
one(aws_instance.app[*].id)              # single value from a 0-or-1 list
[for s in var.subnets : s.id]            # list comprehension
{ for k, v in var.map : k => upper(v) }  # map comprehension
```

---

## 4. Modules

```
   ⭐ A MODULE IS A REUSABLE, PARAMETERIZED GROUP OF RESOURCES.

   modules/
     vpc/
       main.tf         resources
       variables.tf    inputs
       outputs.tf      ⭐ the module's public API
       versions.tf     provider constraints
       README.md
```

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.13"                    # ⭐ ALWAYS pin

  name = "${local.name_prefix}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"   # ⭐ save money in dev

  tags = local.common_tags
}
```

```
   ⭐ MODULE DESIGN PRINCIPLES

   □ One clear purpose. A "vpc" module, not a "networking-and-
     compute-and-database" module.
   □ ⭐ Sensible defaults, minimal required inputs. If every
     variable is required, the module isn't providing value.
   □ Outputs are the public API — treat them as a contract you
     can't casually break.
   □ ⚠️ Don't hardcode provider configuration inside a module;
     let the caller pass it.
   □ Version and tag modules; consume by version, not by branch.
   □ ⚠️ Avoid deep nesting. Modules calling modules calling
     modules becomes impossible to debug or reason about.
```

```
   ⚠️ THE OVER-ABSTRACTION TRAP
     A module wrapping a single resource with pass-through
     variables adds indirection and provides nothing. Modules
     should encapsulate a MEANINGFUL GROUP of resources plus
     the conventions around them (naming, tagging, sensible
     defaults, wiring between them).
```

---

## 5. Multi-Environment Patterns

```
   ┌──────────────────────────────────────────────────────────────┐
   │ ① SEPARATE DIRECTORIES  ⭐ recommended                         │
   │    environments/                                             │
   │      dev/      main.tf, terraform.tfvars, backend.tf         │
   │      staging/                                                │
   │      prod/                                                   │
   │    ✅ Explicit, greppable, ⭐ prod can genuinely differ        │
   │    ✅ Separate state and separate permissions per env         │
   │    ❌ Some duplication (mitigated by shared modules)          │
   ├──────────────────────────────────────────────────────────────┤
   │ ② TERRAFORM WORKSPACES                                       │
   │    terraform workspace new prod                              │
   │    ⚠️ Same config, different state. Prod and dev CANNOT       │
   │      structurally differ — only via conditionals, which      │
   │      gets ugly fast.                                         │
   │    ⚠️ ⭐ Dangerously easy to apply to the wrong workspace.     │
   │      Use for ephemeral/preview environments, NOT for the     │
   │      dev/staging/prod split.                                 │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ TERRAGRUNT                                                 │
   │    A wrapper providing DRY backend config, dependency        │
   │    ordering, and run-all across many states.                 │
   │    ✅ Genuinely helps at scale (many states, many accounts)   │
   │    ❌ Another tool and abstraction layer to learn            │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE PRINCIPLE THAT MATTERS MORE THAN THE PATTERN
     Environments should be as similar as possible, differing
     only in scale and configuration values. When prod has a
     fundamentally different SHAPE from staging, staging stops
     predicting production and its value collapses.
```

---

## 6. Secrets

```
   ⚠️ NEVER PUT SECRETS IN .tf FILES OR .tfvars COMMITTED TO GIT.
   ⚠️ AND REMEMBER: whatever a resource returns lands in STATE
     in plaintext regardless.
```

```hcl
# ✅ Reference from a secrets manager at apply time
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
}

# ✅✅ BETTER: let the provider generate and manage it, so the
#     value never passes through your config at all
resource "aws_db_instance" "main" {
  manage_master_user_password = true    # ⭐ AWS manages it in
                                        #   Secrets Manager, rotates it,
                                        #   and it never enters your code
}
```

```
   ⭐ THE SECRETS HIERARCHY, BEST TO WORST

   1. ⭐ The resource manages its own secret (RDS-managed
      password, IAM roles, workload identity) — nothing to leak
   2. Reference a secrets manager at apply time
   3. Environment variables (TF_VAR_db_password) from CI secrets
   4. Encrypted files (SOPS) committed to git
   5. ⚠️ Plaintext in tfvars — never

   ⭐ And regardless of which you pick: encrypt the state backend
     and lock down read access to it.
```

---

## 7. Importing Existing Infrastructure

```hcl
# ⭐ Import blocks (TF 1.5+) — declarative, reviewable, and
#   they generate config for you
import {
  to = aws_instance.app
  id = "i-0abc123def456"
}

# Generate matching config:
#   terraform plan -generate-config-out=generated.tf
```

```bash
# The older imperative form (still works)
terraform import aws_instance.app i-0abc123def456
```

```
   ⭐ THE IMPORT WORKFLOW
     1. Write (or generate) the resource config
     2. Import it into state
     3. ⭐ RUN PLAN — it MUST show no changes
        If it wants to modify something, your config doesn't
        match reality yet. Fix the config, not the resource.
     4. Repeat

   ⚠️ Importing a large existing estate is slow and tedious.
     Tools like Terraformer can bulk-generate, but the output
     always needs substantial cleanup.
```

### State manipulation — the escape hatches

```bash
terraform state list                    # what's tracked
terraform state show aws_instance.app   # attributes of one resource

# ⭐ Rename/move a resource WITHOUT destroying it
terraform state mv aws_instance.app aws_instance.web
terraform state mv 'module.old' 'module.new'

# ⭐ BETTER (TF 1.1+): declarative moved blocks — reviewable in a PR
moved {
  from = aws_instance.app
  to   = aws_instance.web
}

# ⚠️ Stop managing a resource WITHOUT destroying it
terraform state rm aws_instance.app

# ⭐ Force replacement on the next apply
terraform apply -replace=aws_instance.app
```

```
   ⚠️ ALWAYS BACK UP STATE BEFORE MANUAL MANIPULATION
     terraform state pull > backup.tfstate
     One wrong `state rm` and Terraform forgets it manages a
     production database — then a later apply tries to create
     a second one, or a `destroy` misses it entirely.
```

---

## 8. Team Workflow

```
   ⭐ THE PR-BASED FLOW

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. Open a PR with .tf changes                                │
   │ 2. CI runs: fmt check · validate · tflint · ⭐ tfsec/Checkov  │
   │ 3. CI runs `terraform plan` and ⭐ POSTS IT AS A PR COMMENT   │
   │    → the diff is reviewed like code                          │
   │ 4. Policy checks (OPA/Sentinel) gate dangerous changes       │
   │ 5. Human approval — ⭐ required for prod                      │
   │ 6. Merge triggers `terraform apply` with the SAVED PLAN      │
   └──────────────────────────────────────────────────────────────┘
```

```yaml
# The essential CI shape
- run: terraform fmt -check -recursive
- run: terraform init -backend=false
- run: terraform validate
- run: tflint
- run: tfsec .                        # or checkov
- run: terraform plan -out=tfplan     # ⭐ SAVE the plan
- run: terraform show -no-color tfplan > plan.txt
# → post plan.txt as a PR comment
# on merge:
- run: terraform apply tfplan         # ⭐ apply the EXACT reviewed plan
```

```
   ⭐ WHY `apply` MUST USE THE SAVED PLAN FILE
     If you re-plan at apply time, you're applying something
     nobody reviewed. Between the review and the apply, state
     or reality may have changed. Applying the saved plan
     guarantees that what was approved is what runs — and
     Terraform will refuse if the world has shifted underneath it.
```

```
   ⚠️ NEVER APPLY FROM A LAPTOP TO PRODUCTION
     • No audit trail of who changed what
     • Local provider versions differ from everyone else's
     • Uncommitted local changes get applied
     • ⭐ The person applying needs prod credentials on their
       machine, which is exactly what you're trying to avoid
```

---

## 9. Testing & Policy

```hcl
# ⭐ Native testing (TF 1.6+) — .tftest.hcl
run "validates_cidr" {
  command = plan

  variables {
    vpc_cidr = "10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR did not match input"
  }
}

run "creates_three_subnets" {
  command = apply              # ⭐ actually creates, then destroys

  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "expected 3 private subnets"
  }
}
```

```
   THE TESTING LADDER

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. fmt + validate         syntax and internal consistency    │
   │ 2. tflint                 provider-specific lint, deprecated │
   │                           arguments, invalid instance types  │
   │ 3. ⭐ tfsec / Checkov      SECURITY misconfigurations —       │
   │                           public S3, unencrypted volumes,    │
   │                           open security groups               │
   │ 4. terraform test         unit tests on plan, and real       │
   │                           create/destroy for integration     │
   │ 5. OPA / Sentinel         ⭐ ORGANIZATIONAL POLICY:           │
   │                           "no public S3" · "all resources    │
   │                           tagged" · "only approved instance  │
   │                           types" · "no deletes in prod       │
   │                           without an approval label"         │
   └──────────────────────────────────────────────────────────────┘
```

```rego
# OPA policy: block any plan that would destroy an RDS instance
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_db_instance"
  resource.change.actions[_] == "delete"
  msg := sprintf("Refusing to delete RDS instance %v", [resource.address])
}
```

---

## 10. Common Failure Modes

```
   ⚠️ STATE DRIFT — someone changed something manually
     Detect: `terraform plan` shows unexpected diffs
     ⭐ Prevent: restrict console write access in prod; run a
       scheduled plan and alert on non-empty diffs
     Fix: apply to revert, OR update config to match if the
     manual change was correct

   ⚠️ STATE LOCK STUCK
     Someone Ctrl-C'd mid-apply, or a CI job was killed
     terraform force-unlock <ID>
     ⭐ VERIFY nobody is actually running first

   ⚠️ RESOURCE ALREADY EXISTS
     Created outside Terraform, or a previous apply partially
     succeeded → `terraform import` it

   ⚠️ ⭐ CYCLE ERROR
     Two resources reference each other. Break it by splitting
     into separate resources (e.g. define a security group and
     its rules as separate `aws_security_group_rule` resources)

   ⚠️ PROVIDER VERSION DRIFT
     Different engineers get different behaviour
     ⭐ Commit .terraform.lock.hcl. This is not optional.

   ⚠️ ⭐ THE ACCIDENTAL DESTROY-AND-RECREATE
     A change to an immutable attribute (some name fields,
     certain AMI or engine settings) forces replacement.
     ⭐ ALWAYS READ THE PLAN. "-/+ destroy and then create
       replacement" on a database is the line that ruins a day.
     Protect with lifecycle { prevent_destroy = true }

   ⚠️ TIMEOUT ON SLOW RESOURCES
     RDS and EKS can take 20+ minutes
     timeouts { create = "60m" }
```

```
   ⭐ THE PLAN-READING DISCIPLINE

   Terraform tells you exactly what it will do. The failure is
   almost always that nobody read it carefully.

     +   create           (usually fine)
     ~   update in place  (usually fine)
     -   destroy          ⚠️ read carefully
     -/+ REPLACE          ⚠️⚠️ destroy then create — on stateful
                            resources this is DATA LOSS

   ⭐ In CI, fail the build automatically if a plan touching
     `aws_db_instance` or `aws_s3_bucket` includes a delete,
     unless a specific approval label is present.
```

---

## 11. Alternatives

```
   ┌────────────────┬─────────────────────────────────────────────┐
   │ TERRAFORM/     │ ⭐ Declarative HCL, huge provider ecosystem, │
   │ OPENTOFU       │ multi-cloud. The default choice.            │
   │                │ (OpenTofu is the open-source fork after the │
   │                │  BSL license change — API-compatible.)      │
   ├────────────────┼─────────────────────────────────────────────┤
   │ PULUMI         │ Real programming languages (TS, Python, Go).│
   │                │ ✅ Loops, conditionals, abstractions, and    │
   │                │   real testing frameworks                   │
   │                │ ⚠️ ⭐ Also: the full power to write           │
   │                │   incomprehensible infrastructure code      │
   ├────────────────┼─────────────────────────────────────────────┤
   │ CDK / CDKTF    │ Programming languages that SYNTHESIZE       │
   │                │ CloudFormation or Terraform                 │
   ├────────────────┼─────────────────────────────────────────────┤
   │ CLOUDFORMATION │ AWS-native. ⭐ No state file to manage —      │
   │                │ AWS tracks it. But AWS-only, verbose, and   │
   │                │ slower to support new features than TF.     │
   ├────────────────┼─────────────────────────────────────────────┤
   │ ANSIBLE        │ ⚠️ Different category — configuration        │
   │                │ management, not provisioning. Procedural    │
   │                │ and only eventually idempotent.             │
   └────────────────┴─────────────────────────────────────────────┘
```

```
   ⭐ THE DECLARATIVE-VS-IMPERATIVE POINT WORTH MAKING
     Terraform's HCL constrains you deliberately. That's a
     feature: infrastructure code is read far more often than
     written, usually under pressure during an incident, often
     by someone who didn't write it. A general-purpose language
     lets you build abstractions nobody else can follow.

     Pulumi is genuinely better when you need real logic and
     real unit tests. It's genuinely worse when a team uses
     that power without discipline.
```

---

## 12. Interview Section

<details>
<summary><b>Q1. What is Terraform state and why does it exist?</b></summary>

State maps your configuration to real resource IDs and caches their last-known attributes.

It exists because Terraform has to answer questions the cloud API alone can't. Which resources does it manage, as opposed to ones a human created — otherwise `destroy` would delete everything in the account. What real ID corresponds to `aws_instance.web`. What the dependency graph is, so creates and destroys happen in the right order. And it's a performance cache, because refreshing every attribute of every resource on every run doesn't scale.

The critical operational fact is that state contains secrets in plaintext. Database passwords, generated keys, certificate private keys — anything a resource returns as an attribute. So state is never committed to git, the backend is encrypted, read access is restricted as tightly as production database access, and a leaked state file is treated as a credential compromise.

For teams, remote state with locking is mandatory. Two people applying simultaneously against the same state produces corruption and orphaned resources.
</details>

<details>
<summary><b>Q2. How do you structure Terraform for a large organization?</b></summary>

Split state by blast radius and change frequency. One giant state file is the most common serious mistake — plans take fifteen minutes because everything refreshes, every change locks the entire organization's infrastructure, and a mistake anywhere can destroy anything.

So: networking in one state since it changes rarely and breaks everything if wrong. Data resources separately because they're stateful and most dangerous. Platform-level shared services separately. Then per-service states that change constantly.

Separate directories per environment rather than workspaces. Workspaces share one configuration, so production can't structurally differ from dev without conditionals that get ugly, and it's dangerously easy to apply to the wrong workspace. Directories are explicit and greppable, and allow per-environment permissions.

Shared modules eliminate the duplication that directories introduce, with modules versioned and consumed by version rather than by branch.

At real scale — many accounts, many states — Terragrunt is worth the additional abstraction for DRY backend configuration and dependency ordering. Below that scale it's another tool to learn for limited benefit.

The principle underneath all of it: environments should differ only in scale and configuration values. When production has a fundamentally different shape from staging, staging stops predicting production.
</details>

<details>
<summary><b>Q3. `count` vs `for_each`?</b></summary>

`count` creates resources indexed by number, so they're addressed as `[0]`, `[1]`, `[2]`. The problem is that removing a middle element shifts every subsequent index, and Terraform interprets that as "destroy and recreate all of them." That's catastrophic if those are databases or stateful resources.

`for_each` keys resources by a string or map key. Removing one element affects only that element; nothing else moves.

So I'd use `for_each` by default, reserving `count` for a simple on-off toggle — `count = var.enabled ? 1 : 0`.

If you have existing `count`-based resources you want to convert, `moved` blocks let you do it declaratively without destroying anything, and they're reviewable in a pull request rather than being an out-of-band `state mv` command.
</details>

<details>
<summary><b>Q4. Someone changed infrastructure manually. Now what?</b></summary>

That's drift — state and reality disagree. `terraform plan` detects it, showing a diff you didn't expect.

Two valid resolutions. If the manual change was wrong, apply to revert it. If the manual change was correct — someone fixed something urgent during an incident — update the configuration to match, then apply to confirm no changes remain.

The important part is prevention. Restrict console write access in production so manual changes are difficult by default. Run a scheduled plan on a cron and alert on any non-empty diff, so drift is detected in hours rather than discovered during an unrelated deploy. And provide a documented break-glass process, because during a real incident people will and should make manual changes — the goal is that it's deliberate and reconciled afterward, not casual.

There's an escape hatch worth knowing but using sparingly: `ignore_changes` in a lifecycle block. It's legitimate when another system genuinely owns an attribute — an autoscaler adjusting desired capacity, for instance — but it hides drift by design, so every use should be justified.
</details>

<details>
<summary><b>Q5. How do you handle secrets in Terraform?</b></summary>

The best answer is to avoid them entirely. Let the resource manage its own secret — `manage_master_user_password` on RDS has AWS generate and rotate the password in Secrets Manager, and the value never passes through your configuration or your state.

Where that isn't possible, reference a secrets manager at apply time via a data source, so the secret lives in the secret store and Terraform reads it rather than owning it.

Below that, environment variables from CI secrets, or encrypted files with SOPS.

But the crucial point regardless of approach is that state contains secrets in plaintext anyway. Whatever a resource returns as an attribute is written to state unencrypted. So encrypting the backend, restricting read access to the state bucket, and never committing state to git aren't optional extras — they're the actual control.

I'd also avoid `terraform output` of sensitive values in CI logs, and mark outputs sensitive so they're redacted.
</details>

<details>
<summary><b>Q6. What does a safe team workflow look like?</b></summary>

Pull-request based, with plan as the review artifact.

CI runs format check, validate, tflint, and a security scanner like tfsec or Checkov. Then it runs `terraform plan` with `-out` to save the plan, renders it, and posts it as a PR comment so the infrastructure diff is reviewed exactly like code. Policy checks — OPA or Sentinel — gate dangerous changes automatically. Production requires human approval.

On merge, CI applies the saved plan file.

That last detail matters more than it looks. If you re-plan at apply time, you're applying something nobody reviewed — state or reality may have changed since the review. Applying the saved plan guarantees what was approved is what runs, and Terraform refuses if the world shifted underneath it.

And never apply to production from a laptop. There's no audit trail, local provider versions differ, uncommitted changes get applied, and the person applying needs production credentials on their machine — which is exactly what you're trying to eliminate.

The policy checks are where I'd invest most. Automatically failing any plan that deletes an RDS instance or an S3 bucket without an explicit approval label prevents the class of mistake that's genuinely unrecoverable.
</details>

<details>
<summary><b>Q7. A plan shows your production database will be replaced. What happened?</b></summary>

Something changed an attribute that can't be modified in place, so Terraform's only path is destroy-then-create. Common causes: a change to an identifier field, certain engine or storage settings, or a change to a subnet group or availability zone.

The immediate response is to not apply, and to read the plan carefully to identify which attribute triggered it — Terraform states the reason with "forces replacement" next to the attribute.

Then either revert that attribute, or find an in-place path. Sometimes there's a different argument that achieves the same outcome without replacement, and sometimes the change genuinely requires a planned migration with a snapshot and a maintenance window.

The preventive controls: `prevent_destroy` in a lifecycle block on stateful resources, which makes Terraform hard-fail rather than proceed. And a CI policy that blocks any plan containing a delete of a database or bucket unless a specific approval label is present.

The broader discipline is reading plans properly. Terraform tells you exactly what it will do — `-/+` means replacement. Nearly every Terraform disaster is someone approving a plan they didn't read.
</details>

<details>
<summary><b>Q8. Terraform vs Pulumi vs CloudFormation?</b></summary>

Terraform, or OpenTofu since the license change, is the default. Declarative HCL, by far the largest provider ecosystem, genuinely multi-cloud, and the mature choice with the most operational knowledge available.

CloudFormation is AWS-native, and its one real advantage is no state file to manage — AWS tracks it, so you can't corrupt or lose it. But it's AWS-only, verbose, and historically slower to support new services than the Terraform AWS provider.

Pulumi uses real programming languages, so you get loops, conditionals, genuine abstractions, and real unit testing frameworks. That's a real advantage when infrastructure has actual logic in it.

The argument for Terraform's constrained language is that infrastructure code is read far more often than written — usually under pressure during an incident, often by someone who didn't write it. HCL's limitations force a legible structure. A general-purpose language lets a clever engineer build abstractions nobody else can follow, and infrastructure is a bad place for that.

So: Pulumi when the team has strong software engineering discipline and the infrastructure genuinely benefits from programmatic logic. Terraform otherwise, which is most of the time.
</details>

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                      TERRAFORM — ONE PAGE                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ RECONCILES 3 THINGS: config ↔ STATE ↔ reality                      ║
║   config≠state → normal plan · state≠reality → DRIFT ·               ║
║   state lost → wants to recreate everything                          ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️⚠️ STATE CONTAINS SECRETS IN PLAINTEXT                              ║
║   never in git · encrypt backend · lock down read access             ║
║   remote backend + LOCKING is mandatory for teams                    ║
║ ⭐ SPLIT STATE by blast radius + change frequency                     ║
║   networking / data / platform / per-app                             ║
║   ⚠️ one giant state = 15-min plans, org-wide locks, huge blast radius║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ for_each (key-based, STABLE) over count (index-based, FRAGILE)     ║
║   removing a middle count element recreates ALL subsequent ones      ║
║   count only for on/off: count = var.enabled ? 1 : 0                 ║
║ moved {} blocks to rename/restructure without destroying             ║
╠══════════════════════════════════════════════════════════════════════╣
║ ENVIRONMENTS: separate DIRECTORIES, not workspaces                   ║
║   ⚠️ workspaces = same config, easy to apply to the wrong one         ║
║ MODULES: one purpose · sensible defaults · pin versions ·            ║
║   ⚠️ don't wrap a single resource, don't nest deeply                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ SECRETS: ⭐ best = the resource manages its own                       ║
║   (manage_master_user_password) → then secrets manager data source   ║
║   → env vars → SOPS.  ⚠️ NEVER plaintext tfvars                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ WORKFLOW: PR → fmt/validate/tflint/tfsec → plan POSTED AS COMMENT →  ║
║   policy gate (OPA) → approval → apply the SAVED PLAN FILE           ║
║   ⭐ re-planning at apply time means applying something unreviewed    ║
║ ⚠️ NEVER apply to prod from a laptop                                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ READ THE PLAN. -/+ means DESTROY AND RECREATE.                    ║
║   On a database that's data loss. Use prevent_destroy + CI policy    ║
║   blocking deletes of stateful resources.                            ║
║ COMMIT .terraform.lock.hcl · back up state before any state mv/rm    ║
║ terraform force-unlock <ID> (⭐ verify nobody is running first)       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [AWS](aws.md) · [CI/CD](cicd.md) · [Kubernetes](kubernetes.md)
