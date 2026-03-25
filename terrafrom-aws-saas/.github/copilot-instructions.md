# Copilot Instructions for AWS-Production-Infrastructure-with-terraform

## 1) Big Picture
- Repo is Terraform-only (no app code). Root module is `terrafrom-aws-saas/` with reusable modules under `terrafrom-aws-saas/modules/`.
- Major components:
  - `modules/vpc/` defines VPC, subnets (2 public + 2 private), NAT gateway (single AZ for MVP), route tables, VPC flow logs (CloudWatch + KMS).
  - `modules/security/` defines security groups (ALB + app), IAM role/profile, S3 bucket for ALB logs, kms encryption for logs.
  - `modules/alb/` defines Application Load Balancer with HTTP->HTTPS redirect, HTTPS listener (ACM cert), target group health checks on `/health`.
  - `modules/asg/` defines Graviton2 (arm64) AMI with launch template + ASG + scaling policy + optional scheduled scaling.

## 2) Service boundaries and dataflows
- Internet enters through ALB in public subnets.
- ALB routes traffic to ASG instances in private subnets on port 8080.
- App instances use IAM instance role for CloudWatch logs and SSM Parameter Store read.
- VPC flow logs go to `/aws/vpc/{project}-flow-logs` CloudWatch group.
- ALB logs go to bucket `${project}-alb-logs-${random_suffix}` (block public access + SSE-KMS).

## 3) CI/CD and developer workflow
- Primary GitHub Action: `terrafrom-aws-saas/.github/workflows/terraform.yaml`.
- Steps: checkout -> AWS creds -> setup terraform -> `terraform fmt -check -recursive` -> `terraform init -backend-config=bucket=$TF_STATE_BUCKET` -> `terraform validate` -> `terraform plan` on PR -> `terraform apply` on `main`.
- Security scanning: Checkov (bridgecrew) on all Terraform.
- Required secrets: `AWS_ROLE_ARN`, `TF_STATE_BUCKET`, `GITHUB_TOKEN`.

## 4) Local dev commands (mirrors pipeline)
- `cd terrafrom-aws-saas`
- `terraform fmt -recursive`
- `terraform init -backend-config="bucket=<bucket>"`
- `terraform validate -no-color`
- `terraform plan -no-color -input=false`
- `terraform apply -auto-approve -input=false` (main only)

## 5) Project-specific patterns
- `user_data` rendered via `templatefile()` in `modules/asg/main.tf` and passed as base64.
- Tag propagation (`dynamic "tag"` with `var.common_tags`) in ASG plus override `lifecycle.ignore_changes = [desired_capacity]`.
- Hard-coded numbers explicitly (desired capacity 2, min 2, max 10; health check grace 300). Maintain these as defaults unless variable names exist.
- ALB listener intentionally enforces TLS 1.2+ via `ELBSecurityPolicy-TLS13-1-2-2021-06`.

## 6) Non-obvious integration and constraints
- ACM cert is DNS validated in `modules/alb/main.tf` (`aws_acm_certificate`). In a full deployment, DNS record creation is not defined here (must be pre-created or added later).
- Single NAT gateway is chosen for cost optimization, not HA. New AZ-specific NATs should be added if requiring production HA.
- `aws_eip` depends on IGW so no creation sequencing issues.

## 7) Spot checks for editing and review
- If editing ASG, verify no conflicting `instance_type` change with arm64 AMI filter in `modules/asg/main.tf`.
- If editing network resources, reproduce with `terraform plan` in the same state bucket and region (`us-east-1`).
- Keep `terraform state` in S3 as per workflow, do not mutate local state without awareness.

## 8) Questions for clarification
- Is there an expected variable file pattern (`*.tfvars`) in this environment or should we use inline `-var` from the project context?
- Should each module be validated independently (e.g., `terraform validate` per module) or as root module only?
