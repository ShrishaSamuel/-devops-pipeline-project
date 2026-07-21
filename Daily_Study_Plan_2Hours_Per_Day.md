# Daily Study Plan — 2 Hours Per Day (60 Days)
# Ultimate DevOps Real-World Project — AWS Cloud (Kalyan Reddy Daida)

> **Rule:** Do not move to the next day until today's task is fully verified.
> **Each day = exactly 2 hours.** If you finish early, re-read the concept or explore the tool further.
> **If stuck:** Spend max 30 minutes debugging alone, then search the exact error message online.

---

## WEEK 1 — Linux, Git, AWS Account (Days 1–7)

---

### Day 1 — Linux CLI Essentials
**Goal:** Be comfortable navigating and inspecting a Linux system from terminal.

**Tasks (2 hours):**
- [ ] Open a terminal on your Linux machine or EC2 instance
- [ ] Practice file navigation: `pwd`, `ls -la`, `cd`, `mkdir`, `rm -rf`, `cp`, `mv`
- [ ] Practice file inspection: `cat`, `head`, `tail`, `less`, `grep`
- [ ] Practice process management: `ps aux`, `kill`, `top`, `htop`
- [ ] Practice networking basics: `curl https://google.com`, `ping 8.8.8.8`, `ss -tlnp`
- [ ] Practice environment variables: `export MY_VAR=hello`, `echo $MY_VAR`, `env | grep MY`
- [ ] Practice permissions: `chmod +x script.sh`, `chown`, `ls -la`

**Verify:**
```bash
ls -la /etc | grep hosts      # should show hosts file
curl -s https://ifconfig.me   # should return your public IP
ss -tlnp | grep 22            # should show SSH listening
```

---

### Day 2 — Git Basics
**Goal:** Understand how Git tracks changes and why it's the backbone of DevOps.

**Tasks (2 hours):**
- [ ] Install Git and configure identity:
  ```bash
  git config --global user.name "Your Name"
  git config --global user.email "you@example.com"
  ```
- [ ] Create a local repo, make commits:
  ```bash
  mkdir my-first-repo && cd my-first-repo
  git init
  echo "# My DevOps Journey" > README.md
  git add README.md
  git commit -m "initial commit"
  git log --oneline
  ```
- [ ] Create a GitHub account (if not done)
- [ ] Create a repo `aws-devops-retail-store` on GitHub
- [ ] Push your local repo to GitHub:
  ```bash
  git remote add origin https://github.com/YOUR_USER/aws-devops-retail-store.git
  git push -u origin main
  ```
- [ ] Practice: create a branch, make changes, merge back:
  ```bash
  git checkout -b feature/test
  echo "test line" >> README.md
  git add . && git commit -m "test change"
  git checkout main
  git merge feature/test
  ```

**Verify:**
```bash
git log --oneline    # shows at least 2 commits
git branch -a        # shows main and feature/test
```

---

### Day 3 — AWS Account Setup + IAM
**Goal:** Secure AWS account with non-root IAM user and billing alerts.

**Tasks (2 hours):**
- [ ] Create AWS account at aws.amazon.com (if not done)
- [ ] Enable MFA on root account (AWS Console → Account → Security credentials)
- [ ] Create billing alerts:
  - AWS Console → Billing → Budgets → Create Budget → $50 alert + $100 alert
- [ ] Create an IAM user (NOT root):
  - IAM → Users → Create user → name: `devops-admin`
  - Attach policy: `AdministratorAccess`
  - Create access key → download CSV
- [ ] Install AWS CLI:
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
  unzip awscliv2.zip && sudo ./aws/install
  aws --version
  ```
- [ ] Configure AWS CLI with your IAM user keys:
  ```bash
  aws configure
  # region: us-east-1
  # output: json
  ```

**Verify:**
```bash
aws sts get-caller-identity
# Must show your IAM user ARN — NOT root
aws s3 ls   # should return empty list (no error = credentials working)
```

---

### Day 4 — AWS Core Concepts (Read + Explore)
**Goal:** Understand Regions, AZs, EC2, S3, VPC at a conceptual level.

**Tasks (2 hours):**
- [ ] Read: What is a Region and Availability Zone?
  - Run: `aws ec2 describe-availability-zones --region us-east-1 --output table`
  - Note how many AZs are in us-east-1
- [ ] Explore EC2 in AWS Console — do NOT launch anything yet, just browse
  - Look at: Instance Types, AMIs, Security Groups, Key Pairs
- [ ] Explore S3 in AWS Console — understand bucket creation basics
- [ ] Read: What is IAM? Users vs Roles vs Policies
  - Run: `aws iam list-users` — see your devops-admin user
  - Run: `aws iam list-policies --scope AWS --max-items 5`
- [ ] Explore VPC in AWS Console — see the default VPC
  - Note the subnets, route tables, and internet gateway on the default VPC

**Verify:**
```bash
aws ec2 describe-availability-zones \
  --region us-east-1 \
  --query 'AvailabilityZones[*].ZoneName' \
  --output table
# Should list: us-east-1a, us-east-1b, us-east-1c, etc.
```

---

### Day 5 — Docker Install + First Container
**Goal:** Install Docker and run your first container.

**Tasks (2 hours):**
- [ ] Install Docker:
  ```bash
  sudo dnf install docker -y
  sudo systemctl enable --now docker
  sudo usermod -aG docker $USER
  newgrp docker
  docker --version
  ```
- [ ] Run your first container:
  ```bash
  docker run hello-world
  docker run -d --name my-nginx -p 8080:80 nginx:alpine
  curl http://localhost:8080   # should return Nginx HTML
  ```
- [ ] Inspect the running container:
  ```bash
  docker ps
  docker logs my-nginx
  docker inspect my-nginx | grep IPAddress
  docker exec -it my-nginx sh
    ls /usr/share/nginx/html
    exit
  ```
- [ ] Stop and clean up:
  ```bash
  docker stop my-nginx
  docker rm my-nginx
  docker rmi nginx:alpine
  docker ps -a   # should be empty
  ```

**Verify:**
```bash
docker ps -a | wc -l    # 1 (header only — no containers)
docker images | wc -l   # 1 (header only — no images)
```

---

### Day 6 — Dockerfile Basics
**Goal:** Write your first Dockerfile and build a custom image.

**Tasks (2 hours):**
- [ ] Create a Python app:
  ```bash
  mkdir ~/devops-app && cd ~/devops-app
  cat > app.py <<'EOF'
  from http.server import HTTPServer, BaseHTTPRequestHandler
  class Handler(BaseHTTPRequestHandler):
      def do_GET(self):
          self.send_response(200)
          self.end_headers()
          self.wfile.write(b"Hello from my DevOps container!")
      def log_message(self, *args): pass
  HTTPServer(("", 8080), Handler).serve_forever()
  EOF
  ```
- [ ] Write the Dockerfile:
  ```bash
  cat > Dockerfile <<'EOF'
  FROM python:3.11-slim
  WORKDIR /app
  COPY app.py .
  EXPOSE 8080
  CMD ["python3", "app.py"]
  EOF
  ```
- [ ] Build and run:
  ```bash
  docker build -t devops-app:v1 .
  docker run -d --name devops-app -p 8080:8080 devops-app:v1
  curl http://localhost:8080
  docker stop devops-app && docker rm devops-app
  ```
- [ ] Rebuild with Alpine, compare sizes:
  ```bash
  sed -i 's/python:3.11-slim/python:3.11-alpine/' Dockerfile
  docker build -t devops-app:v2-alpine .
  docker images | grep devops-app   # compare sizes
  ```

**Verify:**
```bash
curl http://localhost:8080   # returns "Hello from my DevOps container!"
docker images | grep devops-app   # shows both v1 and v2-alpine
```

---

### Day 7 — Docker Compose Basics
**Goal:** Run multiple containers together with Docker Compose.

**Tasks (2 hours):**
- [ ] Create a docker-compose.yaml:
  ```bash
  mkdir ~/compose-demo && cd ~/compose-demo
  cat > docker-compose.yaml <<'EOF'
  services:
    db:
      image: postgres:15-alpine
      environment:
        POSTGRES_PASSWORD: mypassword
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U postgres"]
        interval: 5s
        timeout: 3s
        retries: 5

    web:
      image: nginx:alpine
      ports:
        - "8080:80"
      depends_on:
        db:
          condition: service_healthy
  EOF
  ```
- [ ] Start and observe startup order:
  ```bash
  docker compose up -d
  docker compose ps          # db must be healthy before web starts
  docker compose logs db     # see PostgreSQL startup logs
  docker compose logs web    # see Nginx startup logs
  curl http://localhost:8080 # Nginx welcome page
  ```
- [ ] Stop services individually:
  ```bash
  docker compose stop web    # only stops web, db keeps running
  docker compose ps          # db still running
  docker compose down -v     # stop all + remove volumes
  ```

**Verify:**
```bash
docker compose ps   # both services show "healthy" or "running"
curl http://localhost:8080   # returns Nginx welcome page
```

---

## WEEK 2 — Retail Store App Locally + Terraform Basics (Days 8–14)

---

### Day 8 — Clone and Run Retail Store App Locally
**Goal:** Run all 10 microservices locally using Docker Compose.

**Tasks (2 hours):**
- [ ] Clone the retail store app:
  ```bash
  git clone https://github.com/aws-containers/retail-store-sample-app
  cd retail-store-sample-app
  ```
- [ ] Read docker-compose.yaml — identify all 10 services
  ```bash
  cat docker-compose.yaml | grep "image:"    # see all images used
  ```
- [ ] Start all services:
  ```bash
  docker compose up -d
  # Wait 2 minutes for all services to become healthy
  docker compose ps
  ```
- [ ] Test the app in your browser or via curl:
  ```bash
  curl http://localhost:8888/catalogue | jq length   # count products
  ```
- [ ] Explore the app: browse products, add to cart, do a checkout

**Verify:**
```bash
docker compose ps | grep -v "healthy\|running" | grep -v NAME
# Should show nothing — all containers healthy
curl http://localhost:8888/catalogue | jq '.[0].name'
# Should return a product name
```

---

### Day 9 — Explore the Retail Store App Internals
**Goal:** Understand how each container communicates and what each one does.

**Tasks (2 hours):**
- [ ] Map services to containers:
  ```bash
  docker compose ps --format "table {{.Name}}\t{{.Image}}\t{{.Ports}}"
  ```
- [ ] Get inside the catalog container and explore:
  ```bash
  docker exec -it retail-store-sample-app-catalog-1 sh
    ls /                # see app directory structure
    env | grep DB       # see database connection config
    exit
  ```
- [ ] Watch logs from all services:
  ```bash
  docker compose logs -f --tail=5   # follow all logs
  # Open browser, make some requests, watch which services log activity
  ```
- [ ] Test service-to-service communication:
  ```bash
  # The UI calls the catalog API internally
  # Port-forward catalog and call it directly
  docker inspect retail-store-sample-app-catalog-1 | grep IPAddress
  curl http://<catalog-ip>:8080/catalogue | jq '.[0]'
  ```
- [ ] Clean up when done:
  ```bash
  docker compose down -v
  ```

**Verify:**
- You can name what each container does
- You understand that containers talk to each other by service name (not IP)

---

### Day 10 — Terraform Install + First HCL File
**Goal:** Install Terraform and write your first infrastructure-as-code file.

**Tasks (2 hours):**
- [ ] Install Terraform:
  ```bash
  sudo yum-config-manager --add-repo \
    https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
  sudo dnf install terraform -y
  terraform -version
  ```
- [ ] Write your first Terraform file:
  ```bash
  mkdir ~/tf-basics && cd ~/tf-basics
  cat > main.tf <<'EOF'
  terraform {
    required_providers {
      aws = { source = "hashicorp/aws", version = "~> 5.0" }
    }
  }
  provider "aws" { region = "us-east-1" }

  resource "aws_s3_bucket" "my_bucket" {
    bucket = "my-devops-learning-bucket-${random_id.suffix.hex}"
    tags   = { Environment = "dev", Project = "learning" }
  }

  resource "random_id" "suffix" { byte_length = 4 }
  EOF
  ```
- [ ] Run the Terraform workflow:
  ```bash
  terraform init          # downloads providers
  terraform validate      # checks syntax
  terraform plan          # preview changes
  terraform apply         # create resources (type 'yes')
  ```
- [ ] Inspect what was created:
  ```bash
  terraform state list         # shows resources Terraform manages
  terraform state show aws_s3_bucket.my_bucket
  cat terraform.tfstate | jq '.resources[0].instances[0].attributes.bucket'
  ```
- [ ] Clean up:
  ```bash
  terraform destroy   # type 'yes'
  ```

**Verify:**
```bash
terraform state list   # empty after destroy
aws s3 ls | grep devops-learning   # nothing — bucket deleted
```

---

### Day 11 — Terraform Variables + Outputs
**Goal:** Make Terraform code reusable with variables and outputs.

**Tasks (2 hours):**
- [ ] Create a project with variables:
  ```bash
  mkdir ~/tf-variables && cd ~/tf-variables
  ```
- [ ] Create `variables.tf`:
  ```hcl
  variable "environment" {
    description = "Environment name"
    type        = string
    default     = "dev"
  }
  variable "aws_region" {
    type    = string
    default = "us-east-1"
  }
  ```
- [ ] Create `main.tf` that uses the variables:
  ```hcl
  terraform {
    required_providers {
      aws = { source = "hashicorp/aws", version = "~> 5.0" }
    }
  }
  provider "aws" { region = var.aws_region }

  resource "aws_s3_bucket" "bucket" {
    bucket = "learning-${var.environment}-${random_id.id.hex}"
    tags   = { Environment = var.environment }
  }

  resource "random_id" "id" { byte_length = 4 }
  ```
- [ ] Create `outputs.tf`:
  ```hcl
  output "bucket_name" { value = aws_s3_bucket.bucket.id }
  output "bucket_arn"  { value = aws_s3_bucket.bucket.arn }
  ```
- [ ] Test variable override methods:
  ```bash
  terraform init && terraform apply -auto-approve
  terraform output   # see bucket_name and bucket_arn
  # Override at CLI:
  terraform apply -var="environment=staging" -auto-approve
  terraform output bucket_name   # should say "staging" now
  terraform destroy -auto-approve
  ```

**Verify:**
```bash
terraform output bucket_name   # returns the bucket name string
```

---

### Day 12 — Terraform Remote State + S3 Backend
**Goal:** Store Terraform state remotely for team collaboration.

**Tasks (2 hours):**
- [ ] Create the S3 bucket and DynamoDB table for state:
  ```bash
  BUCKET="tfstate-retail-$(openssl rand -hex 4)"
  echo "Save this bucket name: $BUCKET"

  aws s3api create-bucket --bucket $BUCKET --region us-east-1
  aws s3api put-bucket-versioning \
    --bucket $BUCKET \
    --versioning-configuration Status=Enabled
  aws s3api put-bucket-encryption \
    --bucket $BUCKET \
    --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
  aws s3api put-public-access-block \
    --bucket $BUCKET \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

  aws dynamodb create-table \
    --table-name retail-tf-state-lock \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
  ```
- [ ] Update your `main.tf` to use remote state:
  ```hcl
  terraform {
    required_providers {
      aws = { source = "hashicorp/aws", version = "~> 5.0" }
    }
    backend "s3" {
      bucket         = "YOUR_BUCKET_NAME_HERE"
      key            = "learning/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "retail-tf-state-lock"
      encrypt        = true
    }
  }
  ```
- [ ] Re-initialize with remote backend:
  ```bash
  terraform init   # prompts to migrate state to S3
  terraform apply -auto-approve
  ```
- [ ] Verify state is in S3:
  ```bash
  aws s3 ls s3://$BUCKET/learning/   # should show terraform.tfstate
  terraform destroy -auto-approve
  ```

**Verify:**
```bash
aws s3api get-bucket-versioning --bucket $BUCKET
# Status: Enabled
aws s3 ls s3://$BUCKET/learning/terraform.tfstate
# Shows the state file
```

---

### Day 13 — Terraform Modules
**Goal:** Create a reusable Terraform module.

**Tasks (2 hours):**
- [ ] Create a module directory structure:
  ```bash
  mkdir -p ~/tf-modules/modules/s3-bucket
  cd ~/tf-modules
  ```
- [ ] Create `modules/s3-bucket/variables.tf`:
  ```hcl
  variable "bucket_suffix" { type = string }
  variable "environment"   { type = string }
  ```
- [ ] Create `modules/s3-bucket/main.tf`:
  ```hcl
  resource "aws_s3_bucket" "this" {
    bucket = "module-demo-${var.environment}-${var.bucket_suffix}"
    tags   = { Environment = var.environment }
  }
  resource "aws_s3_bucket_versioning" "this" {
    bucket = aws_s3_bucket.this.id
    versioning_configuration { status = "Enabled" }
  }
  ```
- [ ] Create `modules/s3-bucket/outputs.tf`:
  ```hcl
  output "bucket_name" { value = aws_s3_bucket.this.id }
  output "bucket_arn"  { value = aws_s3_bucket.this.arn }
  ```
- [ ] Create root `main.tf` that calls the module twice:
  ```hcl
  terraform {
    required_providers { aws = { source = "hashicorp/aws", version = "~> 5.0" } }
  }
  provider "aws" { region = "us-east-1" }

  module "dev_bucket" {
    source        = "./modules/s3-bucket"
    bucket_suffix = "abc123"
    environment   = "dev"
  }
  module "staging_bucket" {
    source        = "./modules/s3-bucket"
    bucket_suffix = "xyz789"
    environment   = "staging"
  }

  output "dev_bucket"     { value = module.dev_bucket.bucket_name }
  output "staging_bucket" { value = module.staging_bucket.bucket_name }
  ```
- [ ] Apply and verify:
  ```bash
  terraform init && terraform apply -auto-approve
  terraform output   # shows both bucket names
  terraform destroy -auto-approve
  ```

**Verify:**
```bash
terraform output dev_bucket     # "module-demo-dev-abc123"
terraform output staging_bucket # "module-demo-staging-xyz789"
```

---

### Day 14 — Terraform State Deep Dive
**Goal:** Understand how the state file works and how to manage it safely.

**Tasks (2 hours):**
- [ ] Create a small infrastructure and explore the state:
  ```bash
  mkdir ~/tf-state-demo && cd ~/tf-state-demo
  # Create main.tf with an S3 bucket and VPC tag resource
  terraform init && terraform apply -auto-approve

  terraform state list                          # see all managed resources
  terraform state show aws_s3_bucket.bucket     # full attributes
  terraform state pull | jq '.resources | length'  # count resources

  # Simulate a manual change: add a tag in the AWS console
  # Then run:
  terraform refresh    # update state to match real world
  terraform plan       # see the drift
  ```
- [ ] Practice safe rename with state mv:
  ```bash
  # In main.tf, rename: aws_s3_bucket.bucket → aws_s3_bucket.my_bucket
  terraform state mv aws_s3_bucket.bucket aws_s3_bucket.my_bucket
  terraform plan   # should show NO changes — just renamed in state
  ```
- [ ] Practice force-unlock (simulate a stuck lock):
  ```bash
  terraform force-unlock FAKE-LOCK-ID   # will fail — but shows the command
  aws dynamodb scan --table-name retail-tf-state-lock   # see lock table
  ```
- [ ] Clean up:
  ```bash
  terraform destroy -auto-approve
  ```

**Verify:**
- You understand the difference between `terraform plan` showing `~ update` vs `-/+ replace`
- You know how to read the state file JSON structure

---

## WEEK 3 — AWS VPC + EKS Cluster (Days 15–21)

---

### Day 15 — AWS Networking Concepts (Theory + Explore)
**Goal:** Understand VPC, subnets, route tables, and security groups before creating them.

**Tasks (2 hours):**
- [ ] Explore the default VPC in your account:
  ```bash
  aws ec2 describe-vpcs --query 'Vpcs[?IsDefault==`true`]' --output table
  aws ec2 describe-subnets \
    --filters "Name=default-for-az,Values=true" \
    --query 'Subnets[*].{AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}' \
    --output table
  ```
- [ ] Draw on paper (or in a text file):
  - VPC: 10.0.0.0/16
  - 3 public subnets (one per AZ) with CIDR blocks
  - 3 private subnets (one per AZ) with CIDR blocks
  - Internet Gateway → public route table
  - NAT Gateway → private route table
- [ ] Understand CIDR notation: what does /16, /24 mean?
  - /16 = 65,536 IPs
  - /24 = 256 IPs (254 usable)
- [ ] Read about Security Groups vs NACLs:
  ```bash
  aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=default" \
    --query 'SecurityGroups[0].IpPermissions'
  ```

**Verify:**
- You can explain the difference between a public and private subnet
- You can explain why EKS worker nodes go in private subnets

---

### Day 16 — Provision VPC with Terraform
**Goal:** Create a production-grade VPC using Terraform modules.

**Tasks (2 hours):**
- [ ] Install kubectl:
  ```bash
  curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
  chmod +x kubectl && sudo mv kubectl /usr/local/bin/
  kubectl version --client
  ```
- [ ] Create VPC Terraform project:
  ```bash
  mkdir -p ~/retail-infra/01_vpc && cd ~/retail-infra/01_vpc
  ```
- [ ] Create `c1-versions.tf` with S3 backend (use your bucket from Day 12):
  ```hcl
  terraform {
    required_version = ">= 1.6"
    required_providers { aws = { source = "hashicorp/aws", version = "~> 5.0" } }
    backend "s3" {
      bucket         = "YOUR_BUCKET_HERE"
      key            = "vpc/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "retail-tf-state-lock"
      encrypt        = true
    }
  }
  provider "aws" { region = "us-east-1" }
  ```
- [ ] Create `c2-vpc.tf` using the community VPC module:
  ```hcl
  data "aws_availability_zones" "available" { state = "available" }

  module "vpc" {
    source  = "terraform-aws-modules/vpc/aws"
    version = "~> 5.0"
    name    = "retail-store-vpc"
    cidr    = "10.0.0.0/16"
    azs     = slice(data.aws_availability_zones.available.names, 0, 3)
    public_subnets  = ["10.0.0.0/24","10.0.1.0/24","10.0.2.0/24"]
    private_subnets = ["10.0.10.0/24","10.0.11.0/24","10.0.12.0/24"]
    enable_nat_gateway   = true
    single_nat_gateway   = false
    enable_dns_hostnames = true
    public_subnet_tags  = { "kubernetes.io/role/elb" = "1" }
    private_subnet_tags = { "kubernetes.io/role/internal-elb" = "1" }
  }
  ```
- [ ] Create `c3-outputs.tf` and apply:
  ```bash
  terraform init && terraform plan && terraform apply -auto-approve
  ```

**Verify:**
```bash
terraform output   # shows vpc_id, subnet IDs
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=retail-store-vpc" \
  --query 'Vpcs[0].State'   # "available"
```
> ⚠️ **Keep this running — do NOT destroy. You need it for EKS.**

---

### Day 17 — Kubernetes Concepts (Theory)
**Goal:** Understand Kubernetes architecture before touching EKS.

**Tasks (2 hours):**
- [ ] Draw Kubernetes architecture on paper:
  - Control Plane: API Server, etcd, Scheduler, Controller Manager
  - Worker Node: kubelet, kube-proxy, container runtime
- [ ] Understand core objects — write definitions in your own words:
  - Pod: ___
  - Deployment: ___
  - ReplicaSet: ___
  - Service (ClusterIP): ___
  - Namespace: ___
- [ ] Install Helm:
  ```bash
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  helm version
  ```
- [ ] Read EKS vs self-managed Kubernetes differences:
  - EKS = AWS manages control plane (you pay $0.10/hour for it)
  - You manage: worker nodes, add-ons, networking
- [ ] Understand EKS Add-ons vs Helm controllers:
  - EKS Add-ons: managed by AWS (CoreDNS, kube-proxy, VPC CNI, EBS CSI)
  - Helm controllers: you install and manage (AWS LBC, External DNS, Karpenter)

**Verify:**
- You can draw the control loop: desired state → controller → actual state
- You can explain the difference between `kubectl apply` and `helm install`

---

### Day 18 — Provision EKS Cluster (Part 1 — Write Terraform)
**Goal:** Write the Terraform code for the EKS cluster (will apply on Day 19).

**Tasks (2 hours):**
- [ ] Create EKS Terraform project:
  ```bash
  mkdir -p ~/retail-infra/02_eks && cd ~/retail-infra/02_eks
  ```
- [ ] Create `c1-versions.tf` with S3 backend (key: `eks/terraform.tfstate`)
- [ ] Create `c2-remote-state.tf`:
  ```hcl
  data "terraform_remote_state" "vpc" {
    backend = "s3"
    config = {
      bucket = "YOUR_BUCKET_HERE"
      key    = "vpc/terraform.tfstate"
      region = "us-east-1"
    }
  }
  ```
- [ ] Create `c3-eks.tf`:
  ```hcl
  module "eks" {
    source  = "terraform-aws-modules/eks/aws"
    version = "~> 20.0"

    cluster_name    = "retail-store-eks"
    cluster_version = "1.29"
    vpc_id     = data.terraform_remote_state.vpc.outputs.vpc_id
    subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
    cluster_endpoint_public_access = true

    eks_managed_node_groups = {
      default = {
        instance_types = ["t3.medium"]
        min_size = 2 ; max_size = 5 ; desired_size = 3
      }
    }
    cluster_addons = {
      coredns = {} ; kube-proxy = {} ; vpc-cni = {}
      eks-pod-identity-agent = {} ; aws-ebs-csi-driver = {}
    }
  }
  ```
- [ ] Create `c4-outputs.tf`:
  ```hcl
  output "cluster_name"     { value = module.eks.cluster_name }
  output "cluster_endpoint" { value = module.eks.cluster_endpoint }
  ```
- [ ] Run `terraform init` to download providers (do NOT apply yet):
  ```bash
  terraform init
  terraform validate   # must show "Success!"
  terraform plan       # review what will be created
  ```

**Verify:**
```bash
terraform validate   # "Success! The configuration is valid."
terraform plan 2>&1 | grep "will be created" | wc -l
# Should show ~20+ resources to create
```

---

### Day 19 — Provision EKS Cluster (Part 2 — Apply + Verify)
**Goal:** Apply the EKS Terraform and connect kubectl.

**Tasks (2 hours):**
> ⚠️ EKS takes ~15 minutes to create. Start the apply immediately.

- [ ] Apply EKS Terraform:
  ```bash
  cd ~/retail-infra/02_eks
  terraform apply -auto-approve
  # This takes 12–15 minutes — read the output while it runs
  ```
- [ ] Connect kubectl to EKS:
  ```bash
  aws eks update-kubeconfig \
    --region us-east-1 \
    --name $(terraform output -raw cluster_name)
  ```
- [ ] Verify the cluster:
  ```bash
  kubectl get nodes                 # 3 nodes, all Ready
  kubectl get pods -A               # all system pods Running
  kubectl cluster-info              # shows cluster API endpoint
  ```
- [ ] Explore what was created:
  ```bash
  kubectl get pods -n kube-system   # CoreDNS, kube-proxy, VPC CNI
  aws eks list-addons --cluster-name retail-store-eks   # see installed addons
  kubectl get storageclass          # gp2 should be default
  ```

**Verify:**
```bash
kubectl get nodes | grep Ready | wc -l   # 3
kubectl get pods -A | grep -v Running | grep -v NAME | wc -l   # 0
```
> ⚠️ **Keep EKS running for the next days. Destroy at the end of the week to save costs.**

---

### Day 20 — Install Helm Controllers (Part 1)
**Goal:** Install AWS Load Balancer Controller and External DNS via Helm.

**Tasks (2 hours):**
- [ ] Add Helm repos:
  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
  helm repo update
  ```
- [ ] Create IAM role for AWS Load Balancer Controller using Pod Identity:
  ```bash
  # Download the IAM policy
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

  aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
  ```
- [ ] Install AWS Load Balancer Controller:
  ```bash
  VPC_ID=$(cd ~/retail-infra/01_vpc && terraform output -raw vpc_id)
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=retail-store-eks \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set region=us-east-1 \
    --set vpcId=$VPC_ID
  ```
- [ ] Verify LBC is running:
  ```bash
  kubectl get deployment aws-load-balancer-controller -n kube-system
  kubectl logs -n kube-system deploy/aws-load-balancer-controller | tail -5
  ```

**Verify:**
```bash
kubectl get deployment aws-load-balancer-controller -n kube-system \
  -o jsonpath='{.status.availableReplicas}'   # 2
```

---

### Day 21 — Install Helm Controllers (Part 2) + Weekly Review
**Goal:** Install Secrets Store CSI Driver + review Week 3.

**Tasks (2 hours):**
- [ ] Install Secrets Store CSI Driver:
  ```bash
  helm repo add secrets-store-csi-driver \
    https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
  helm repo update
  helm install csi-secrets-store \
    secrets-store-csi-driver/secrets-store-csi-driver \
    -n kube-system \
    --set syncSecret.enabled=true

  kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
  ```
- [ ] Verify all controllers:
  ```bash
  kubectl get all -n kube-system | grep -E "aws-load-balancer|secrets-store"
  ```
- [ ] Weekly review — answer these without looking:
  - What does NAT Gateway do that Internet Gateway doesn't?
  - What is the Terraform state file used for?
  - What is the difference between Helm install and kubectl apply?
  - Why do EKS worker nodes go in private subnets?
- [ ] **Cost save:** Destroy EKS if done for the week:
  ```bash
  # Only destroy EKS, keep VPC:
  cd ~/retail-infra/02_eks && terraform destroy -auto-approve
  # VPC costs near nothing to keep
  ```

**Verify:**
```bash
kubectl get daemonset secrets-store-csi-driver -n kube-system
# DESIRED and READY should be equal
```

---

## WEEK 4 — Kubernetes Core Objects (Days 22–28)

---

### Day 22 — kubectl Fundamentals
**Goal:** Master the essential kubectl commands.

**Tasks (2 hours):**
> If you destroyed EKS: `cd ~/retail-infra/02_eks && terraform apply -auto-approve` (15 min wait)

- [ ] Practice all essential kubectl commands:
  ```bash
  kubectl get nodes -o wide                    # node details
  kubectl get pods -A                          # all pods all namespaces
  kubectl get pods -A --field-selector=status.phase!=Running  # non-running pods

  # Run a temporary debug pod
  kubectl run tmp --image=busybox --rm -it --restart=Never -- sh
    nslookup kubernetes    # in-cluster DNS works
    wget -q -O- https://ifconfig.me   # outbound internet via NAT
    exit

  # Describe a system pod
  kubectl describe pod -n kube-system -l k8s-app=kube-dns | head -30

  # Get logs
  kubectl logs -n kube-system \
    -l app.kubernetes.io/name=aws-load-balancer-controller --tail=10
  ```
- [ ] Practice output formats:
  ```bash
  kubectl get nodes -o yaml | head -30        # raw YAML
  kubectl get nodes -o json | jq '.items[0].metadata.name'  # JSON path
  kubectl get pods -A -o wide                 # wide table
  ```

**Verify:**
```bash
kubectl get pods -A | grep -v Running | grep -v Completed | grep -v NAME
# Should show nothing — all pods healthy
```

---

### Day 23 — Deployments + Services
**Goal:** Deploy your first app to Kubernetes and expose it.

**Tasks (2 hours):**
- [ ] Create namespace and deploy nginx:
  ```bash
  kubectl create namespace demo

  kubectl apply -f - <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-demo
    namespace: demo
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:alpine
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-svc
    namespace: demo
  spec:
    selector:
      app: nginx
    ports:
    - port: 80
      targetPort: 80
  EOF
  ```
- [ ] Explore the deployment:
  ```bash
  kubectl get deployment,replicaset,pod -n demo
  kubectl rollout status deployment/nginx-demo -n demo

  # Test service discovery from inside cluster
  kubectl run test --image=busybox --rm -it --restart=Never \
    -n demo -- wget -q -O- http://nginx-svc

  # Scale down and watch reconciliation
  kubectl scale deployment nginx-demo --replicas=1 -n demo
  kubectl get pods -n demo -w   # watch one pod terminate
  kubectl scale deployment nginx-demo --replicas=3 -n demo
  ```
- [ ] Deliberately break it and debug:
  ```bash
  kubectl set image deployment/nginx-demo nginx=nginx:BROKEN -n demo
  kubectl get pods -n demo   # ImagePullBackOff
  kubectl describe pod -n demo -l app=nginx | grep -A5 Events
  kubectl set image deployment/nginx-demo nginx=nginx:alpine -n demo
  ```

**Verify:**
```bash
kubectl get deployment nginx-demo -n demo \
  -o jsonpath='{.status.readyReplicas}'   # 3
```

---

### Day 24 — ConfigMaps + Secrets + Environment Variables
**Goal:** Inject configuration into pods without hardcoding values.

**Tasks (2 hours):**
- [ ] Create a ConfigMap and mount it:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-config
    namespace: demo
  data:
    APP_ENV: "production"
    DB_HOST: "mydb.example.com"
    DB_PORT: "5432"
  EOF
  ```
- [ ] Create a Secret (base64 encoded):
  ```bash
  kubectl create secret generic app-secret \
    --from-literal=DB_PASSWORD=mypassword123 \
    -n demo

  # See what's in the secret
  kubectl get secret app-secret -n demo -o yaml
  kubectl get secret app-secret -n demo \
    -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
  # You can read it! This is why K8s Secrets are NOT secure
  ```
- [ ] Deploy a pod that reads both:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata:
    name: config-test
    namespace: demo
  spec:
    containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "env && sleep 3600"]
      envFrom:
      - configMapRef:
          name: app-config
      - secretRef:
          name: app-secret
  EOF

  kubectl exec -it config-test -n demo -- env | grep -E "APP|DB"
  ```

**Verify:**
```bash
kubectl exec -it config-test -n demo -- env | grep DB_PASSWORD
# Shows DB_PASSWORD=mypassword123 — demonstrates K8s Secrets are base64 only
```

---

### Day 25 — Kubernetes Storage (PV, PVC, StorageClass)
**Goal:** Understand how persistent storage works in Kubernetes.

**Tasks (2 hours):**
- [ ] Explore available StorageClasses:
  ```bash
  kubectl get storageclass
  kubectl describe storageclass gp2
  ```
- [ ] Create a PVC and verify it gets bound:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: test-pvc
    namespace: demo
  spec:
    accessModes: [ReadWriteOnce]
    storageClassName: gp2
    resources:
      requests:
        storage: 1Gi
  EOF

  kubectl get pvc test-pvc -n demo   # STATUS: Pending (no pod using it yet)
  ```
- [ ] Create a pod that uses the PVC:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata:
    name: pvc-test
    namespace: demo
  spec:
    containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo 'persistent data' > /data/test.txt && sleep 3600"]
      volumeMounts:
      - name: storage
        mountPath: /data
    volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: test-pvc
  EOF

  kubectl get pvc test-pvc -n demo   # STATUS: Bound (EBS volume created)
  kubectl exec pvc-test -n demo -- cat /data/test.txt
  ```
- [ ] See the EBS volume in AWS:
  ```bash
  aws ec2 describe-volumes \
    --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=test-pvc" \
    --query 'Volumes[0].{State:State,Size:Size}'
  ```

**Verify:**
```bash
kubectl get pvc test-pvc -n demo -o jsonpath='{.status.phase}'   # Bound
```

---

### Day 26 — Helm Basics
**Goal:** Install and manage an application using Helm.

**Tasks (2 hours):**
- [ ] Explore a public Helm chart before installing:
  ```bash
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm repo update
  helm search repo bitnami/nginx
  helm show values bitnami/nginx | head -50   # see all configurable values
  ```
- [ ] Install nginx with Helm:
  ```bash
  helm install my-nginx bitnami/nginx \
    --namespace demo \
    --set replicaCount=2 \
    --set service.type=ClusterIP
  helm list -n demo
  helm status my-nginx -n demo
  ```
- [ ] Upgrade the release:
  ```bash
  helm upgrade my-nginx bitnami/nginx \
    --namespace demo \
    --set replicaCount=3
  kubectl get pods -n demo | grep nginx   # 3 nginx pods
  ```
- [ ] Roll back:
  ```bash
  helm history my-nginx -n demo
  helm rollback my-nginx 1 -n demo
  kubectl get pods -n demo | grep nginx   # back to 2 pods
  ```
- [ ] Uninstall:
  ```bash
  helm uninstall my-nginx -n demo
  ```

**Verify:**
```bash
helm history my-nginx -n demo
# Shows revision 1 (install), 2 (upgrade), 3 (rollback)
```

---

### Day 27 — Deploy Catalog Microservice Manually
**Goal:** Deploy one microservice with all its dependencies by hand.

**Tasks (2 hours):**
- [ ] Create namespace:
  ```bash
  kubectl create namespace retail-store
  ```
- [ ] Deploy catalog MySQL (StatefulSet with emptyDir — dev only):
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: catalog-mysql
    namespace: retail-store
  spec:
    selector:
      matchLabels:
        app: catalog-mysql
    serviceName: catalog-mysql
    replicas: 1
    template:
      metadata:
        labels:
          app: catalog-mysql
      spec:
        containers:
        - name: mysql
          image: mysql:8.0
          env:
          - name: MYSQL_ROOT_PASSWORD
            value: "password"
          - name: MYSQL_DATABASE
            value: "catalog"
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: catalog-mysql
    namespace: retail-store
  spec:
    clusterIP: None
    selector:
      app: catalog-mysql
    ports:
    - port: 3306
  EOF
  kubectl get pods -n retail-store -w   # wait for mysql to be Running
  ```
- [ ] Deploy catalog API:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: catalog
    namespace: retail-store
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: catalog
    template:
      metadata:
        labels:
          app: catalog
      spec:
        containers:
        - name: catalog
          image: public.ecr.aws/aws-containers/retail-store-sample-catalog:0.7.0
          env:
          - name: DB_PASSWORD
            value: "password"
          - name: DB_NAME
            value: "catalog"
          - name: DB_ENDPOINT
            value: "catalog-mysql"
          ports:
          - containerPort: 8080
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: catalog
    namespace: retail-store
  spec:
    selector:
      app: catalog
    ports:
    - port: 80
      targetPort: 8080
  EOF

  kubectl rollout status deployment/catalog -n retail-store
  kubectl port-forward svc/catalog 8080:80 -n retail-store &
  curl http://localhost:8080/catalogue | jq length
  ```

**Verify:**
```bash
curl http://localhost:8080/catalogue | jq '.[0].name'
# Returns a product name — catalog is connected to MySQL
```

---

### Day 28 — Week Review + HPA Basics
**Goal:** Test HPA and review all Week 4 concepts.

**Tasks (2 hours):**
- [ ] Apply HPA to the catalog deployment:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: catalog-hpa
    namespace: retail-store
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: catalog
    minReplicas: 2
    maxReplicas: 5
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  EOF
  ```
- [ ] Install Metrics Server (required for HPA):
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  sleep 60
  kubectl top pods -n retail-store   # should show CPU/mem
  kubectl get hpa -n retail-store    # TARGETS should show actual%/50%
  ```
- [ ] Generate load and watch HPA:
  ```bash
  kubectl run load -n retail-store --image=busybox --restart=Never -- \
    /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue; done"
  kubectl get hpa catalog-hpa -n retail-store -w   # watch replicas increase
  kubectl delete pod load -n retail-store
  ```

**Verify:**
```bash
kubectl get hpa catalog-hpa -n retail-store
# TARGETS shows actual CPU%, REPLICAS increased under load
```

---

## WEEK 5 — AWS Managed Databases + Secrets (Days 29–35)

---

### Day 29 — AWS Secrets Manager + Pod Identity (Theory + Setup)
**Goal:** Understand why Secrets Manager is better than K8s Secrets and set up the chain.

**Tasks (2 hours):**
- [ ] Create a test secret in Secrets Manager:
  ```bash
  aws secretsmanager create-secret \
    --name "retail-store/test-db" \
    --secret-string '{"username":"admin","password":"TestPass123!","host":"mydb.example.com"}'

  # Verify
  aws secretsmanager get-secret-value \
    --secret-id retail-store/test-db \
    --query SecretString --output text | jq .
  ```
- [ ] Create an IAM policy for Secrets Manager access:
  ```bash
  cat > sm-policy.json <<'EOF'
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue","secretsmanager:DescribeSecret"],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:retail-store/*"
    }]
  }
  EOF
  aws iam create-policy \
    --policy-name RetailStoreSecretsPolicy \
    --policy-document file://sm-policy.json
  ```
- [ ] Create an IAM role for the catalog service account:
  ```bash
  # Get OIDC provider URL for your cluster
  OIDC_URL=$(aws eks describe-cluster --name retail-store-eks \
    --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')
  echo $OIDC_URL

  # Create trust policy
  ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  cat > trust-policy.json <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_URL}"},
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_URL}:sub": "system:serviceaccount:retail-store:catalog-sa",
          "${OIDC_URL}:aud": "sts.amazonaws.com"
        }
      }
    }]
  }
  EOF
  aws iam create-role --role-name retail-catalog-sa-role \
    --assume-role-policy-document file://trust-policy.json
  aws iam attach-role-policy --role-name retail-catalog-sa-role \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/RetailStoreSecretsPolicy
  ```

**Verify:**
```bash
aws iam get-role --role-name retail-catalog-sa-role \
  --query 'Role.AssumeRolePolicyDocument' --output text
# Shows the trust policy with the OIDC provider
```

---

### Day 30 — Deploy Secret Into a Pod via CSI Driver
**Goal:** Mount a Secrets Manager secret into a pod as a file and env var.

**Tasks (2 hours):**
- [ ] Create a ServiceAccount linked to the IAM role:
  ```bash
  ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: catalog-sa
    namespace: retail-store
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::${ACCOUNT_ID}:role/retail-catalog-sa-role
  EOF
  ```
- [ ] Create a SecretProviderClass:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: secrets-store.csi.x-k8s.io/v1
  kind: SecretProviderClass
  metadata:
    name: retail-test-secret
    namespace: retail-store
  spec:
    provider: aws
    parameters:
      objects: |
        - objectName: "retail-store/test-db"
          objectType: "secretsmanager"
          jmesPath:
            - path: password
              objectAlias: db-password
    secretObjects:
    - secretName: retail-test-k8s-secret
      type: Opaque
      data:
      - objectName: db-password
        key: DB_PASSWORD
  EOF
  ```
- [ ] Deploy a test pod that mounts the secret:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata:
    name: secret-test
    namespace: retail-store
  spec:
    serviceAccountName: catalog-sa
    containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: retail-test-k8s-secret
            key: DB_PASSWORD
      volumeMounts:
      - name: secrets
        mountPath: /mnt/secrets
        readOnly: true
    volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: retail-test-secret
  EOF

  kubectl exec secret-test -n retail-store -- cat /mnt/secrets/db-password
  kubectl exec secret-test -n retail-store -- env | grep DB_PASSWORD
  ```

**Verify:**
```bash
kubectl exec secret-test -n retail-store -- env | grep DB_PASSWORD
# DB_PASSWORD=TestPass123! — from AWS Secrets Manager, not hardcoded
```

---

### Day 31 — AWS RDS MySQL Setup
**Goal:** Provision an RDS MySQL instance for the catalog microservice.

**Tasks (2 hours):**
- [ ] Create RDS subnet group (uses private subnets from VPC):
  ```bash
  PRIVATE_SUBNETS=$(cd ~/retail-infra/01_vpc && \
    terraform output -json private_subnet_ids | jq -r '.[]' | tr '\n' ' ')

  aws rds create-db-subnet-group \
    --db-subnet-group-name retail-store-db-subnets \
    --db-subnet-group-description "Retail store DB subnets" \
    --subnet-ids $PRIVATE_SUBNETS
  ```
- [ ] Create a Security Group for RDS (allows EKS nodes):
  ```bash
  VPC_ID=$(cd ~/retail-infra/01_vpc && terraform output -raw vpc_id)

  SG_ID=$(aws ec2 create-security-group \
    --group-name retail-rds-sg \
    --description "RDS security group" \
    --vpc-id $VPC_ID \
    --query GroupId --output text)

  # Allow MySQL from within the VPC
  aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp --port 3306 \
    --cidr 10.0.0.0/16
  echo "RDS SG: $SG_ID"
  ```
- [ ] Create RDS MySQL instance:
  ```bash
  aws rds create-db-instance \
    --db-instance-identifier retail-catalog \
    --engine mysql --engine-version 8.0 \
    --db-instance-class db.t3.micro \
    --allocated-storage 20 \
    --db-name catalog \
    --master-username catalog_user \
    --master-user-password "CatalogPass123!" \
    --vpc-security-group-ids $SG_ID \
    --db-subnet-group-name retail-store-db-subnets \
    --no-publicly-accessible \
    --no-multi-az    # skip multi-AZ for cost during learning
  ```
- [ ] Wait for RDS to be available (~5 minutes):
  ```bash
  aws rds wait db-instance-available --db-instance-identifier retail-catalog
  aws rds describe-db-instances \
    --db-instance-identifier retail-catalog \
    --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address}'
  ```

**Verify:**
```bash
aws rds describe-db-instances \
  --db-instance-identifier retail-catalog \
  --query 'DBInstances[0].DBInstanceStatus'   # "available"
```

---

### Day 32 — Wire Catalog to RDS + Secrets Manager
**Goal:** Connect the catalog microservice to the real RDS MySQL database.

**Tasks (2 hours):**
- [ ] Store RDS credentials in Secrets Manager:
  ```bash
  RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier retail-catalog \
    --query 'DBInstances[0].Endpoint.Address' --output text)

  aws secretsmanager update-secret \
    --secret-id retail-store/test-db \
    --secret-string "{
      \"username\": \"catalog_user\",
      \"password\": \"CatalogPass123!\",
      \"host\": \"$RDS_ENDPOINT\",
      \"port\": \"3306\",
      \"dbname\": \"catalog\"
    }"
  ```
- [ ] Create ExternalName Service pointing to RDS:
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: catalog-mysql-rds
    namespace: retail-store
  spec:
    type: ExternalName
    externalName: $RDS_ENDPOINT
    ports:
    - port: 3306
  EOF
  ```
- [ ] Test DNS resolution from inside cluster:
  ```bash
  kubectl run dns-test --image=busybox --rm -it --restart=Never \
    -n retail-store -- nslookup catalog-mysql-rds
  # Should resolve to the RDS IP address
  ```
- [ ] Update catalog deployment to use RDS endpoint from secret:
  - Rebuild the SecretProviderClass to include host, username, password
  - Update the catalog Deployment env vars to use the mounted secrets

**Verify:**
```bash
kubectl exec -it deploy/catalog -n retail-store -- \
  curl http://localhost:8080/catalogue | jq length
# Returns products from RDS MySQL — not the local MySQL StatefulSet
```

---

### Day 33 — PDB + Topology Spread Constraints
**Goal:** Make the app resilient to node failures.

**Tasks (2 hours):**
- [ ] Apply a PDB to catalog:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: catalog-pdb
    namespace: retail-store
  spec:
    minAvailable: 2
    selector:
      matchLabels:
        app: catalog
  EOF
  kubectl get pdb -n retail-store
  ```
- [ ] Scale catalog to 3 replicas and test PDB:
  ```bash
  kubectl scale deployment catalog --replicas=3 -n retail-store
  kubectl get pods -n retail-store -o wide   # see which nodes pods are on

  # Try to drain a node — PDB should limit eviction
  NODE=$(kubectl get pods -n retail-store -l app=catalog \
    -o jsonpath='{.items[0].spec.nodeName}')
  kubectl cordon $NODE
  kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
  # PDB ensures at least 2 catalog pods keep running during drain
  kubectl get pods -n retail-store -w
  kubectl uncordon $NODE
  ```
- [ ] Add Topology Spread Constraints (spread across AZs):
  ```bash
  # Edit catalog deployment to add topologySpreadConstraints:
  kubectl patch deployment catalog -n retail-store --type=json -p='[{
    "op": "add", "path": "/spec/template/spec/topologySpreadConstraints",
    "value": [{
      "maxSkew": 1,
      "topologyKey": "topology.kubernetes.io/zone",
      "whenUnsatisfiable": "DoNotSchedule",
      "labelSelector": {"matchLabels": {"app": "catalog"}}
    }]
  }]'
  kubectl get pods -n retail-store -o wide   # see spread across AZs
  ```

**Verify:**
```bash
kubectl get pdb catalog-pdb -n retail-store
# ALLOWED DISRUPTIONS = 1 (can evict 1 pod, keeps 2 running)
```

---

### Day 34 — ALB Ingress + Route53 (Part 1)
**Goal:** Expose the UI via an ALB and public DNS.

**Tasks (2 hours):**
- [ ] Register a domain in Route53 (if you have one) or use the ALB DNS directly for testing
- [ ] Deploy the UI microservice:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ui
    namespace: retail-store
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: ui
    template:
      metadata:
        labels:
          app: ui
      spec:
        containers:
        - name: ui
          image: public.ecr.aws/aws-containers/retail-store-sample-ui:0.7.0
          ports:
          - containerPort: 8080
          env:
          - name: RETAIL_CATALOG_URL
            value: "http://catalog"
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: ui
    namespace: retail-store
  spec:
    selector:
      app: ui
    ports:
    - port: 80
      targetPort: 8080
  EOF
  ```
- [ ] Create an Ingress to expose UI via ALB:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: retail-store
    namespace: retail-store
    annotations:
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
  spec:
    rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: ui
              port:
                number: 80
  EOF

  kubectl get ingress -n retail-store -w   # wait for ADDRESS to appear
  ```

**Verify:**
```bash
ALB_DNS=$(kubectl get ingress retail-store -n retail-store \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$ALB_DNS/catalogue | jq length   # products from catalog
```

---

### Day 35 — Week Review + Full App Status Check
**Goal:** Verify everything is working and review Week 5 concepts.

**Tasks (2 hours):**
- [ ] Full status check:
  ```bash
  kubectl get pods -n retail-store        # all Running
  kubectl get svc -n retail-store         # all ClusterIP + 1 Ingress
  kubectl get ingress -n retail-store     # ALB hostname present
  kubectl get hpa -n retail-store         # targets not unknown
  kubectl get pdb -n retail-store         # disruptions allowed >= 1
  ```
- [ ] Test the full app via the ALB:
  ```bash
  ALB=$(kubectl get ingress retail-store -n retail-store \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  curl http://$ALB/catalogue | jq '.[0].name'
  ```
- [ ] Review questions (answer without looking):
  - What is the difference between Secrets Manager and a Kubernetes Secret?
  - What does a PDB with `minAvailable: 2` guarantee?
  - Why does an ExternalName Service require zero app code changes?
- [ ] **Cost save:** Note your daily AWS cost in billing dashboard

**Verify:**
```bash
curl http://$ALB/catalogue | jq length   # > 0 products returned
```

---

## WEEK 6 — Karpenter + Full Helm Deploy (Days 36–42)

---

### Day 36 — Karpenter Install + NodePool
**Goal:** Install Karpenter and configure Spot instance autoscaling.

**Tasks (2 hours):**
- [ ] Install Karpenter via Helm:
  ```bash
  helm repo add karpenter https://charts.karpenter.sh/
  helm repo update
  helm install karpenter karpenter/karpenter \
    --namespace karpenter --create-namespace \
    --set settings.clusterName=retail-store-eks \
    --set settings.interruptionQueue=retail-store-karpenter-interruption \
    --set controller.resources.requests.cpu=1 \
    --set controller.resources.requests.memory=1Gi
  kubectl get pods -n karpenter
  ```
- [ ] Create a NodePool:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: karpenter.sh/v1beta1
  kind: NodePool
  metadata:
    name: default-pool
  spec:
    template:
      spec:
        requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: [t3.medium, t3.large]
    limits:
      cpu: "50"
    disruption:
      consolidationPolicy: WhenEmpty
      consolidateAfter: 30s
  EOF
  ```
- [ ] Test Karpenter by forcing a scale event:
  ```bash
  kubectl scale deployment catalog --replicas=10 -n retail-store
  kubectl get nodes -w   # Karpenter should provision a new node
  kubectl get pods -n retail-store -w
  kubectl scale deployment catalog --replicas=3 -n retail-store
  ```

**Verify:**
```bash
kubectl get nodepools   # shows default-pool
kubectl get nodes -L karpenter.sh/registered   # new Karpenter node shows up
```

---

### Day 37 — Deploy Full Retail Store App via Helm
**Goal:** Deploy the complete 5-microservice app using Helm.

**Tasks (2 hours):**
- [ ] Clean up manual resources from earlier weeks:
  ```bash
  kubectl delete all --all -n retail-store
  kubectl delete pdb,hpa,pvc --all -n retail-store
  ```
- [ ] Clone the retail store app and explore the Helm chart:
  ```bash
  cd retail-store-sample-app
  ls deploy/kubernetes/   # see what's available
  helm show values deploy/helm/retailstore 2>/dev/null || \
    ls deploy/kubernetes/
  ```
- [ ] Create `values-dev.yaml` pointing to your RDS endpoint:
  ```yaml
  catalog:
    image:
      tag: "0.7.0"
    mysql:
      host: "retail-catalog.YOUR_RDS_ENDPOINT.us-east-1.rds.amazonaws.com"
  ```
- [ ] Install the full app via Helm:
  ```bash
  kubectl create namespace retail-store
  helm install retail-store ./deploy/helm/retailstore \
    --namespace retail-store \
    --values values-dev.yaml \
    --wait --timeout 5m0s
  ```
- [ ] Verify all pods:
  ```bash
  kubectl get pods -n retail-store   # all 5 services Running
  helm list -n retail-store          # retail-store release: deployed
  ```

**Verify:**
```bash
helm status retail-store -n retail-store
# STATUS: deployed
kubectl get pods -n retail-store | grep -v Running | grep -v NAME
# Nothing — all Running
```

---

### Day 38 — HTTPS + ACM Certificate
**Goal:** Secure the app with a real TLS certificate.

**Tasks (2 hours):**
- [ ] Request an ACM certificate for your domain:
  ```bash
  # If you have a domain in Route53:
  CERT_ARN=$(aws acm request-certificate \
    --domain-name "*.yourdomain.com" \
    --validation-method DNS \
    --query CertificateArn --output text)
  echo $CERT_ARN

  # Validate via DNS (follow console prompts or use CLI)
  aws acm describe-certificate --certificate-arn $CERT_ARN \
    --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
  # Add this CNAME to Route53, then wait:
  aws acm wait certificate-validated --certificate-arn $CERT_ARN
  ```
- [ ] Update Ingress with HTTPS:
  ```bash
  kubectl annotate ingress retail-store -n retail-store \
    "alb.ingress.kubernetes.io/certificate-arn=$CERT_ARN" \
    "alb.ingress.kubernetes.io/listen-ports=[{\"HTTPS\":443},{\"HTTP\":80}]" \
    "alb.ingress.kubernetes.io/ssl-redirect=443"
  ```
- [ ] If you don't have a domain, test the ALB with HTTP only:
  ```bash
  ALB=$(kubectl get ingress retail-store -n retail-store \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  curl http://$ALB/catalogue | jq length
  ```

**Verify:**
```bash
curl -L https://retailstore.yourdomain.com/catalogue | jq length
# OR (no domain):
curl http://$ALB/catalogue | jq length   # > 0
```

---

### Day 39 — Helm Chart Customisation Practice
**Goal:** Customise the Helm chart values and understand templating.

**Tasks (2 hours):**
- [ ] Explore the chart templates:
  ```bash
  ls deploy/helm/retailstore/templates/
  cat deploy/helm/retailstore/values.yaml | head -50
  ```
- [ ] Dry-run with custom values:
  ```bash
  helm template retail-store ./deploy/helm/retailstore \
    --values values-dev.yaml \
    --set catalog.replicaCount=5 | grep -A3 "replicas:"
  ```
- [ ] Practice helm upgrade with --set:
  ```bash
  helm upgrade retail-store ./deploy/helm/retailstore \
    --namespace retail-store \
    --values values-dev.yaml \
    --set catalog.replicaCount=3
  kubectl get pods -n retail-store | grep catalog
  ```
- [ ] Create a `values-prod.yaml`:
  ```yaml
  catalog:
    replicaCount: 3
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
  ui:
    replicaCount: 3
  ```
- [ ] Test rollback:
  ```bash
  helm upgrade retail-store ./deploy/helm/retailstore \
    --namespace retail-store \
    --set ui.image.tag=BROKEN-TAG-DOES-NOT-EXIST
  kubectl get pods -n retail-store | grep ui   # ImagePullBackOff
  helm rollback retail-store -n retail-store   # roll back to previous
  kubectl get pods -n retail-store | grep ui   # Running again
  ```

**Verify:**
```bash
helm history retail-store -n retail-store
# shows at least 3 revisions with a failed + rollback
```

---

### Day 40 — Spot NodePool + PDB Integration
**Goal:** Configure Karpenter for Spot with interruption handling.

**Tasks (2 hours):**
- [ ] Create SQS queue for interruption handling:
  ```bash
  aws sqs create-queue \
    --queue-name retail-store-karpenter-interruption \
    --attributes '{"MessageRetentionPeriod":"300"}'
  ```
- [ ] Create EventBridge rule for Spot interruptions:
  ```bash
  aws events put-rule \
    --name retail-store-spot-interruption \
    --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Spot Instance Interruption Warning"]}'

  QUEUE_ARN=$(aws sqs get-queue-attributes \
    --queue-url $(aws sqs get-queue-url --queue-name retail-store-karpenter-interruption \
      --query QueueUrl --output text) \
    --attribute-names QueueArn --query Attributes.QueueArn --output text)

  aws events put-targets \
    --rule retail-store-spot-interruption \
    --targets "Id=SQSQueue,Arn=$QUEUE_ARN"
  ```
- [ ] Update NodePool to use Spot:
  ```bash
  kubectl patch nodepool default-pool --type=json -p='[{
    "op": "replace",
    "path": "/spec/template/spec/requirements/1/values",
    "value": ["spot","on-demand"]
  }]'
  ```
- [ ] Apply PDB for all 5 microservices:
  ```bash
  for svc in catalog carts checkout orders ui; do
  kubectl apply -f - <<EOF
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: ${svc}-pdb
    namespace: retail-store
  spec:
    minAvailable: 2
    selector:
      matchLabels:
        app: ${svc}
  EOF
  done
  kubectl get pdb -n retail-store
  ```

**Verify:**
```bash
kubectl get pdb -n retail-store   # 5 PDBs, all showing ALLOWED DISRUPTIONS >= 1
```

---

### Day 41 — X-Ray Traces Setup (ADOT Part 1)
**Goal:** Install ADOT and start sending traces to X-Ray.

**Tasks (2 hours):**
- [ ] Install Cert-Manager (required for ADOT Operator):
  ```bash
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
  kubectl wait --for=condition=available deployment/cert-manager \
    -n cert-manager --timeout=120s
  ```
- [ ] Install ADOT Operator:
  ```bash
  helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
  helm repo update
  helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
    --namespace opentelemetry-operator-system --create-namespace \
    --set admissionWebhooks.certManager.enabled=true
  ```
- [ ] Create ADOT Collector for traces (sends to X-Ray):
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: opentelemetry.io/v1alpha1
  kind: OpenTelemetryCollector
  metadata:
    name: retail-traces
    namespace: retail-store
  spec:
    mode: deployment
    serviceAccount: adot-collector-sa
    config: |
      receivers:
        otlp:
          protocols:
            grpc:
              endpoint: 0.0.0.0:4317
            http:
              endpoint: 0.0.0.0:4318
      processors:
        memory_limiter:
          limit_mib: 512
          check_interval: 1s
        batch: {}
      exporters:
        awsxray:
          region: us-east-1
      service:
        pipelines:
          traces:
            receivers: [otlp]
            processors: [memory_limiter, batch]
            exporters: [awsxray]
  EOF

  kubectl get opentelemetrycollector -n retail-store
  ```
- [ ] Generate traffic and check X-Ray:
  ```bash
  ALB=$(kubectl get ingress retail-store -n retail-store \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  for i in {1..20}; do curl -s http://$ALB/catalogue > /dev/null; done
  sleep 30
  aws xray get-trace-summaries \
    --start-time $(date -d '5 minutes ago' +%s) \
    --end-time $(date +%s) \
    --query 'length(TraceSummaries)'
  ```

**Verify:**
```bash
aws xray get-trace-summaries \
  --start-time $(date -d '5 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'length(TraceSummaries)'   # > 0
```

---

### Day 42 — CloudWatch Logs + Prometheus Metrics
**Goal:** Complete the observability stack with logs and metrics.

**Tasks (2 hours):**
- [ ] Create ADOT Collector for logs (DaemonSet):
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: opentelemetry.io/v1alpha1
  kind: OpenTelemetryCollector
  metadata:
    name: retail-logs
    namespace: retail-store
  spec:
    mode: daemonset
    config: |
      receivers:
        filelog:
          include: [/var/log/pods/retail-store_*/*/*.log]
          include_file_path: true
      processors:
        batch: {}
      exporters:
        awscloudwatchlogs:
          log_group_name: /aws/retail-store/application
          log_stream_name: pod-logs
          region: us-east-1
      service:
        pipelines:
          logs:
            receivers: [filelog]
            processors: [batch]
            exporters: [awscloudwatchlogs]
  EOF
  ```
- [ ] Verify logs in CloudWatch:
  ```bash
  sleep 60
  aws logs describe-log-groups \
    --log-group-name-prefix /aws/retail-store
  aws logs filter-log-events \
    --log-group-name /aws/retail-store/application \
    --limit 5
  ```
- [ ] Install Prometheus Node Exporter and Kube State Metrics:
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  helm install kube-state-metrics prometheus-community/kube-state-metrics \
    -n monitoring --create-namespace
  helm install prometheus-node-exporter \
    prometheus-community/prometheus-node-exporter -n monitoring
  ```
- [ ] Verify metrics endpoints are available:
  ```bash
  kubectl port-forward -n monitoring svc/kube-state-metrics 8080:8080 &
  curl http://localhost:8080/metrics | grep kube_pod_status_ready | head -5
  ```

**Verify:**
```bash
aws logs filter-log-events \
  --log-group-name /aws/retail-store/application \
  --limit 1 \
  --query 'events[0].message'   # returns a log line from your app
```

---

## WEEK 7 — CI/CD GitOps Pipeline (Days 43–49)

---

### Day 43 — AWS ECR + GitHub OIDC Setup
**Goal:** Create ECR repositories and set up passwordless GitHub → AWS authentication.

**Tasks (2 hours):**
- [ ] Create ECR repositories:
  ```bash
  for svc in ui catalog carts checkout orders; do
    aws ecr create-repository \
      --repository-name retail-store/$svc \
      --image-scanning-configuration scanOnPush=true \
      --region us-east-1
  done
  aws ecr describe-repositories --query 'repositories[*].repositoryUri'
  ```
- [ ] Set up OIDC provider for GitHub Actions:
  ```bash
  aws iam create-open-id-connect-provider \
    --url https://token.actions.githubusercontent.com \
    --client-id-list sts.amazonaws.com \
    --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
  ```
- [ ] Create IAM role for GitHub Actions:
  ```bash
  ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  cat > github-trust-policy.json <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_ORG/aws-devops-retail-store:*"
        }
      }
    }]
  }
  EOF
  aws iam create-role \
    --role-name retail-store-github-actions \
    --assume-role-policy-document file://github-trust-policy.json
  aws iam attach-role-policy \
    --role-name retail-store-github-actions \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
  echo "Role ARN: arn:aws:iam::${ACCOUNT_ID}:role/retail-store-github-actions"
  ```

**Verify:**
```bash
aws ecr describe-repositories \
  --query 'repositories[*].repositoryName' --output table
# Shows 5 repositories
aws iam get-role --role-name retail-store-github-actions \
  --query 'Role.Arn' --output text
# Shows the role ARN
```

---

### Day 44 — Write GitHub Actions CI Workflow
**Goal:** Write the CI workflow that builds, tags, and pushes Docker images to ECR.

**Tasks (2 hours):**
- [ ] In your `aws-devops-retail-store` GitHub repo, create:
  ```bash
  mkdir -p .github/workflows
  ```
- [ ] Create `.github/workflows/build-ui.yaml`:
  ```yaml
  name: Build and Push UI

  on:
    push:
      branches: [main]
      paths: ['src/ui/**']

  env:
    AWS_REGION: us-east-1
    ECR_REGISTRY: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPOSITORY: retail-store/ui

  jobs:
    build-and-push:
      runs-on: ubuntu-latest
      permissions:
        id-token: write
        contents: write

      steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/retail-store-github-actions
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image tag
        run: |
          SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-8)
          echo "IMAGE_TAG=sha-${SHORT_SHA}" >> $GITHUB_ENV

      - name: Build and push multi-platform image
        uses: docker/build-push-action@v5
        with:
          context: ./src/ui
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:latest
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}

      - name: Update Helm values
        run: |
          sed -i "s|tag:.*|tag: \"${{ env.IMAGE_TAG }}\"|" \
            deploy/helm/retailstore/values-ui.yaml
          git config user.email "ci-bot@github.com"
          git config user.name "ci-bot"
          git add deploy/helm/retailstore/values-ui.yaml
          git commit -m "ci: update ui image to ${{ env.IMAGE_TAG }}"
          git push
  ```
- [ ] Replace `ACCOUNT_ID` and `YOUR_GITHUB_ORG` with your real values
- [ ] Commit and push:
  ```bash
  git add .github/
  git commit -m "ci: add GitHub Actions workflow for UI"
  git push
  ```

**Verify:**
- GitHub → Actions tab → workflow appears and runs
- Check it gets to at least the "Configure AWS credentials" step without error

---

### Day 45 — Debug + Fix CI Pipeline
**Goal:** Get the CI pipeline fully passing end-to-end.

**Tasks (2 hours):**
- [ ] Trigger a test run by making a change:
  ```bash
  echo "<!-- test $(date) -->" >> src/ui/README.md
  git add . && git commit -m "test: trigger CI" && git push
  ```
- [ ] Watch the GitHub Actions run in browser
- [ ] Common issues to fix:
  - OIDC trust policy `sub` condition doesn't match your repo path → fix the ARN
  - `docker buildx` not available → add `docker/setup-buildx-action@v3` step
  - `git push` fails with 403 → ensure `permissions: contents: write`
  - `sed` command fails → check the values YAML path
- [ ] Verify after CI passes:
  ```bash
  aws ecr describe-images \
    --repository-name retail-store/ui \
    --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags'
  # Shows ["latest", "sha-xxxxxxxx"]
  ```
- [ ] Verify Git was updated by ci-bot:
  ```bash
  git pull && git log --oneline -3
  # Should show: "ci: update ui image to sha-xxxxxxxx"
  ```

**Verify:**
```bash
aws ecr describe-images --repository-name retail-store/ui \
  --query 'length(imageDetails)'   # >= 1
```

---

### Day 46 — Install ArgoCD
**Goal:** Install ArgoCD and access its UI.

**Tasks (2 hours):**
- [ ] Install ArgoCD:
  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd \
    -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl wait --for=condition=available deployment/argocd-server \
    -n argocd --timeout=300s
  ```
- [ ] Get initial admin password:
  ```bash
  ARGOCD_PWD=$(kubectl get secret argocd-initial-admin-secret \
    -n argocd -o jsonpath="{.data.password}" | base64 -d)
  echo "ArgoCD password: $ARGOCD_PWD"
  ```
- [ ] Access the ArgoCD UI:
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443 &
  # Open: https://localhost:8080
  # Login: admin / $ARGOCD_PWD
  ```
- [ ] Install ArgoCD CLI:
  ```bash
  curl -sSL -o argocd-linux-amd64 \
    https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
  chmod +x argocd-linux-amd64 && sudo mv argocd-linux-amd64 /usr/local/bin/argocd
  argocd login localhost:8080 \
    --username admin --password $ARGOCD_PWD --insecure
  argocd version
  ```
- [ ] Explore the UI — there are no apps yet, just explore navigation

**Verify:**
```bash
argocd version   # shows both client and server version
kubectl get pods -n argocd | grep Running | wc -l   # 5+ pods Running
```

---

### Day 47 — Create ArgoCD Application (GitOps CD)
**Goal:** Connect ArgoCD to your Git repo and deploy the retail store.

**Tasks (2 hours):**
- [ ] Connect ArgoCD to your GitHub repo:
  ```bash
  argocd repo add https://github.com/YOUR_ORG/aws-devops-retail-store \
    --username YOUR_GITHUB_USER \
    --password YOUR_GITHUB_PAT    # create a PAT at github.com/settings/tokens
  ```
- [ ] Create the ArgoCD Application:
  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: retail-store
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://github.com/YOUR_ORG/aws-devops-retail-store
      targetRevision: main
      path: deploy/helm/retailstore
      helm:
        valueFiles:
        - values-dev.yaml
    destination:
      server: https://kubernetes.default.svc
      namespace: retail-store
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
      - CreateNamespace=true
  EOF
  ```
- [ ] Watch ArgoCD sync:
  ```bash
  argocd app get retail-store
  argocd app sync retail-store   # force first sync
  kubectl get pods -n retail-store -w
  ```
- [ ] Verify ArgoCD UI shows Synced + Healthy

**Verify:**
```bash
argocd app get retail-store | grep -E "Health Status|Sync Status"
# Health Status:  Healthy
# Sync Status:    Synced
```

---

### Day 48 — End-to-End GitOps Test
**Goal:** Prove the full pipeline works: code → ECR → Git → ArgoCD → EKS.

**Tasks (2 hours):**
- [ ] Make a visible code change:
  ```bash
  echo "<!-- v2 pipeline test $(date) -->" >> \
    src/ui/src/main/resources/templates/home.html
  git add . && git commit -m "test: v2 marker" && git push
  ```
- [ ] Watch each step in sequence:
  ```bash
  # Step 1: GitHub Actions builds (watch in browser → Actions tab)

  # Step 2: New image in ECR (run after ~2 min):
  aws ecr describe-images --repository-name retail-store/ui \
    --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags'

  # Step 3: ci-bot updated Git:
  git pull && git log --oneline -3

  # Step 4: ArgoCD detects change (within 3 min):
  watch argocd app get retail-store   # OutOfSync → Syncing → Synced

  # Step 5: New pod running:
  kubectl get pods -n retail-store -w
  ```
- [ ] Verify image tag on running pod matches Git SHA:
  ```bash
  kubectl get pod -n retail-store -l app=ui \
    -o jsonpath='{.items[0].spec.containers[0].image}'
  # Should contain sha-xxxxxxxx matching the latest git commit
  ```
- [ ] Test ArgoCD self-healing:
  ```bash
  kubectl scale deployment ui --replicas=1 -n retail-store
  sleep 200
  kubectl get deployment ui -n retail-store \
    -o jsonpath='{.spec.replicas}'   # ArgoCD restores to original value
  ```

**Verify:**
```bash
# Running pod image SHA matches last Git commit SHA
git log --oneline -1 | awk '{print $1}' | cut -c1-8
# AND:
kubectl get pod -n retail-store -l app=ui \
  -o jsonpath='{.items[0].spec.containers[0].image}' | grep -o 'sha-[a-f0-9]*'
# Both should match
```

---

### Day 49 — Week Review + Rollback Practice
**Goal:** Practice rollback scenarios and review all CI/CD concepts.

**Tasks (2 hours):**
- [ ] Practice ArgoCD rollback:
  ```bash
  argocd app history retail-store
  argocd app rollback retail-store <prev-revision>
  kubectl get pods -n retail-store -w   # old image re-deployed
  argocd app sync retail-store          # go back to latest
  ```
- [ ] Practice Helm rollback (for emergencies):
  ```bash
  helm history retail-store -n retail-store
  helm rollback retail-store 2 -n retail-store  # roll back 1 revision
  helm upgrade retail-store ./deploy/helm/retailstore -n retail-store  # go forward
  ```
- [ ] Review questions (answer without looking):
  - Why do we use `sha-xxxxxxxx` instead of `latest` as image tag?
  - What is the difference between `argocd app rollback` and `helm rollback`?
  - What does `selfHeal: true` in ArgoCD prevent?
  - Why does CI use OIDC instead of stored AWS access keys?
- [ ] **Cost save:** Review your monthly cost estimate in AWS billing

**Verify:**
```bash
argocd app get retail-store | grep "Sync Status"   # Synced
kubectl get pods -n retail-store | grep -v Running | grep -v NAME
# Nothing — all pods healthy
```

---

## WEEK 8 — Final Validation + Production Hardening (Days 50–56)

---

### Day 50 — Full System Validation
**Goal:** Run every verification check from the Master Checklists.

**Tasks (2 hours):**
- [ ] Infrastructure checks:
  ```bash
  kubectl get nodes                         # 3 Ready
  kubectl get pods -A | grep -v Running     # nothing
  kubectl get hpa -n retail-store           # targets not unknown
  kubectl get pdb -n retail-store           # disruptions allowed
  kubectl get ingress -n retail-store       # ALB address present
  argocd app get retail-store               # Synced + Healthy
  ```
- [ ] Application checks:
  ```bash
  ALB=$(kubectl get ingress retail-store -n retail-store \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  curl http://$ALB/catalogue | jq length       # products returned
  curl http://$ALB/health | jq .status         # healthy
  ```
- [ ] Security checks:
  ```bash
  kubectl exec -it deploy/catalog -n retail-store -- \
    env | grep AWS_ACCESS_KEY   # NOTHING — Pod Identity only
  kubectl exec -it deploy/catalog -n retail-store -- \
    cat /mnt/secrets/db-password   # real password from Secrets Manager
  ```
- [ ] Observability checks:
  ```bash
  aws xray get-trace-summaries \
    --start-time $(date -d '1 hour ago' +%s) \
    --end-time $(date +%s) \
    --query 'length(TraceSummaries)'   # > 0
  aws logs describe-log-groups \
    --log-group-name-prefix /aws/retail-store   # exists
  ```

**Verify:** All checks above pass without errors.

---

### Day 51 — HPA Load Testing
**Goal:** Prove autoscaling works under real load.

**Tasks (2 hours):**
- [ ] Baseline check:
  ```bash
  kubectl get hpa -n retail-store
  kubectl get pods -n retail-store | grep catalog | wc -l   # minReplicas
  ```
- [ ] Generate sustained load:
  ```bash
  kubectl run load-1 -n retail-store --image=busybox --restart=Never -- \
    /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue; done"
  kubectl run load-2 -n retail-store --image=busybox --restart=Never -- \
    /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue; done"
  ```
- [ ] Record observations:
  ```bash
  # Run this every 30 seconds for 5 minutes:
  watch -n 30 'kubectl get hpa catalog-hpa -n retail-store && \
    kubectl get pods -n retail-store | grep catalog | wc -l'
  ```
- [ ] Record: at what CPU% did scale-up trigger? How many replicas peaked?
- [ ] Stop load and time scale-down:
  ```bash
  kubectl delete pod load-1 load-2 -n retail-store
  # Record how many minutes until replicas return to minReplicas
  ```

**Verify:**
```bash
kubectl get hpa catalog-hpa -n retail-store
# REPLICAS peaked at > minReplicas during load test
```

---

### Day 52 — Spot Interruption Test
**Goal:** Simulate a Spot interruption and verify zero downtime.

**Tasks (2 hours):**
- [ ] Set up a continuous request loop to count errors:
  ```bash
  ALB=$(kubectl get ingress retail-store -n retail-store \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  # Run in background — logs error count
  for i in {1..1000}; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://$ALB/catalogue)
    [[ "$STATUS" != "200" ]] && echo "ERROR at request $i: HTTP $STATUS"
    sleep 0.5
  done &
  ```
- [ ] Simulate interruption by draining a node:
  ```bash
  # Pick a node that has catalog pods on it
  NODE=$(kubectl get pods -n retail-store -l app=catalog \
    -o jsonpath='{.items[0].spec.nodeName}')
  echo "Draining: $NODE"
  kubectl cordon $NODE
  kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --grace-period=30
  ```
- [ ] Observe:
  ```bash
  kubectl get pods -n retail-store -w   # pods migrate, PDB respected
  kubectl get nodes                     # Karpenter provisions replacement
  # Check the error count in the request loop output
  ```
- [ ] Restore:
  ```bash
  kubectl uncordon $NODE
  kill %1   # stop the request loop
  ```

**Verify:**
- Zero HTTP errors in the request loop during the drain
- PDB kept at least `minAvailable` pods running at all times
- Karpenter provisioned a replacement node automatically

---

### Day 53 — Add New Microservice (Advanced Exercise A-04)
**Goal:** Add the `recommendations` microservice end-to-end.

**Tasks (2 hours):**
- [ ] Write a minimal Node.js recommendations service:
  ```bash
  mkdir -p src/recommendations
  cat > src/recommendations/app.js <<'EOF'
  const express = require('express')
  const app = express()
  app.get('/recommendations', (req, res) => {
    res.json(["product-1", "product-2", "product-3"])
  })
  app.get('/health', (req, res) => res.json({status: 'ok'}))
  app.listen(8080, () => console.log('recommendations service on :8080'))
  EOF

  cat > src/recommendations/Dockerfile <<'EOF'
  FROM node:20-alpine
  WORKDIR /app
  RUN echo '{"name":"recommendations","version":"1.0.0"}' > package.json && \
      npm install express
  COPY app.js .
  EXPOSE 8080
  CMD ["node", "app.js"]
  EOF
  ```
- [ ] Create ECR repo and build locally:
  ```bash
  aws ecr create-repository --repository-name retail-store/recommendations
  docker build -t retail-store/recommendations:v1 src/recommendations/
  docker run -d -p 8082:8080 --name rec-test retail-store/recommendations:v1
  curl http://localhost:8082/recommendations
  docker stop rec-test && docker rm rec-test
  ```
- [ ] Add Helm templates for the new service
- [ ] Add GitHub Actions workflow for the new service
- [ ] Commit and push — verify CI builds it and ArgoCD deploys it

**Verify:**
```bash
kubectl get pods -n retail-store | grep recommendations   # Running
curl http://$ALB/recommendations   # returns product list
```

---

### Day 54 — Multi-Environment GitOps (Advanced Exercise A-05)
**Goal:** Implement dev auto-deploy and prod manual approval.

**Tasks (2 hours):**
- [ ] Create `values-prod.yaml` with prod settings:
  ```yaml
  catalog:
    replicaCount: 3
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
  ui:
    replicaCount: 3
  ```
- [ ] Create two ArgoCD Applications:
  ```bash
  # Dev — auto-sync
  kubectl apply -f - <<'EOF'
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: retail-store-dev
    namespace: argocd
  spec:
    source:
      path: deploy/helm/retailstore
      helm:
        valueFiles: [values-dev.yaml]
    destination:
      namespace: retail-store-dev
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  EOF

  # Prod — manual sync only
  kubectl apply -f - <<'EOF'
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: retail-store-prod
    namespace: argocd
  spec:
    source:
      path: deploy/helm/retailstore
      helm:
        valueFiles: [values-prod.yaml]
    destination:
      namespace: retail-store-prod
    syncPolicy: {}   # no automated sync
  EOF
  ```
- [ ] Test the promotion workflow:
  ```bash
  # Push a code change
  echo "<!-- v3 -->" >> src/ui/README.md
  git add . && git commit -m "test: v3 for promo" && git push

  # Dev auto-deploys (watch in ArgoCD UI)
  argocd app get retail-store-dev   # Synced automatically

  # Prod stays at old version
  argocd app get retail-store-prod  # OutOfSync (waiting for approval)

  # Manual prod sync
  argocd app sync retail-store-prod
  argocd app get retail-store-prod  # Synced
  ```

**Verify:**
```bash
argocd app list | grep retail-store
# retail-store-dev:  Synced   (auto-synced)
# retail-store-prod: OutOfSync (waiting for manual sync after code push)
```

---

### Day 55 — Cost Optimisation Review
**Goal:** Understand your AWS costs and apply optimisations.

**Tasks (2 hours):**
- [ ] Check current month's costs:
  ```bash
  aws ce get-cost-and-usage \
    --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=SERVICE,Key=SERVICE \
    --query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}' \
    --output table | sort -k3 -n
  ```
- [ ] Identify top 3 cost drivers — usually: EKS, EC2, RDS, NAT Gateway
- [ ] Apply cost saving measures:
  ```bash
  # Switch managed node group to Spot
  # (update instance_types in EKS Terraform)

  # Enable RDS storage autoscaling instead of over-provisioning

  # Use single NAT Gateway for non-prod
  # (set single_nat_gateway = true in VPC Terraform)
  ```
- [ ] Create a shutdown script for end-of-day:
  ```bash
  cat > ~/shutdown-infra.sh <<'EOF'
  #!/bin/bash
  echo "Scaling down EKS node group to 0..."
  aws eks update-nodegroup-config \
    --cluster-name retail-store-eks \
    --nodegroup-name default \
    --scaling-config minSize=0,maxSize=5,desiredSize=0
  echo "Done. Cost: ~$2.40/day for EKS control plane only."
  echo "To restore: aws eks update-nodegroup-config ... desiredSize=3"
  EOF
  chmod +x ~/shutdown-infra.sh
  ```

**Verify:**
- You know your current monthly cost
- You have a shutdown script to save costs overnight

---

### Day 56 — Final Checklist + Certificate of Completion
**Goal:** Complete all 156 checklist items and prove production-readiness.

**Tasks (2 hours):**
- [ ] Run through the Master Checklist in `Architecture_Patterns_and_AWS_Services.md`
- [ ] Score yourself:
  ```
  Checklist 1 — Architecture Understanding:  ___/30
  Checklist 2 — Project Setup:               ___/27
  Checklist 3 — Running the Project:         ___/27
  Checklist 4 — Modifying the Project:       ___/29
  Checklist 5 — Adding a New Feature:        ___/43
  TOTAL:                                     ___/156
  ```
- [ ] Answer the final validation questions:
  ```
  Q1: Draw the full CI/CD pipeline from git push to running pod (7 steps)
  Q2: Explain what happens during a Spot interruption (all 6 stages)
  Q3: How does a pod get its DB password without it being in Git?
  Q4: What would you check first if all catalog pods are in CrashLoopBackOff?
  Q5: How do you roll back a bad production deployment?
  ```
- [ ] **Celebrate:** You have built and understand a production-grade AWS DevOps system.

---

## Days 57–60 — Buffer Days
> Use these for anything that took longer than expected, or to explore extras:

- **Day 57:** Redo any exercise you found difficult
- **Day 58:** Set up Grafana dashboards for your Prometheus metrics
- **Day 59:** Implement Exercise A-02 (Prometheus alerting pipeline)
- **Day 60:** Write your own runbook documenting this system for a new team member

---

## Quick Reference — Daily Rhythm

```
First 15 min  │ Review what you learned yesterday (read your notes)
Next 90 min   │ Execute today's tasks
Last 15 min   │ Write 3 things you learned today in a notes file
```

## Cost-Saving Rules
```
End of each day  │ Run ~/shutdown-infra.sh (saves ~$9/day)
Start of each day│ aws eks update-nodegroup-config ... desiredSize=3
End of Week 3    │ terraform destroy on EKS (takes 15 min to rebuild)
Always running   │ VPC + S3 + DynamoDB (costs pennies per day)
```

## If You Get Stuck
```
30 minutes → copy the exact error message → Google it
60 minutes → post on Stack Overflow or AWS re:Post with full error
Never       → skip and move on (each step builds on the previous)
```
