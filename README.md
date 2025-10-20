# terraform-atlantis-pipeline

**Terraform based CI Pipeline**

<img width="796" height="506" alt="Atlantis1 drawio" src="https://github.com/user-attachments/assets/0505758b-ec00-4296-8b1c-edd7c23d9295" />

**Q.1:  How to explain a Terraform-based infrastructure production pipeline in an interview?**

**Ans:** In our setup, Terraform deployments are fully automated through Atlantis and Terragrunt. 
Each environment (dev, qa, prod) has its own Terragrunt configuration, and state files are isolated in an S3 backend.

Developers never run Terraform apply locally. They push code to GitLab, which triggers static validation and tfsec scans. Atlantis handles the plan/apply workflow via pull (or merge) requests on the GitLab CI, ensuring auditability and approvals.

This ensures environmental consistency, eliminates manual errors, and provides a clear approval workflow, a key production-grade practice.

**Q.2: Why do we need Terragrunt when we have Terraform?**

**Ans:** Terraform is the main Infrastructure-as-Code (IaC) tool from HashiCorp.

**It lets you:**
Define infrastructure as code using .tf files
Create reproducible environments
Manage infrastructure lifecycle (init, plan, apply, destroy)
Store state (locally or remotely)

But when you start using Terraform in a real production setup with multiple environments, modules, and teams, you quickly hit challenges.

**Problems with Terraform working alone:**

**Some real-world pain points when using plain Terraform:**

Repetition Across Environments:

You often have separate folders like dev, qa, prod, each repeating similar code with only a few variables changed.

**State File Management**

Managing remote state configurations per environment (like S3 backend and DynamoDB lock) becomes repetitive.

**Complexity with Modules**
Reusing shared modules across environments can lead to long, error-prone path references.

**Team Collaboration**
Without automation, teams manually run terraform plan and terraform apply, which introduces human errors and inconsistent changes.

**Terragrunt: The Terraform Wrapper or Terraform Orchestrator**

Terragrunt is a thin wrapper around Terraform, designed specifically to reduce repetition and manage multi-environment Terraform setups.

Think of it as: Terraform + structure, DRY (Don't Repeat Yourself) principle, and automation.

**Atlantis: The Automation Layer**

Now, you’ve automated the structure (with Terragrunt).

The next problem: who runs Terraform commands, and how do we control it?

This is where Atlantis comes in.

**What Atlantis Does?**

Atlantis is a self-hosted service that automates Terraform commands through Git workflows.

**In short:**
It lets you run terraform plan and terraform apply directly from your GitLab or GitHub Pull or Merge Requests.

**How Atlantis Works?**

A developer opens a Merge Request (MR) in GitLab with Terraform changes.

The Atlantis webhook gets triggered.

Atlantis runs:

terragrunt plan

and posts the plan result as a comment in the MR.

Reviewer checks and comments:

atlantis apply

Atlantis then securely runs:

terragrunt apply

and applies it in the correct environment.

All actions are logged, visible in the MR, and auditable; no one runs Terraform apply manually.

Let us understand with an example (Note: This is not a hands-on demo).

**Terraform folder structure:**

<img width="247" height="492" alt="image" src="https://github.com/user-attachments/assets/f0d64e9e-3a8e-47e2-be63-4080befed753" />

**Limitations with Terraform:**

1. backend.tf (Repeated in every environment)
terraform {
  backend "s3" {
    bucket = "my-terraform-states"
    key    = "dev/terraform.tfstate"  # this changes per env
    region = "us-east-1"
  }
}

2. provider.tf (Repeated as well in some cases)

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0" # Specifies a compatible update constraint for AWS provider
    }
    random = {
      source  = "hashicorp/random"
      version = "3.1.0" # Specifies an exact version for the Random provider
    }
  }
  required_version = ">= 1.2.0" # Specifies the minimum required Terraform CLI version
}

provider "aws" {
  region = "us-east-1" # Configuration specific to the AWS provider
}

3. modules

module "vpc" {
  source = "../../modules/vpc"
  cidr_block   = "10.0.0.0/16"
  subnet_cidr  = "10.0.1.0/24"
}

# The pain point: Terraform can’t automatically pass outputs from one module to another

# You must manually wire them:

module "ec2" {

  source       = "../../modules/ec2"
  
  subnet_id    = module.vpc.subnet_id
  
  instance_name = "dev-instance"
}


**In Short:**
Terraform with .tfvars helps parameterize environments,
but Terragrunt helps structure, scale, and orchestrate multi-environment, multi-module infrastructure.

**So:**
If you have one or two modules → Terraform + .tfvars is fine.
If you manage 10+ modules across 3 environments → Terragrunt saves you time and mistakes.

**Terragrunt Folder structure:**

<img width="302" height="555" alt="image" src="https://github.com/user-attachments/assets/10db7ee7-2ee8-4bb2-b6e5-257d5a9ddd15" />

Root infra/terragrunt.hcl

Shared config for all environments and modules — sets up remote state and provider.

# infra/terragrunt.hcl

remote_state {

  backend = "s3"
  
  config = {
  
    bucket = "my-terraform-states"
    
    key    = "${path_relative_to_include()}/terraform.tfstate"
    
    region = "us-east-1"
  }
}

generate "provider" {

  path      = "provider.tf"
  
  if_exists = "overwrite"
  
  contents  = <<EOF
  
provider "aws" {

  region = "us-east-1"
}
EOF
}

Dev Environment:

infra/envs/dev/vpc/terragrunt.hcl

include "root" {

  path = find_in_parent_folders()
}

terraform {

  source = "../../../modules/vpc"
  
}

inputs = {

  cidr_block   = "10.0.0.0/16"
  
  subnet_cidr  = "10.0.1.0/24"
  
}

infra/envs/dev/ec2/terragrunt.hcl

include "root" {

  path = find_in_parent_folders()
}

terraform {

  source = "../../../modules/ec2"
}

dependencies {

  paths = ["../vpc"]
  
}

inputs = {

  instance_name = "dev-ec2"
  
  subnet_id     = dependency.vpc.outputs.subnet_id
  
}

**Commands:**

Go to your environment folder and run:

cd infra/envs/dev

terragrunt run-all init

terragrunt run-all plan

terragrunt run-all apply
