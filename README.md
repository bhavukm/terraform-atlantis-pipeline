# terraform-atlantis-pipeline

**Terraform based CI Pipeline**

<img width="782" height="517" alt="Atlantinew-latest" src="https://github.com/user-attachments/assets/3cf7786a-000a-4d0d-b778-5bec2296a6f2" />

**Q.1:  How to explain a Terraform-based infrastructure production pipeline in an interview?**

**Ans:** In our setup, Terraform deployments are fully automated through Atlantis and Terragrunt. 

Each environment (dev, qa, prod) has its own Terragrunt configuration, and state files are isolated in an S3 backend.

Developers never run Terraform apply locally. They push code to GitLab, which triggers static validation and tfsec scans. Atlantis handles the plan/apply workflow via pull (or merge) requests on the GitLab CI, ensuring auditability and 

approvals.

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
   
<img width="592" height="172" alt="image" src="https://github.com/user-attachments/assets/ce867bd3-a6dc-4e0e-b78f-41a4fcc14fd4" />

2. provider.tf (Repeated as well in some cases)

<img width="847" height="418" alt="image" src="https://github.com/user-attachments/assets/0a8837b1-db32-44c0-980a-0abc09fbed8f" />

3. modules

<img width="312" height="127" alt="image" src="https://github.com/user-attachments/assets/d464bfff-dff6-4e3b-89fc-3ac78e1548ff" />

# The pain point: Terraform can’t automatically pass outputs from one module to another

# You must manually wire them:

<img width="376" height="140" alt="image" src="https://github.com/user-attachments/assets/a6b32776-2540-4fb2-8fde-a5cf3530847f" />

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

<img width="491" height="752" alt="image" src="https://github.com/user-attachments/assets/5d71b5b4-5d32-416b-89ae-ed1ac272c02a" />

**Dev Environment:**

infra/envs/dev/vpc/terragrunt.hcl

<img width="491" height="752" alt="image" src="https://github.com/user-attachments/assets/2c36d8c7-552e-4969-8196-7af8814548b5" />

**Commands to use terragrunt locally:**

Go to your environment folder and run:

cd infra/envs/dev

terragrunt run-all init

terragrunt run-all plan

terragrunt run-all apply

On the GitLab UI, the stage names could be as follows:

<img width="1100" height="476" alt="image" src="https://github.com/user-attachments/assets/f8758818-2e16-4f4a-a391-a1d5103a4cb7" />

Final step to apply terragrunt-based terraform apply is via a comment on the raised MR section:

Example comments to deploy the infrastructure:

# Runs apply for all unapplied plans from this pull request.

atlantis apply

# Runs apply in the root directory of the repo with workspace `default`.

atlantis apply -d .

# Runs apply in the `project1` directory of the repo with workspace `default`

atlantis apply -p project1

# Runs apply in the root directory of the repo with workspace `staging`

atlantis apply -w staging

Source: https://www.runatlantis.io/docs/using-atlantis

