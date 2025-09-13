# From IaC-to Guardrails Building Secure and Scalable Infrastructure

Authored by Mark Mallia 

---

## 1. Why IaC + Guardrails?  

In today’s cloud‑first world, *Infrastructure as Code* (IaC) is the only way to keep your environments reproducible, auditable and cost‑effective.  
But IaC alone isn’t enough—your teams also need **guardrails**: consistent policy enforcement, secure secrets handling, and proactive drift detection.

---

## 2. Architecture Overview  

* **GitHub** – Source of truth for modules, terragrunt configs, and policies.  
* **Terragrunt** – Orchestrator that pulls all modules into a single pipeline per account.  
* **Remote‑state** (S3) – Keeps state files out of your local machine so teams never fight over locks.  
* **OPA** – Reusable guardrails to enforce naming, tagging, cost‑management, and drift detection.

---

## 3. Terraform Modules & Terragrunt  

### 3.1 Terraform Module: VPC + Subnets

```hcl
# modules/vpc/main.tf
variable "region" {}
variable "env" {}

resource "aws_vpc" "main" {
  cidr_block = "10.${var.env}.0.0/16"

  tags = merge(
    { Environment = var.env },
    { Project      = "IaC-Guardrails" }
  )
}

module "public_subnet" {
  source   = "./modules/subnet"
  subnet_cidr = "10.${var.env}.1.0/24"
}
```

> **Tip:** Keep modules small, testable, and version‑controlled.  
> *note:* “You can see the module variables and tags—easy to audit.”

### 3.2 Terragrunt Configuration

```hcl
# terragrunt.hcl
locals {
  envs = ["dev", "prod"]
}

remote_state {
  backend = "s3"
  config {
    bucket         = "iac-guardrails-state"
    key            = "${path_relative_to_include}/terraform.tfstate"
    region         = var.region
  }
}

include = [
  for e in local.envs : { path = "./${e}" }
]

run_all = [
  {
    name   = "OPA Policy",
    command= "opa eval -i ./terraform.tfstate -d ./policies/vpc.rego"
  }
]
```

*What it does:*  
- Defines a single S3 bucket for all remote‑states.  
- Uses Terragrunt’s `include` to spin up the same VPC module on every account/region.

> *note:* “The code shows you can reuse modules across accounts with minimal duplication.”

---

## 4. Remote State & Multi‑Account Strategy  

### 4.1 S3 Bucket Policy

```json
# terragrunt-s3-policy.json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{"AWS":"arn:aws:iam::111122223333:user/IaC-Engineer"},
      "Action":["s3:*"],
      "Resource":["arn:aws:s3:::iac-guardrails-state/*"]
    }
  ]
}
```

### 4.2 Account Map

| Account | Region | Terraform Root |
|---------|--------|----------------|
| `111122223333` | `us-east-1` | `terragrunt/us-east-1/` |
| `444455556666` | `eu-west-1` | `terragrunt/eu-west-1/` |

> *note:* “Clear mapping means fewer mistakes when deploying across accounts.”

---

## 5. Policy as Code with OPA  

OPA (Open Policy Agent) lets you write declarative checks against your IaC outputs.

### 5.1 Rego Policy File

```rego
# policies/vpc.rego
package iac.guardrails.vpc

default allow = false

allow {
  input.aws_vpc.main.tags.Project == "IaC-Guardrails"
  input.aws_vpc.main.tags.Environment == input.env
}
```

### 5.2 Terragrunt Hook to Run OPA

```hcl
# terragrunt.hcl (continued)
run_all = [
  { 
    name   = "OPA Policy",
    command= "opa eval -i ./terraform.tfstate -d ./policies/vpc.rego"
  }
]
```

> * note:* “You can see the policy is evaluated on every run, ensuring your infra matches standards.”

---

## 6. Secrets, Drift Detection & Ops  

### 6.1 AWS Parameter Store for Sensitive Variables

```hcl
# terragrunt.hcl (continued)
remote_state {
  backend = "ssm"
  config {
    path   = "/iac/guardrails/${var.env}"
    region = var.region
  }
}
```

### 6.2 Drift Detection Using Terragrunt’s `run-all`  

Terragrunt automatically checks for drift in all modules and runs OPA on every pipeline:

```bash
terragrunt run-all --terragrunt-log-level=info
```

> *note:* “The single command covers all accounts, making your CI/CD simple.”



---

## 7. Take‑away 

* IaC + guardrails = repeatable, auditable infra.  
* Terraform modules keep code DRY; Terragrunt orchestrates cross‑account deployments.  
* OPA lets you enforce policies without manual review.  
* Remote state in S3 + Parameter Store keeps secrets secure and drift out of the way.  

