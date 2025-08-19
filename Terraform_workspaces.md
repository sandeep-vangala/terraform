### Consolidated Explanation: Terraform Workspaces with Reusable Code

**Objective**: Use a single Terraform codebase to manage dev, stage, and prod environments for an application using AWS resources (VPC, EKS, RDS, IAM, SG) with Terraform workspaces and modules to ensure reusability and state isolation.

**Terraform Workspaces**: Workspaces allow multiple state files for the same `.tf` configuration, enabling isolated deployments (e.g., `terraform/state/dev/terraform.tfstate`). They’re ideal for similar environments, with differences handled via variables.

**Terraform Modules**: Modules (e.g., `modules/vpc`) encapsulate reusable resource definitions (VPC, EKS, etc.). They’re called from the root `main.tf` with environment-specific variables (e.g., via `dev.tfvars`), ensuring DRY code.

**How They Work Together**: 
- **Modules** provide reusable code for components (VPC, EKS, RDS).
- **Workspaces** isolate state per environment (dev, stage, prod) using the same `.tf` files.
- **Variables** (via `.tfvars`) customize module behavior (e.g., instance sizes, CIDRs).
- In production, an S3 backend with workspace-aware state keys (e.g., `terraform/state/${terraform.workspace}/terraform.tfstate`) ensures state separation, with CI/CD and IAM controls for safety.

**Production Workflow**:
1. Initialize: `terraform init` (sets up S3 backend, downloads modules).
2. Create workspaces: `terraform workspace new dev/stage/prod`.
3. Apply per env: `terraform workspace select dev && terraform apply -var-file=tfvars/dev.tfvars`.
4. State is stored separately in S3, preventing conflicts.

**Production Considerations**:
- **State Security**: Use S3 + DynamoDB for locking; avoid local state.
- **Access Control**: Restrict S3 bucket and Terraform IAM roles; require approvals for prod changes.
- **Secrets**: Store sensitive vars (e.g., RDS passwords) in AWS SSM.
- **CI/CD**: Automate applies with pipelines (e.g., GitHub Actions).
- **Limitations**: Workspaces are best for similar envs. For divergent architectures, use separate directories.

**Modules and Workspaces Together**: Yes, they’re complementary. Modules ensure code reusability; workspaces provide state isolation. For example, a single `main.tf` calls the VPC module for all envs, with workspaces separating dev and prod state.

### Folder Structure
```
terraform-project/
├── modules/                  # Reusable modules
│   ├── vpc/                  # VPC: Network foundation
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── eks/                  # EKS: Kubernetes cluster
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── rds/                  # RDS: Database instance
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── iam/                  # IAM: Roles for EKS
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   └── sg/                   # SG: Security groups for EKS, RDS
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
├── tfvars/                   # Env-specific variables
│   ├── dev.tfvars
│   ├── stage.tfvars
│   └── prod.tfvars
├── main.tf                   # Root: Calls modules
├── variables.tf              # Root: Declares variables
├── outputs.tf                # Root: Outputs for env
├── backend.tf                # S3 backend with workspace-aware key
├── versions.tf               # Terraform/provider versions
└── README.md                 # Usage instructions
```

### Root Configuration Files
These files reside at the root (`terraform-project/`) and orchestrate the modules with workspaces.

**main.tf**


```
provider "aws" {
  region = var.region
}

module "vpc" {
  source              = "./modules/vpc"
  region              = var.region
  environment         = var.environment
  cidr_block          = var.cidr_block
  public_subnet_cidrs = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  availability_zones  = var.availability_zones
}

module "iam" {
  source      = "./modules/iam"
  region      = var.region
  environment = var.environment
}

module "sg" {
  source      = "./modules/sg"
  region      = var.region
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  db_port     = var.db_port
}

module "eks" {
  source              = "./modules/eks"
  region              = var.region
  environment         = var.environment
  subnet_ids          = module.vpc.private_subnet_ids
  security_group_ids  = [module.sg.eks_security_group_id]
  cluster_role_arn    = module.iam.cluster_role_arn
  node_role_arn       = module.iam.node_role_arn
  node_instance_type  = var.node_instance_type
  node_desired_size   = var.node_desired_size
  node_max_size       = var.node_max_size
  node_min_size       = var.node_min_size
}

module "rds" {
  source              = "./modules/rds"
  region              = var.region
  environment         = var.environment
  subnet_ids          = module.vpc.private_subnet_ids
  security_group_ids  = [module.sg.rds_security_group_id]
  db_engine           = var.db_engine
  db_engine_version   = var.db_engine_version
  db_instance_class   = var.db_instance_class
  db_allocated_storage = var.db_allocated_storage
  db_username         = var.db_username
  db_password         = var.db_password
  multi_az            = var.environment == "prod" ? true : false
}
```


```
**variables.tf**
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "List of public subnet CIDRs"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "List of private subnet CIDRs"
  type        = list(string)
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "node_instance_type" {
  description = "EKS node instance type"
  type        = string
}

variable "node_desired_size" {
  description = "Desired number of EKS nodes"
  type        = number
}

variable "node_max_size" {
  description = "Maximum number of EKS nodes"
  type        = number
}

variable "node_min_size" {
  description = "Minimum number of EKS nodes"
  type        = number
}

variable "db_engine" {
  description = "RDS database engine"
  type        = string
}

variable "db_engine_version" {
  description = "RDS database engine version"
  type        = string
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
}

variable "db_allocated_storage" {
  description = "RDS storage size in GB"
  type        = number
}

variable "db_username" {
  description = "RDS admin username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "RDS admin password"
  type        = string
  sensitive   = true
}

variable "db_port" {
  description = "RDS database port"
  type        = number
}
```
**outputs.tf**

```
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "eks_cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "rds_endpoint" {
  description = "RDS instance endpoint"
  value       = module.rds.db_endpoint
}
```

**backend.tf**
```
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "terraform/state/${terraform.workspace}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```
**versions.tf**
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0.0"
}
```
**tfvars/dev.tfvars**
```
region               = "us-east-1"
environment         = "dev"
cidr_block          = "10.0.0.0/16"
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.3.0/24", "10.0.4.0/24"]
availability_zones  = ["us-east-1a", "us-east-1b"]
node_instance_type  = "t3.medium"
node_desired_size   = 2
node_max_size       = 3
node_min_size       = 1
db_engine           = "postgres"
db_engine_version   = "13.7"
db_instance_class   = "db.t3.micro"
db_allocated_storage = 20
db_username         = "devuser"
db_password         = "devpassword" # Use SSM in prod
db_port             = 5432
```
**tfvars/prod.tfvars**
```
region               = "us-east-1"
environment         = "prod"
cidr_block          = "10.2.0.0/16"
public_subnet_cidrs = ["10.2.1.0/24", "10.2.2.0/24"]
private_subnet_cidrs = ["10.2.3.0/24", "10.2.4.0/24"]
availability_zones  = ["us-east-1a", "us-east-1b"]
node_instance_type  = "t3.large"
node_desired_size   = 4
node_max_size       = 6
node_min_size       = 2
db_engine           = "postgres"
db_engine_version   = "13.7"
db_instance_class   = "db.m5.large"
db_allocated_storage = 100
db_username         = "produser"
db_password         = "prodpassword" # Use SSM in prod
db_port             = 5432
```

### Module Files
Below are the contents of `main.tf`, `variables.tf`, and `outputs.tf` for each module (`vpc`, `eks`, `rds`, `iam`, `sg`). These are reusable across environments, with differences handled by variables.

#### 1. VPC Module (`modules/vpc/`)
Creates a VPC, public/private subnets, internet gateway, and NAT gateway.

**modules/vpc/main.tf**

```
provider "aws" {
  region = var.region
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  tags = {
    Name        = "${var.environment}-public-subnet-${count.index}"
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  tags = {
    Name        = "${var.environment}-private-subnet-${count.index}"
    Environment = var.environment
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = {
    Name        = "${var.environment}-public-rt"
    Environment = var.environment
  }
}

resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_eip" "nat" {
  count = length(var.private_subnet_cidrs) > 0 ? 1 : 0
  vpc   = true
}

resource "aws_nat_gateway" "main" {
  count         = length(var.private_subnet_cidrs) > 0 ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
  tags = {
    Name        = "${var.environment}-nat"
    Environment = var.environment
  }
}

resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs) > 0 ? 1 : 0
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }
  tags = {
    Name        = "${var.environment}-private-rt"
    Environment = var.environment
  }
}

resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[0].id
}
```

**modules/vpc/variables.tf**
```
variable "region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name (e.g., dev, stage, prod)"
  type        = string
}

variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "List of public subnet CIDRs"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "List of private subnet CIDRs"
  type        = list(string)
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}
```

**modules/vpc/outputs.tf**
```
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}
```

#### 2. EKS Module (`modules/eks/`)
Creates an EKS cluster and node group, depending on VPC and IAM modules.

**modules/eks/main.tf**
```
provider "aws" {
  region = var.region
}

resource "aws_eks_cluster" "main" {
  name     = "${var.environment}-eks"
  role_arn = var.cluster_role_arn
  vpc_config {
    subnet_ids         = var.subnet_ids
    security_group_ids = var.security_group_ids
  }
  tags = {
    Environment = var.environment
  }
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.environment}-node-group"
  node_role_arn   = var.node_role_arn
  subnet_ids      = var.subnet_ids
  instance_types  = [var.node_instance_type]
  scaling_config {
    desired_size = var.node_desired_size
    max_size     = var.node_max_size
    min_size     = var.node_min_size
  }
  tags = {
    Environment = var.environment
  }
}
```
**modules/eks/varibales.tf**
```
variable "region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "subnet_ids" {
  description = "List of subnet IDs (from VPC module)"
  type        = list(string)
}

variable "security_group_ids" {
  description = "List of security group IDs (from SG module)"
  type        = list(string)
}

variable "cluster_role_arn" {
  description = "IAM role ARN for EKS cluster"
  type        = string
}

variable "node_role_arn" {
  description = "IAM role ARN for EKS node group"
  type        = string
}

variable "node_instance_type" {
  description = "Instance type for EKS nodes"
  type        = string
}

variable "node_desired_size" {
  description = "Desired number of nodes"
  type        = number
}

variable "node_max_size" {
  description = "Maximum number of nodes"
  type        = number
}

variable "node_min_size" {
  description = "Minimum number of nodes"
  type        = number
}
```
**modules/eks/outputs.tf**

```
output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = aws_eks_cluster.main.endpoint
}

output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.main.name
}
```

#### 3. RDS Module (`modules/rds/`)
Creates an RDS instance in VPC private subnets.
**modules/rds/main.tf**
```
provider "aws" {
  region = var.region
}

resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-db-subnet-group"
  subnet_ids = var.subnet_ids
  tags = {
    Environment = var.environment
  }
}

resource "aws_db_instance" "main" {
  identifier           = "${var.environment}-rds"
  engine               = var.db_engine
  engine_version       = var.db_engine_version
  instance_class       = var.db_instance_class
  allocated_storage    = var.db_allocated_storage
  db_subnet_group_name = aws_db_subnet_group.main.name
  vpc_security_group_ids = var.security_group_ids
  username             = var.db_username
  password             = var.db_password
  multi_az             = var.multi_az
  skip_final_snapshot  = var.environment != "prod"
  tags = {
    Environment = var.environment
  }
}
```
**modules/rds/variables.tf**
variable "region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "subnet_ids" {
  description = "List of subnet IDs (from VPC module)"
  type        = list(string)
}

variable "security_group_ids" {
  description = "List of security group IDs (from SG module)"
  type        = list(string)
}

variable "db_engine" {
  description = "Database engine (e.g., postgres, mysql)"
  type        = string
}

variable "db_engine_version" {
  description = "Database engine version"
  type        = string
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
}

variable "db_allocated_storage" {
  description = "Storage size in GB"
  type        = number
}

variable "db_username" {
  description = "Database admin username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database admin password"
  type        = string
  sensitive   = true
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = false
}

**modules/rds/outputs.tf**

output "db_endpoint" {
  description = "RDS instance endpoint"
  value       = aws_db_instance.main.endpoint
}

output "db_arn" {
  description = "RDS instance ARN"
  value       = aws_db_instance.main.arn
}


#### 4. IAM Module (`modules/iam/`)
Creates IAM roles for EKS cluster and node group.

**modules/iam/main.tf**
```
provider "aws" {
  region = var.region
}

resource "aws_iam_role" "eks_cluster" {
  name = "${var.environment}-eks-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.eks_cluster.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "eks_node" {
  name = "${var.environment}-eks-node-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_node_policy" {
  role       = aws_iam_role.eks_node.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  role       = aws_iam_role.eks_node.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSCNIPolicy"
}
```
**modules/iam/variables.tf**

variable "region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

**modules/iam/outputs.tf**

output "cluster_role_arn" {
  description = "IAM role ARN for EKS cluster"
  value       = aws_iam_role.eks_cluster.arn
}

output "node_role_arn" {
  description = "IAM role ARN for EKS node group"
  value       = aws_iam_role.eks_node.arn
}


#### 5. Security Group Module (`modules/sg/`)
Creates security groups for EKS and RDS.

**modules/sg/main.tf**

```
provider "aws" {
  region = var.region
}

resource "aws_security_group" "eks" {
  name        = "${var.environment}-eks-sg"
  vpc_id      = var.vpc_id
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Restrict in prod
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Environment = var.environment
  }
}

resource "aws_security_group" "rds" {
  name        = "${var.environment}-rds-sg"
  vpc_id      = var.vpc_id
  ingress {
    from_port       = var.db_port
    to_port         = var.db_port
    protocol        = "tcp"
    security_groups = [aws_security_group.eks.id]
  }
  tags = {
    Environment = var.environment
  }
}

```
**modules/sg/varibales.tf**
```
variable "region" {
  description = "AWS region"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID (from VPC module)"
  type        = string
}

variable "db_port" {
  description = "Database port (e.g., 5432 for Postgres)"
  type        = number
}
```

**modules/sg/outputs.tf**
```
output "eks_security_group_id" {
  description = "Security group ID for EKS"
  value       = aws_security_group.eks.id
}

output "rds_security_group_id" {
  description = "Security group ID for RDS"
  value       = aws_security_group.rds.id
}
```

### Workflow with Workspaces
1. **Initialize**: `terraform init`.
2. **Create Workspaces**: `terraform workspace new dev/stage/prod`.
3. **Apply**: `terraform workspace select dev && terraform apply -var-file=tfvars/dev.tfvars`.
4. **State**: Stored in S3 (e.g., `terraform/state/dev/terraform.tfstate`).

### Interview Answer
**Q: How do you use Terraform workspaces with reusable code in production?**
A: I use modules for reusable components (VPC, EKS, RDS) in a `modules/` directory, called from a single `main.tf` with variables from `.tfvars` files (e.g., `dev.tfvars`). Workspaces isolate state per environment (dev, stage, prod) using an S3 backend with keys like `terraform/state/${terraform.workspace}/terraform.tfstate`. I initialize with `terraform init`, create workspaces with `terraform workspace new prod`, and apply with `terraform apply -var-file=prod.tfvars`. For production, I secure state with S3/DynamoDB, restrict IAM access, and automate applies via CI/CD. If envs differ significantly, I’d use separate directories instead.

