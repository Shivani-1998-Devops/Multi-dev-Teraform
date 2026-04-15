# terraform-root

Root Terraform configuration that consumes all infrastructure modules from separate repos.

## Repo Architecture

| Repo | Purpose |
|------|---------|
| [terraform-module-vpc](https://github.com/<your-org>/terraform-module-vpc) | Creates an AWS VPC |
| [terraform-module-subnet](https://github.com/<your-org>/terraform-module-subnet) | Creates subnets inside a VPC |
| [terraform-module-ec2](https://github.com/<your-org>/terraform-module-ec2) | Creates EC2 instances + Security Groups |
| [terraform-module-s3](https://github.com/<your-org>/terraform-module-s3) | Creates S3 bucket with encryption & versioning |
| **terraform-root** (this repo) | Wires all modules together per environment |

## Usage

### First-time Setup

```bash
terraform init
```

### Deploy Dev

```bash
terraform workspace new dev
terraform workspace select dev
terraform plan  -var-file="envs/dev.tfvars"
terraform apply -var-file="envs/dev.tfvars"
```

### Deploy UAT

```bash
terraform workspace new uat
terraform workspace select uat
terraform plan  -var-file="envs/uat.tfvars"
terraform apply -var-file="envs/uat.tfvars"
```

### Deploy Prod

```bash
terraform workspace new prod
terraform workspace select prod
terraform plan  -var-file="envs/prod.tfvars"
terraform apply -var-file="envs/prod.tfvars"
```

## Environment Differences

| Feature               | Dev         | UAT         | Prod        |
|-----------------------|-------------|-------------|-------------|
| VPC CIDR              | 10.0.0.0/16 | 10.1.0.0/16 | 10.2.0.0/16 |
| EC2 Instance Type     | t2.micro    | t2.small    | t3.medium   |
| SSH Access            | Open        | Open        | VPC-only    |
| Termination Protection| No          | No          | Yes         |
| S3 Versioning         | Suspended   | Suspended   | Enabled     |
| S3 force_destroy      | Yes         | Yes         | No          |

> Replace `<your-org>` in `main.tf` source URLs with your actual GitHub organisation or username.
