---
type: study-note
subject: Terraform-05-Terraform-Modules-and-Provisioners
category: devops
status: active
---

# 22 - Provisioners and Modules

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    command = "echo IP: ${self.private_ip}"
  }
  provisioner "remote-exec" {
    inline = ["sudo apt-get update"]
  }
}
```

> [!quote] Deck's explanation of Provisioners
> Execute commands or scripts on a resource after creation or before destruction. Use as a last resort - prefer provider features when possible.

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = var.vpc_cidr
}
```

> [!quote] Deck's explanation of Modules
> A container for multiple resources used together. Every Terraform configuration has at least one - the root module. Modules can call other modules, enabling reuse, packaging, and consistent patterns across teams.

## Hands-on Exercise (as stated in the deck)

> [!example] Deck's exercise outline
> Provider leads to VPC leads to 2 Subnets leads to 2 Instances per Subnet leads to Security Group (HTTPS) leads to Output Private IPs leads to Provisioner leads to Destroy.

**[EXTRA]** The deck names modules but does not show the anatomy of one. A module is simply a directory of `.tf` files with its own `variable` and `output` blocks acting as its interface - nothing more special than that:

modules/vpc/  
├── main.tf -> the actual resources  
├── variables.tf -> its inputs  
└── outputs.tf -> what it exposes to whoever calls it


```hcl
# modules/vpc/variables.tf
variable "cidr" {
  type = string
}

# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}
```

```hcl
# root config calling it, matching the deck's own module example
module "vpc" {
  source = "./modules/vpc"
  cidr   = var.vpc_cidr
}

resource "aws_subnet" "public" {
  vpc_id = module.vpc.vpc_id      # reading the module's output
}
```

**[EXTRA]** Module sources beyond a local path, since real teams rarely keep every module inside their own repository:

| Source type | Example |
|---|---|
| Local path | `source = "./modules/vpc"` |
| Git, versioned by tag | `source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.0"` |
| Terraform Registry | `source = "terraform-aws-modules/vpc/aws"`, `version = "5.0.0"` |

The Terraform Registry option is worth knowing specifically - `terraform-aws-modules/vpc/aws` is an extremely widely used, battle-tested community module handling subnets, NAT gateways, and route tables correctly, and is often a more sensible starting point than hand-rolling a VPC module from scratch for a real project.

### Self-Check Q and A

1. **Q: The deck says "every Terraform configuration has at least one module - the root module." What actually makes something the root module, structurally?**
   A: **[EXTRA]** There is no special syntax marking a directory as "the root" - it is simply the directory you run `terraform` commands from directly. Any directory referenced by a `module` block, local or remote, is structurally identical to the root; it is purely a matter of which one you are standing in when you invoke the CLI.
2. **Q: Following the deck's module example, `module "vpc" { source = "./modules/vpc" cidr = var.vpc_cidr }`, how would `aws_subnet.public` in the root config actually get the VPC's ID without hardcoding it?**
   A: Through the module's own output - `module.vpc.vpc_id`, assuming the module's `outputs.tf` exposes `vpc_id` as shown above. The module's outputs are its only interface for exposing internal values to whatever calls it.

---

# 23 - Terraform Provisioners

**Day 2 - Execute scripts on resources after creation.**

> [!quote] Deck's explanation
> Provisioners run scripts or commands on a local machine or remote server after a resource is created. Use as a last resort - prefer cloud-init or user data when possible.

```hcl
provisioner "local-exec" {
  command = "echo Instance created"
}
```

> [!quote] Deck's explanation of local-exec
> Runs a command on the machine where Terraform executes.

```hcl
provisioner "remote-exec" {
  inline = ["sudo apt update",
    "sudo apt install nginx -y"]
}
```

> [!quote] Deck's explanation of remote-exec
> Runs commands on the created resource (e.g. EC2 instance) via SSH.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install nginx -y"
    ]
  }
}
```

> [!warning] Deck's own important note
> Provisioners only run when the resource is created for the first time. If the resource has not changed on re-apply, the provisioner will NOT run again. They re-run only if the resource is destroyed and recreated.

**[EXTRA]** This last point deserves a concrete failure scenario, since it explains exactly why the deck (twice, in this section and Section 22) tells you to prefer other options first. A `remote-exec` provisioner fails partway through its `inline` command list - say, the `nginx` install step errors. The EC2 instance still exists and is marked as successfully created in state, but is only partially configured. On the next `plan`, nothing shows as needing to change, because Terraform has no concept of "partially provisioned" - it only knows the instance exists, not which of the inline commands actually completed. The broken, half-configured instance sits there indefinitely unless someone notices manually.

**[EXTRA]** The deck's own guidance, "prefer cloud-init or user data when possible," refers to this alternative for the exact same nginx-install task:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
  user_data = <<-EOF
              #!/bin/bash
              apt update
              apt install nginx -y
              EOF
}
```

`user_data` runs as part of the instance's own boot process (cloud-init), managed by the instance itself rather than requiring Terraform to hold open an SSH connection mid-apply. It automatically re-runs on any freshly launched replacement instance, and its content is a plain, visible attribute of the resource - reviewable in a `plan` diff like any other argument, unlike a provisioner's imperative, undiffable execution.

### Self-Check Q and A

1. **Q: A `remote-exec` provisioner's `inline` list has three commands. The second one fails. What state is the real EC2 instance left in, and how would you discover this from `terraform plan` alone?**
   A: **[EXTRA]** The instance exists, with the first command's effects applied and the second and third never run or only partially run - a broken, half-configured machine. You would not discover this from `plan` at all: Terraform's state shows the instance as successfully created with no further changes needed, since it has no way to represent "partially provisioned."
2. **Q: You change `instance_type` from `t2.micro` to `t2.small` on the resource in this section's final example, and re-apply. Does the `remote-exec` provisioner run again?**
   A: No - per the deck's own note, provisioners only run on initial creation, and re-run only if the resource is destroyed and recreated entirely. An in-place update, like a simple instance type change (assuming it does not force replacement), does not trigger the provisioner again.

---

# Closing

> [!quote] Deck's closing slide
> Thank You. Questions and Discussion.
> registry.terraform.io
> developer.hashicorp.com/terraform
> github.com/hashicorp/terraform

---

# Everything Beyond This Deck - What a DevOps Engineer Also Needs

**[EXTRA]** This whole section is additional to the deck, covering ground a working DevOps engineer needs that these 26 slides did not touch on at all.

## Security scanning

| Tool | Catches |
|---|---|
| `terraform validate` | Syntax and type errors only, no security awareness |
| tfsec / Trivy | Open security groups, unencrypted S3/EBS, public RDS, overly broad IAM |
| checkov | Similar security and compliance scanning, broader policy library |

```bash
tfsec .
checkov -d . --framework terraform
```

## IAM least privilege, in Terraform specifically

```hcl
resource "aws_iam_role_policy" "s3_read" {
  name = "s3-read-only"
  role = aws_iam_role.ec2_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Resource = ["arn:aws:s3:::my-app-bucket", "arn:aws:s3:::my-app-bucket/*"]
    }]
  })
}
```

Scope `Resource` to the exact ARN needed, never `"*"` "to make it work" - a policy this specific is fully visible in a pull request review, which is precisely the "coding best practices" advantage the deck itself claims for Terraform in Section 03.

## The lifecycle block

```hcl
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true    # new resource created before old one destroyed - avoids downtime
    prevent_destroy         = true    # destroy / removal REFUSES to touch this resource
    ignore_changes           = [tags] # stop fighting drift on this specific attribute
  }
}
```

## CI/CD pipeline for Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
  push:
    branches: [main]

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform fmt -check -recursive
      - run: terraform validate
      - run: terraform plan -out=tfplan

  apply:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform apply -auto-approve
```

`plan` runs on every pull request as the human-reviewed step; `apply` only runs after merge to `main`, so `-auto-approve` there is automating the mechanical step after review already happened, not skipping review itself.

## Common gotchas, gathered in one table

| Trap | Symptom | Fix |
|---|---|---|
| Renaming a resource's local name | Plan shows destroy plus create instead of a rename | `terraform state mv` first |
| Env var secret in a committed `.tfvars` | Secret leaked in git history | Use `TF_VAR_*` from CI secrets instead |
| `count` with a middle item removed | Unrelated resources show as destroy and recreate | Use `for_each` with stable keys instead |
| `-/+` on a stateful resource, unnoticed | Real data loss on apply | Always read the plan diff for `-/+` before typing yes |
| Provisioner fails partway | Instance silently left half-configured, state shows no issue | Prefer `user_data`; treat provisioners as last resort per the deck's own guidance |
| No `dynamodb_table` on an S3 backend | Two applies can corrupt state | Always pair S3 backend with a lock table |
| Hardcoded AMI ID | Config breaks in a different region or when AMI is deprecated | Use a `data "aws_ami"` lookup with filters instead |

## Functions and terraform console

```hcl
locals {
  joined   = join(",", ["a", "b", "c"])          # "a,b,c"
  merged   = merge({a = 1}, {b = 2})               # {a=1, b=2}
  policy   = jsonencode({Version = "2012-10-17"})    # common for IAM policies
}
```

Terraform has no user-defined functions - only a fixed built-in library. Reusable logic is expressed through modules instead, not custom functions. Test unfamiliar functions safely in `terraform console` before dropping them into a real config, exactly as shown in Section 16.

---
