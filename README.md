# Terraform AWS Two-Tier Infrastructure

**Author:** Saurab Poudel

This project uses Terraform to provision a two-tier web application infrastructure on AWS. It creates a VPC across two Availability Zones, public web subnets, private database subnets, an Application Load Balancer, an EC2 Auto Scaling group, an Aurora MySQL database, and a CloudFront distribution protected by AWS WAF.

## Architecture

- **Network:** VPC with two public and two private subnets across `us-east-1a` and `us-east-1b`, an Internet Gateway, two NAT Gateways, and route tables.
- **Web tier:** Public Application Load Balancer forwarding HTTP traffic to an EC2 Auto Scaling group. The launch template uses an AMI selected by name and runs the script in `modules/aws-autoscaling/deploy.sh` to install Nginx and serve a sample site.
- **Database tier:** Aurora MySQL cluster and instances in private subnets, with database access restricted by security group rules.
- **Access and scaling:** An EC2 IAM role and instance profile for Systems Manager access, plus CloudWatch CPU alarms and scaling policies.
- **Edge and DNS:** CloudFront, an ACM DNS-validated certificate, Route 53 records for the apex and `www` names, and a CloudFront-scoped WAF web ACL.

## Repository Layout

```text
.
├── backend.tf
├── main.tf
├── variables.tf
├── variables.tfvars
└── modules/
	├── alb-tg/
	├── aws-autoscaling/
	├── aws-iam/
	├── aws-rds/
	├── aws-vpc/
	├── aws-waf-cdn-acm-route53/
	└── security-group/
```

## Prerequisites

- Terraform `>= 1.0.0` and the AWS provider `>= 5.0.0`.
- AWS credentials configured for an identity allowed to create the resources in this project. The configuration creates networking, EC2, RDS, IAM, CloudFront, WAF, ACM, and Route 53 resources.
- An existing public Route 53 hosted zone for the domain configured in `variables.tfvars`.
- The S3 state bucket and DynamoDB lock table configured in `backend.tf` must already exist, and the AWS identity must be able to access them.
- An AMI whose name matches the configured `AMI-NAME` and is available in the target region.

The current VPC module explicitly places subnets in `us-east-1a` and `us-east-1b`. The ACM certificate used by CloudFront must be created in `us-east-1`.

## Configuration

Set resource names, CIDR ranges, the domain, and other inputs in `variables.tfvars`. Update the domain to one managed by the AWS account's public Route 53 hosted zone. Ensure the configured AMI name matches an AMI available in the deployment region.

Do not store real database credentials in version control. The example `variables.tfvars` contains a sample password; replace it with a strong secret supplied through a secure mechanism, and avoid committing secret values.

## Deploy

From the repository root, initialize Terraform, review the proposed changes, and apply them:

```sh
terraform init
terraform validate
terraform plan -var-file="variables.tfvars"
terraform apply -var-file="variables.tfvars"
```

Review the plan carefully before applying. This configuration can create billable resources, including NAT Gateways, load balancers, CloudFront, and Aurora database instances.

> **Path note:** The current root `main.tf` refers to modules using `../modules`, but the modules in this repository are under `./modules`. The Auto Scaling module's user-data file reference also uses `../modules/...`. These paths may need to be changed to repository-relative module paths before Terraform can initialize and apply successfully from the repository root.

## Destroy

When the infrastructure is no longer needed, review the destroy plan and remove the resources:

```sh
terraform plan -destroy -var-file="variables.tfvars"
terraform destroy -var-file="variables.tfvars"
```

The RDS cluster is configured to skip its final snapshot on deletion. Back up any data you need before destroying the infrastructure.
