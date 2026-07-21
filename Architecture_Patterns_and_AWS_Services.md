# Architecture Analysis — Ultimate DevOps Real-World Project (AWS Cloud)

---

## 1. Overall Business Purpose

The project implements a **Retail Store e-commerce platform** as a cloud-native, production-grade
microservices application deployed on AWS EKS (Elastic Kubernetes Service).

**Business goals it solves:**
- Handle variable traffic loads (low at night, peak on Black Friday) without manual intervention
- Ensure zero-downtime deployments — customers never see maintenance pages
- Independently scale and deploy each service without affecting others
- Full audit trail of every infrastructure and application change via Git
- Reduce operational overhead by offloading databases, queues, and caching to AWS managed services

---

## 2. High-Level System Design

```
                        ┌─────────────────────────────────┐
                        │           INTERNET               │
                        └────────────┬────────────────────┘
                                     │ HTTPS
                              ┌──────▼──────┐
                              │  Route 53   │  DNS: retailstore.stacksimplify.com
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │  AWS ALB    │  (created by AWS Load Balancer Controller)
                              └──────┬──────┘
                                     │
                   ┌─────────────────▼─────────────────┐
                   │         AWS EKS CLUSTER            │
                   │  ┌─────────────────────────────┐  │
                   │  │   Retail Store Application   │  │
                   │  │   (5 Microservices via Helm) │  │
                   │  └─────────────────────────────┘  │
                   └───────────────┬───────────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                         │
   ┌──────▼──────┐          ┌──────▼──────┐         ┌───────▼──────┐
   │  AWS RDS    │          │  AWS        │         │  AWS         │
   │  MySQL /    │          │  DynamoDB   │         │  ElastiCache │
   │  PostgreSQL │          │  (Cart)     │         │  Redis       │
   └─────────────┘          └─────────────┘         └──────────────┘
```

**VPC Layout:**
```
VPC: 10.0.0.0/16
├── Public Subnets  (AZ1/2/3) → ALB, NAT Gateway         [internet-facing]
└── Private Subnets (AZ1/2/3) → EKS Worker Nodes, Pods   [no direct internet]
```

---

## 3. Major Components and Their Responsibilities

### 3.1 Application Layer — 5 Microservices

| Service | Language | Responsibility |
|---|---|---|
| **UI** | Java 21, Spring Boot | Aggregates all backend services, serves web frontend on port 8080 |
| **Catalog** | Go, Gin, GORM | Product listings and inventory — reads/writes MySQL |
| **Cart** | Java 21, Spring Boot | Shopping cart state — uses DynamoDB as session store |
| **Checkout** | Node.js, Express | Orchestrates checkout flow — caches session in Redis |
| **Orders** | Java 21, Spring Boot | Persists orders to PostgreSQL, publishes events to SQS |

### 3.2 Data Layer — AWS Managed Services

| Service | AWS Resource | Responsibility |
|---|---|---|
| Catalog DB | AWS RDS MySQL 8.0 | Product and inventory relational data |
| Orders DB | AWS RDS PostgreSQL 15 | Financial order records (ACID compliance) |
| Cart Store | AWS DynamoDB | Key-value cart session storage (serverless, millisecond latency) |
| Cache | AWS ElastiCache Redis 7 | Checkout session caching, reduces DB load |
| Messaging | AWS SQS | Async order event delivery to downstream consumers |

### 3.3 Infrastructure Layer — Terraform Provisioned

| Component | Responsibility |
|---|---|
| **VPC** | Isolated network with public/private subnet separation across 3 AZs |
| **EKS Cluster** | Managed Kubernetes control plane (AWS manages etcd, API server, scheduler) |
| **Karpenter** | Just-in-time EC2 node provisioning — replaces Cluster Autoscaler |
| **EBS CSI Driver** | Dynamic persistent volume provisioning for stateful pods |
| **Pod Identity Agent** | Assigns IAM roles to pods without static AWS credentials |

### 3.4 Platform Layer — Kubernetes Add-ons

| Add-on | Responsibility |
|---|---|
| AWS Load Balancer Controller | Creates AWS ALB from Kubernetes Ingress objects automatically |
| External DNS | Auto-creates/updates Route53 records from Ingress annotations |
| Metrics Server | Provides `metrics.k8s.io` API — required for HPA to function |
| Secrets Store CSI Driver + ASCP | Mounts AWS Secrets Manager values into pods as files or env vars |
| Cert-Manager | Manages TLS certificates for internal components (ADOT webhooks) |
| ADOT Operator | Manages OpenTelemetry Collector lifecycle via Kubernetes CRDs |
| Kube State Metrics | Exposes Kubernetes object-level metrics (pod phase, deployment status) |
| Prometheus Node Exporter | Exposes EC2 node-level metrics (CPU, memory, disk) per worker node |

### 3.5 Observability Layer

| Signal | Collector Mode | Destination | Purpose |
|---|---|---|---|
| **Traces** | ADOT Deployment | AWS X-Ray | Visualize request flow and latency across microservices |
| **Logs** | ADOT DaemonSet | AWS CloudWatch Logs | Centralized log aggregation and search |
| **Metrics** | ADOT Deployment | Amazon Managed Prometheus → Grafana | Dashboards, SLO tracking, autoscaling input |

### 3.6 CI/CD Layer

| Tool | Responsibility |
|---|---|
| **GitHub Actions** | CI: build Docker image, push to AWS ECR, update Helm values file |
| **AWS ECR** | Private container registry — images never leave AWS network |
| **ArgoCD** | CD: polls Git every 3 minutes, applies Helm chart changes to EKS |

---

## 4. Data Flow Between Components

### 4.1 User Request Flow (Read)

```
1. User browser → HTTPS → Route53 DNS lookup
2. Route53 → AWS ALB (public subnet, internet-facing)
3. ALB → Kubernetes Ingress rule → UI ClusterIP Service
4. UI Pod → parallel fan-out to:
   ├── Catalog Service → Catalog Pod → RDS MySQL   (product listings)
   ├── Cart Service    → Cart Pod    → DynamoDB     (cart items)
   └── Checkout Service (only on checkout page)
5. UI aggregates all responses → renders HTML → returns to browser
```

### 4.2 Order Placement Flow (Write)

```
1. User submits order → UI → Checkout Service
2. Checkout → Orders Service (HTTP POST /orders)
3. Orders Service:
   ├── Writes order record → RDS PostgreSQL (synchronous, ACID transaction)
   └── Publishes OrderPlaced event → AWS SQS (asynchronous)
4. Downstream consumer reads SQS → triggers fulfillment / notification
```

### 4.3 Secrets Injection Flow

```
1. Pod scheduled on node
2. Pod Identity Agent intercepts AWS credential request from pod
3. EKS Pod Identity → returns temporary IAM role credentials
4. Secrets Store CSI Driver + ASCP → calls AWS Secrets Manager API
5. Secret value mounted at /mnt/secrets/ (file) AND synced to K8s Secret (env var)
6. App reads DB_PASSWORD from env var — zero hardcoded credentials anywhere
```

### 4.4 Observability Flow

```
App Pod (OTEL SDK auto-instrumented)
  │  OTLP gRPC port 4317
  ▼
ADOT Collector
  ├── Traces  → awsxray exporter          → AWS X-Ray       (request tracing)
  ├── Logs    → awscloudwatchlogs          → CloudWatch Logs (log aggregation)
  └── Metrics → prometheusremotewrite      → Amazon Managed Prometheus
                                                    │
                                           Amazon Managed Grafana
                                           (dashboards + alerts)
```

### 4.5 CI/CD Deployment Flow

```
Developer: git push to src/ui/src/**
  │
  ▼  GitHub Actions triggered (on: push)
  Step 1: Checkout source code
  Step 2: Configure AWS credentials via OIDC (no stored keys)
  Step 3: Login to Amazon ECR
  Step 4: Define image tags (latest + git-sha)
  Step 5: docker buildx build --platform linux/amd64,linux/arm64 → push to ECR
  Step 6: Update charts/values-ui.yaml → tag: sha-xxxx
  Step 7: git commit + git push → Helm chart repo (main branch)
  │
  ▼  ArgoCD polls Git every 3 minutes
  Step 8: Detects values-ui.yaml changed (image tag differs)
  Step 9: helm upgrade → EKS cluster (rolling update)
  Step 10: New pods start → readiness probe passes → old pods terminate
  Result: Zero-downtime deployment. Full git history = audit trail.
```

---

## 5. Communication Protocols

| Layer | Protocol | Detail |
|---|---|---|
| User → ALB | HTTPS / TLS 1.2+ | Internet-facing, ACM managed certificate |
| ALB → Pod | HTTP | TLS terminated at ALB; internal traffic unencrypted |
| Pod → Pod | HTTP / REST | Via Kubernetes ClusterIP Service (kube-proxy iptables rules) |
| App → ADOT Collector | gRPC (OTLP) | Port 4317 for traces/metrics; 4318 HTTP fallback |
| ADOT → X-Ray | HTTPS + AWS SigV4 | AWS API authentication via Pod Identity |
| ADOT → CloudWatch | HTTPS + AWS SigV4 | AWS API authentication via Pod Identity |
| ADOT → AMP | HTTPS + SigV4 + SigV4Auth Extension | Prometheus Remote Write protocol |
| Pods → RDS / DynamoDB / SQS / Secrets Manager | HTTPS + SigV4 | Via Pod Identity IAM role, through NAT Gateway |
| ArgoCD → GitHub | HTTPS + GitHub Token | Polls manifests repo every 3 minutes |
| GitHub Actions → ECR | HTTPS + OIDC | Federated identity, no long-lived credentials |
| EKS Nodes → ECR | HTTPS | Via NAT Gateway from private subnet |

---

## 6. Technology Stack

### Application
| Component | Technology |
|---|---|
| Web UI | Java 21, Spring Boot 3, Thymeleaf templates |
| Catalog API | Go 1.21, Gin HTTP framework, GORM ORM |
| Cart API | Java 21, Spring Boot 3, AWS SDK v2 for DynamoDB |
| Checkout API | Node.js 20, Express.js, ioredis |
| Orders API | Java 21, Spring Boot 3, Spring AMQP |
| Container Base | Amazon Linux 2023 (public.ecr.aws/amazonlinux/amazonlinux:2023) |

### Infrastructure
| Component | Technology |
|---|---|
| Cloud Provider | AWS (us-east-1, 3 AZs) |
| IaC | Terraform >= 1.6, AWS Provider ~> 5.0 |
| Container Runtime | containerd |
| Orchestration | Kubernetes 1.29+, AWS EKS |
| Node Provisioning | Karpenter v0.35+ |
| Package Manager | Helm v3 |
| State Backend | AWS S3 (versioned) + DynamoDB (locking) |

### Data
| Component | Technology |
|---|---|
| Relational (Catalog) | AWS RDS MySQL 8.0, Multi-AZ |
| Relational (Orders) | AWS RDS PostgreSQL 15, Multi-AZ |
| NoSQL (Cart) | AWS DynamoDB (on-demand capacity) |
| Cache (Checkout) | AWS ElastiCache Redis 7 |
| Message Queue | AWS SQS Standard Queue |

### Observability
| Component | Technology |
|---|---|
| Telemetry SDK | OpenTelemetry (auto-instrumentation per language) |
| Collector | AWS Distro for OpenTelemetry (ADOT) |
| Tracing Backend | AWS X-Ray |
| Log Backend | AWS CloudWatch Logs + CloudWatch Insights |
| Metrics Backend | Amazon Managed Prometheus (AMP) |
| Dashboards | Amazon Managed Grafana |
| K8s Metrics | Kube State Metrics + Prometheus Node Exporter + Metrics Server |

### CI/CD
| Component | Technology |
|---|---|
| Source Control | GitHub |
| CI Engine | GitHub Actions (OIDC auth to AWS) |
| Container Registry | AWS ECR (private, regional) |
| CD / GitOps Engine | ArgoCD (self-healing, drift correction) |

---

## 7. Right Architecture Patterns and Correct AWS Services

### Containerization Pattern
- Multi-stage Docker builds — small, secure production images
- Multi-platform builds (`linux/amd64` + `linux/arm64`) for Graviton node support
- AWS ECR as private registry — never Docker Hub in production

### Infrastructure as Code Pattern
- All AWS resources via Terraform — no manual console changes
- Remote state on S3 + DynamoDB locking for team collaboration
- Separate state files per project (VPC, EKS) — independent lifecycles
- Terraform Modules for reusable infrastructure blocks

### Security Pattern
- EKS Pod Identity — IAM roles per pod, no hardcoded access keys
- AWS Secrets Manager — encrypted, audited, rotatable credentials
- Non-root container user — `appuser` in every Dockerfile
- Nodes in private subnets — no direct internet exposure

### Resilience Pattern
- Multi-AZ deployment across 3 Availability Zones
- Topology Spread Constraints — pods never concentrated in one AZ
- PDB + Karpenter — zero-downtime node recycling and Spot interruption handling
- HPA — automatic pod scaling based on CPU/memory

### GitOps Pattern
- Git is the single source of truth for all deployments
- No developer has direct `kubectl apply` access to production
- ArgoCD `selfHeal` — reverts manual changes automatically
- Git history = complete audit trail of every deployment

---

## Summary — The Core Principle

> **Declare desired state in Git → Automated tooling reconciles to actual state → AWS managed services handle the undifferentiated heavy lifting.**

This is the foundation of a modern, production-grade cloud-native system.

# Right Architecture Patterns and Correct AWS Services

---

## Section 1 — Containerization Pattern

- Build Docker images using **multi-stage builds** to keep production images small and secure
- Use **multi-platform builds** (`linux/amd64` + `linux/arm64`) to support both Intel and Graviton nodes
- Push images to **AWS ECR** (private registry) — never Docker Hub in production
- Use **Docker Compose** for local development of all 10 containers together

---

## Section 2 — Infrastructure as Code Pattern

- All AWS resources provisioned via **Terraform** — no manual console clicks
- **Remote state** stored in S3 + DynamoDB locking — enables team collaboration
- **Separate state files** per project (VPC, EKS) — independent lifecycle management
- **Terraform Modules** for reusable, DRY infrastructure (VPC module, EKS module)
- **Remote state datasource** to share outputs between projects without tight coupling

---

## Section 3 — Networking Pattern

- **VPC** with public + private subnets across **3 Availability Zones** for high availability
- Worker nodes in **private subnets** — no direct internet exposure
- **ALB in public subnets** — only entry point for user traffic
- **NAT Gateway** — nodes can pull images/reach AWS APIs outbound, no inbound possible
- **Route Tables** explicitly separate public (IGW) and private (NAT) traffic paths

---

## Section 4 — Kubernetes Workload Pattern

- **Deployments** for stateless microservices (API pods are interchangeable)
- **StatefulSets** for stateful workloads that need stable identity
- **ClusterIP Services** for internal pod-to-pod communication
- **Topology Spread Constraints** to distribute pods across all 3 AZs
- **Pod Disruption Budgets (PDB)** to enforce minimum availability during disruptions

---

## Section 5 — Managed Dataplane Pattern

| Instead of (self-hosted) | Use (AWS managed) | Reason |
|---|---|---|
| MySQL StatefulSet | AWS RDS MySQL | Automated backups, Multi-AZ failover |
| PostgreSQL StatefulSet | AWS RDS PostgreSQL | ACID compliance, managed patching |
| Redis Deployment | AWS ElastiCache | Managed replication, no ops burden |
| RabbitMQ StatefulSet | AWS SQS | Fully serverless, no cluster management |
| DynamoDB Local | AWS DynamoDB | Serverless, millisecond latency, auto-scaling |

- Use **ExternalName Services** to bridge in-cluster DNS to external RDS endpoints — zero app code changes

---

## Section 6 — Security Pattern

- **EKS Pod Identity** — IAM roles assigned per pod, no hardcoded access keys
- **AWS Secrets Manager** — encrypted, audited, rotatable secrets
- **Secrets Store CSI Driver + ASCP** — secrets mounted as files or env vars at pod startup
- **Never store secrets in Kubernetes Secrets** (base64 is not encryption)
- **Never run containers as root** — use non-root `appuser` in Dockerfile

---

## Section 7 — Storage Pattern

- **EBS CSI Driver** for dynamic persistent volume provisioning
- **gp3 StorageClass** with `encrypted: true` — faster and cheaper than gp2
- `WaitForFirstConsumer` binding mode — EBS volume created in same AZ as pod
- `volumeClaimTemplates` in StatefulSets — one PVC + one EBS per replica automatically

---

## Section 8 — Load Balancer and DNS Pattern

- **AWS Load Balancer Controller** — creates ALB automatically from Kubernetes Ingress
- **Internet-facing ALB** in public subnets with `target-type: ip` (pod-level routing)
- **External DNS** — watches Ingress resources and auto-creates Route53 A records
- No manual DNS management — DNS is fully automated on every deployment

---

## Section 9 — Autoscaling Pattern

- **HPA (Horizontal Pod Autoscaler)** scales pods based on CPU/memory utilization
- **Metrics Server** provides the resource metrics API that HPA depends on
- **Karpenter** provisions new EC2 nodes just-in-time when pods are unschedulable
- **Spot NodePools** via Karpenter for up to 90% cost savings on worker nodes
- **EventBridge + SQS** delivers Spot interruption warnings to Karpenter for graceful drain

---

## Section 10 — Observability Pattern (Three Pillars)

- **Traces → AWS X-Ray** via ADOT Collector (Deployment mode) — distributed request tracing
- **Logs → AWS CloudWatch Logs** via ADOT Collector (DaemonSet mode) — centralized log aggregation
- **Metrics → Amazon Managed Prometheus → Amazon Managed Grafana** — dashboards and alerting
- **ADOT Operator** manages collector lifecycle via Kubernetes CRDs
- **k8s Attributes Processor** enriches all telemetry with pod/namespace/node metadata
- **Memory Limiter Processor** prevents collector OOM under high load

---

## Section 11 — GitOps CI/CD Pattern

- **GitHub Actions** for CI — triggered on code push, builds and pushes image to ECR
- **OIDC authentication** to AWS — no stored access keys in GitHub secrets
- **Git SHA as image tag** — every build is uniquely traceable to its source commit
- **CI updates Helm values file** with new image tag and commits back to Git
- **ArgoCD** for CD — polls Git every 3 minutes, auto-syncs Helm chart to EKS
- `selfHeal: true` — ArgoCD auto-reverts any manual `kubectl` changes (drift correction)
- `prune: true` — resources deleted from Git are automatically removed from cluster
- **Git history = complete audit trail** of every deployment, who changed what, when

---

## Summary — The Architecture Principle

> Every layer follows the same principle:
> **Declare desired state in Git → Automated tooling reconciles to actual state → AWS managed services handle the undifferentiated heavy lifting.**

This is the foundation of a modern, production-grade cloud-native system.

---

# Learning Roadmap

---

## Stage 1 — Beginner (Weeks 1–2)
> Goal: Understand containers and run the application locally

### Concepts You Must Understand First

**Linux & Command Line**
- File permissions, processes, environment variables
- `curl`, `grep`, `cat`, `ssh`, `export`

**Networking Fundamentals**
- What is an IP address, port, DNS, HTTP vs HTTPS
- What happens when you type a URL in a browser
- TCP/IP basics — why it matters for microservices

**Git Basics**
- `git clone`, `git add`, `git commit`, `git push`, `git pull`
- Branches and merge basics
- Why Git is the source of truth in GitOps

**Docker Core Concepts**
- What is a container vs a VM
- Docker image vs container
- `docker build`, `docker run`, `docker ps`, `docker logs`, `docker exec`
- Dockerfile instructions: `FROM`, `COPY`, `RUN`, `EXPOSE`, `CMD`, `ENTRYPOINT`
- Port mapping: `-p 8080:8080`
- Environment variables in containers: `-e`, `ENV`

**Docker Compose**
- What `docker-compose.yaml` is and why it replaces multiple `docker run` commands
- `depends_on` and healthchecks for startup ordering
- Named volumes and networks
- `docker compose up`, `docker compose down`, `docker compose logs`

**AWS Fundamentals**
- What is a Region and Availability Zone
- IAM: Users, Roles, Policies — principle of least privilege
- What is an EC2 instance, S3 bucket, and VPC at a high level
- AWS CLI: `aws configure`, basic `aws s3`, `aws ec2` commands

### Hands-On Practice
```bash
# Pull and run the UI service
docker pull stacksimplify/retail-store-sample-ui:1.0.0
docker run -p 8888:8080 stacksimplify/retail-store-sample-ui:1.0.0

# Run the full app locally
git clone https://github.com/aws-containers/retail-store-sample-app
cd retail-store-sample-app
docker compose up -d
docker compose ps
docker compose logs catalog
```

---

## Stage 2 — Intermediate (Weeks 3–5)
> Goal: Provision infrastructure and deploy the app to Kubernetes on AWS

### Terraform (IaC)
- HCL syntax: `resource`, `variable`, `output`, `data`, `locals`, `provider`, `terraform` blocks
- `terraform init` → `validate` → `plan` → `apply` → `destroy`
- Variable precedence: CLI flag > auto.tfvars > terraform.tfvars > env vars > default
- Local state vs Remote state (S3 + DynamoDB locking) — why remote state is mandatory in teams
- Terraform modules: root module vs child module, `source`, `module.name.output`
- Remote state datasource: sharing VPC outputs with EKS project

### AWS Networking
- VPC CIDR blocks, subnets, route tables
- Internet Gateway (public subnets) vs NAT Gateway (private subnets)
- Security Groups as stateful firewalls
- Why EKS worker nodes go in private subnets only

### Kubernetes Core (kubectl)
- Cluster architecture: control plane (API server, etcd, scheduler) vs worker nodes (kubelet, kube-proxy)
- Core objects: Pod, Deployment, ReplicaSet, Service (ClusterIP, NodePort, LoadBalancer)
- ConfigMap and Secret
- Namespaces
- `kubectl get`, `describe`, `logs`, `exec`, `apply`, `delete`, `port-forward`
- Labels and selectors — how Services find their Pods

### Kubernetes Storage
- PersistentVolume (PV) vs PersistentVolumeClaim (PVC) vs StorageClass
- EBS CSI Driver: how AWS EBS volumes are dynamically provisioned
- StatefulSet vs Deployment — when to use each
- `emptyDir` (ephemeral) vs EBS-backed PVC (persistent)

### Helm
- Why Helm exists: template + values = Kubernetes manifests
- `helm install`, `helm upgrade`, `helm rollback`, `helm list`, `helm history`
- `values.yaml` — override per environment
- Chart structure: `templates/`, `values.yaml`, `Chart.yaml`

### AWS Load Balancer + DNS
- How AWS Load Balancer Controller creates ALBs from Ingress objects
- ALB annotations: `scheme: internet-facing`, `target-type: ip`
- How External DNS writes Route53 records from Ingress hostnames

### Hands-On Practice
```bash
# Provision VPC
cd 01_VPC_terraform-manifests
terraform init && terraform apply

# Provision EKS cluster
cd ../02_EKS_terraform-manifests
terraform init && terraform apply

# Connect kubectl to EKS
aws eks update-kubeconfig --region us-east-1 --name retail-store-eks
kubectl get nodes

# Deploy catalog microservice
kubectl apply -f catalog-deployment.yaml
kubectl apply -f catalog-mysql-statefulset.yaml
kubectl get all

# Deploy full app with Helm
helm install retail-store ./charts/retail-store \
  --namespace retail-store --create-namespace
helm list -A
```

---

## Stage 3 — Advanced (Weeks 6–8)
> Goal: Add production-grade security, autoscaling, observability, and full CI/CD

### Security — Secrets Management
- Why Kubernetes Secrets are not secure (base64 ≠ encryption)
- AWS Secrets Manager: create, version, rotate secrets
- EKS Pod Identity: how pods get IAM credentials without access keys
- Secrets Store CSI Driver + ASCP: how secrets are mounted into pods
- `SecretProviderClass` CRD — maps AWS secret to pod volume

### Autoscaling — Two Levels
- **HPA**: formula, `minReplicas`, `maxReplicas`, `targetCPUUtilization`
- Metrics Server as the HPA data source
- **Karpenter**: NodePool, EC2NodeClass, consolidation policy
- Spot instances: pricing model, interruption risk, 2-minute warning
- Spot interruption handling: EventBridge → SQS → Karpenter → cordon + drain
- Pod Disruption Budget: `minAvailable` vs `maxUnavailable`, eviction enforcement

### Observability — OpenTelemetry + AWS
- OpenTelemetry concepts: traces, spans, metrics, logs, OTLP protocol
- ADOT Collector pipeline: receivers → processors → exporters → service
- Collector deployment modes: Deployment, DaemonSet, StatefulSet, Sidecar
- AWS X-Ray: trace map, segment timeline, latency percentiles
- CloudWatch Logs: log groups, log streams, CloudWatch Insights queries
- Amazon Managed Prometheus (AMP): remote write, PromQL queries
- Amazon Managed Grafana: data source setup, dashboard creation, alerting

### CI/CD — GitOps Pipeline
- GitHub Actions: workflow triggers, jobs, steps, secrets, `GITHUB_TOKEN`
- OIDC authentication to AWS: why it's safer than stored access keys
- AWS ECR: repository creation, image lifecycle policies, VPC endpoint
- Multi-platform Docker builds: `docker buildx`, QEMU emulation
- ArgoCD: Application CRD, sync policy, `selfHeal`, `prune`, drift detection
- GitOps principle: Git as the only path to deploy — no direct kubectl in production

### Hands-On Practice
```bash
# Test HPA
kubectl autoscale deployment catalog --cpu-percent=50 --min=3 --max=10
kubectl run load-gen --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://catalog; done"
kubectl get hpa -w

# Verify Karpenter node provisioning
kubectl get nodes -w

# Check ArgoCD sync status
argocd app get retail-store
argocd app diff retail-store
argocd app history retail-store

# Query X-Ray traces
aws xray get-trace-summaries \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s)

# Query CloudWatch logs
aws logs filter-log-events \
  --log-group-name /aws/containerinsights/retail-store-eks/application \
  --filter-pattern "ERROR" \
  --limit 20
```

---

## Stage 4 — Recommended Learning Order (Complete Sequence)

```
Week 1
  Day 1-2 │ Linux CLI + Git + AWS account setup + IAM basics
  Day 3-4 │ Docker commands + Dockerfile + run UI container on EC2
  Day 5-7 │ Docker Compose + run all 10 containers locally

Week 2
  Day 1-3 │ Terraform basics: HCL syntax + local state + provision VPC manually
  Day 4-5 │ Terraform remote state + S3 backend + DynamoDB lock
  Day 6-7 │ Terraform modules + refactor VPC to module

Week 3
  Day 1-2 │ AWS VPC deep dive: subnets, route tables, IGW, NAT GW
  Day 3-5 │ Terraform EKS cluster: VPC project + EKS project + remote state datasource
  Day 6-7 │ kubectl basics + connect to EKS + explore cluster

Week 4
  Day 1-3 │ Kubernetes core: Deployments, Services, ConfigMaps, Secrets, Namespaces
  Day 4-5 │ Kubernetes storage: PV, PVC, StorageClass, EBS CSI Driver
  Day 6-7 │ Deploy catalog microservice manually with kubectl

Week 5
  Day 1-2 │ Helm: chart structure, values, install/upgrade/rollback
  Day 3-4 │ AWS Load Balancer Controller + Ingress + ALB creation
  Day 5-6 │ External DNS + Route53 auto-registration
  Day 7   │ Deploy full app with Helm + verify DNS

Week 6
  Day 1-2 │ EKS Pod Identity + AWS Secrets Manager + Secrets Store CSI Driver
  Day 3-4 │ AWS RDS MySQL/PostgreSQL + ExternalName Service pattern
  Day 5-7 │ Karpenter NodePools + Spot instances + PDB + interruption handling

Week 7
  Day 1-2 │ HPA + Metrics Server + load testing
  Day 3-5 │ ADOT Operator + traces to X-Ray + view trace map
  Day 6-7 │ ADOT logs to CloudWatch + Insights queries

Week 8
  Day 1-2 │ ADOT metrics to AMP + Grafana dashboards
  Day 3-5 │ GitHub Actions CI pipeline: OIDC + ECR + multi-platform build
  Day 6-7 │ ArgoCD CD pipeline: Application + auto-sync + drift correction
           │ End-to-end GitOps test: code change → ECR → Git → ArgoCD → EKS
```

---

## Stage 5 — Skills Validation Checklist

After completing the roadmap, you should be able to answer YES to all of these:

### Infrastructure
- [ ] Can you provision a VPC + EKS cluster from scratch using Terraform with remote state?
- [ ] Can you explain why worker nodes are in private subnets?
- [ ] Can you recover from a corrupted Terraform state?

### Kubernetes
- [ ] Can you deploy a microservice, debug a CrashLoopBackOff, and fix it?
- [ ] Can you explain the difference between Deployment and StatefulSet?
- [ ] Can you write a PDB that guarantees 3 pods minimum during node drain?

### Security
- [ ] Can you inject a secret from AWS Secrets Manager into a pod without storing it in Git?
- [ ] Can you explain how EKS Pod Identity works and what it replaces?

### Autoscaling
- [ ] Can you configure HPA and generate load to trigger it?
- [ ] Can you explain the 6-stage Spot interruption handling flow?

### Observability
- [ ] Can you find a slow endpoint using X-Ray trace map?
- [ ] Can you write a CloudWatch Insights query to find all ERRORs in the last hour?
- [ ] Can you create a Grafana dashboard showing request rate per microservice?

### CI/CD
- [ ] Can you explain why OIDC is safer than stored AWS access keys in GitHub?
- [ ] Can you trace a code commit all the way to a running pod in EKS?
- [ ] Can you roll back a bad deployment using ArgoCD?

---

# Implementation Phases

---

## Phase 1 — Environment Setup

### Objective
Get every required tool installed and verified, configure AWS access, bootstrap Terraform remote state infrastructure, and run the full 10-container retail store application locally using Docker Compose — all before touching any cloud infrastructure.

---

### Concepts to Learn

| Concept | Why It Matters |
|---|---|
| **Linux CLI** | All DevOps tooling is CLI-first; no GUI in production |
| **Environment variables** | How tools (AWS CLI, Terraform, Docker) read config |
| **IAM Users vs Roles** | Users = long-term credentials (dev); Roles = temporary credentials (prod) |
| **AWS Regions and AZs** | Your resources live in a specific region — wrong region = not found |
| **Docker image vs container** | Image = blueprint; Container = running instance |
| **Docker Compose** | Orchestrates multiple containers with one YAML file locally |
| **Terraform remote state** | S3 stores state; DynamoDB prevents two people applying at the same time |
| **S3 bucket versioning** | Allows recovery of previous Terraform state if corrupted |

---

### Tasks to Perform

#### 1.1 Local Workstation Setup

```bash
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
aws --version

# Terraform
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo dnf install terraform -y
terraform -version

# kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Docker
sudo dnf install docker -y
sudo systemctl start docker
sudo usermod -aG docker $USER
docker --version

# eksctl (optional but useful)
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
```

### 1.2 AWS Account Configuration

```bash
# Configure AWS CLI with your IAM user credentials
aws configure
# AWS Access Key ID:     [your key]
# AWS Secret Access Key: [your secret]
# Default region:        us-east-1
# Default output format: json

# Verify access
aws sts get-caller-identity
aws ec2 describe-regions --query 'Regions[].RegionName' --output table
```

### 1.3 Terraform Remote State Bootstrap (One-Time)

```bash
# Create S3 bucket for Terraform state
aws s3api create-bucket \
  --bucket tfstate-dev-us-east-1-$(openssl rand -hex 4) \
  --region us-east-1

# Enable versioning (required — allows state recovery)
aws s3api put-bucket-versioning \
  --bucket <your-bucket-name> \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket <your-bucket-name> \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name dev-remote-state-locking \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### 1.4 Run Application Locally with Docker Compose

```bash
git clone https://github.com/aws-containers/retail-store-sample-app
cd retail-store-sample-app
docker compose up -d

# Verify all 10 containers are healthy
docker compose ps

# Access the app
curl http://localhost:8888
# or open http://localhost:8888 in browser

# Tear down when done
docker compose down -v
```

### Expected Outcome
- All 6 CLI tools (`aws`, `terraform`, `kubectl`, `helm`, `docker`, `eksctl`) installed and version-verified
- `aws sts get-caller-identity` returns your AWS account ID and IAM user ARN
- S3 bucket with versioning + encryption, and DynamoDB lock table both exist
- `docker compose ps` shows all 10 containers in `healthy` state
- Retail store UI fully functional at `http://localhost:8888` — you can browse products and add to cart

---

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| Using root AWS account for CLI | Security risk — root has unlimited power | Create a dedicated IAM user with least-privilege policy |
| Skipping S3 bucket versioning | Corrupted state = unrecoverable infrastructure | Always enable versioning before first `terraform apply` |
| Using same S3 bucket name as someone else | Bucket names are globally unique — creation fails | Add a random hex suffix: `$(openssl rand -hex 4)` |
| Not logging out and back in after `usermod -aG docker` | `docker` commands fail with permission denied | Run `newgrp docker` or log out/in to pick up group |
| Running `docker compose up` without checking port conflicts | Port 8080/8888 already in use = containers crash | Run `ss -tlnp \| grep 8080` before starting |
| Storing AWS access keys in code or `.env` files committed to Git | Credential leak → account compromise | Use `aws configure` which writes to `~/.aws/credentials` — gitignored |

---

### Phase 1 — Exit Criteria
- [ ] All CLI tools installed and version-verified
- [ ] `aws sts get-caller-identity` returns your account ID
- [ ] S3 bucket + DynamoDB table for Terraform state exist
- [ ] `docker compose up` runs all 10 containers successfully
- [ ] Retail store UI accessible at `http://localhost:8888`

---

## Phase 2 — Core Services (VPC + EKS Cluster)

### Objective
Provision a production-grade AWS network (VPC with public/private subnets across 3 AZs) and a fully functional EKS cluster using Terraform with remote state — with all required add-ons (Load Balancer Controller, Karpenter, Secrets Store CSI, External DNS) installed and healthy.

---

### Concepts to Learn

| Concept | Why It Matters |
|---|---|
| **VPC CIDR blocks** | Defines the IP address space for your entire network |
| **Public vs Private subnets** | Public = internet-reachable; Private = internal only |
| **Internet Gateway** | Allows public subnet resources to reach the internet |
| **NAT Gateway** | Allows private subnet pods to reach internet (one-way outbound) |
| **Route Tables** | Controls where network traffic is directed per subnet |
| **Terraform modules** | Reusable blocks of infrastructure — VPC and EKS are separate modules |
| **Terraform remote state datasource** | EKS project reads VPC outputs without hardcoding values |
| **EKS control plane vs worker nodes** | AWS manages the control plane; you pay for and manage worker EC2s |
| **EKS Add-ons** | AWS-managed plugins (EBS CSI, Pod Identity Agent) with automatic updates |
| **Helm controllers** | Controllers installed via Helm (AWS LBC, Karpenter, External DNS) |
| **kubeconfig** | File that tells `kubectl` which cluster to talk to and how to authenticate |

---

### Tasks to Perform

#### 2.1 Provision VPC

```bash
cd 01_VPC_terraform-manifests

# Update backend config with your S3 bucket name
# c1-versions.tf → backend "s3" { bucket = "your-bucket" }

terraform init
terraform validate
terraform plan -out=vpc.tfplan
terraform apply vpc.tfplan

# Verify outputs
terraform output
# Expected: vpc_id, public_subnet_ids, private_subnet_ids, azs
```

**What gets created:**
```
VPC: 10.0.0.0/16
├── Public Subnets:  10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24  (AZ1/2/3)
├── Private Subnets: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
├── Internet Gateway
├── NAT Gateway (1 per AZ for HA)
└── Route Tables (public → IGW, private → NAT)
```

### 2.2 Provision EKS Cluster

```bash
cd ../02_EKS_terraform-manifests
terraform init
terraform plan -out=eks.tfplan
terraform apply eks.tfplan
# Takes ~15 minutes

# Connect kubectl to the new cluster
aws eks update-kubeconfig \
  --region us-east-1 \
  --name $(terraform output -raw eks_cluster_name)

# Verify cluster is healthy
kubectl get nodes          # should show 3 worker nodes
kubectl get pods -A        # should show system pods running
kubectl cluster-info
```

**What gets created:**
```
EKS Cluster (control plane managed by AWS)
├── Managed Node Group (3 x t3.medium across 3 AZs)
├── EKS Add-ons:
│   ├── Pod Identity Agent
│   ├── EBS CSI Driver
│   ├── CoreDNS
│   └── kube-proxy
└── Helm Controllers:
    ├── AWS Load Balancer Controller
    ├── Karpenter
    ├── Secrets Store CSI Driver + ASCP
    └── External DNS
```

### 2.3 Verify Add-ons

```bash
kubectl get pods -n kube-system
kubectl get pods -n karpenter

# Verify AWS Load Balancer Controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# Verify Secrets Store CSI Driver
kubectl get daemonset -n kube-system secrets-store-csi-driver
```

### Expected Outcome
- `terraform output` returns VPC ID, 3 public subnet IDs, 3 private subnet IDs
- EKS cluster visible in AWS console with status `Active`
- `kubectl get nodes` shows 3 worker nodes all in `Ready` state
- `kubectl get pods -A` shows all system pods in `Running` state — no `Pending` or `CrashLoopBackOff`
- AWS Load Balancer Controller, Karpenter, Secrets Store CSI Driver, External DNS all confirmed running

---

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| Forgetting to update the S3 backend bucket name in `c1-versions.tf` | `terraform init` fails or writes to wrong bucket | Replace placeholder bucket name before `terraform init` |
| Running EKS project before VPC project | EKS can't read remote state → apply fails | Always apply VPC first, verify outputs, then apply EKS |
| Running `terraform apply` without `terraform plan` first | Unexpected resource deletion/replacement | Always review plan output; look for red `destroy` lines |
| Not waiting for EKS cluster to fully stabilize | Add-on installations fail intermittently | Wait for `kubectl get nodes` to show all 3 nodes `Ready` before proceeding |
| Applying both VPC and EKS in the same Terraform project | Can't destroy EKS independently of VPC | Keep them as separate state files — separate projects |
| Forgetting `aws eks update-kubeconfig` after cluster creation | `kubectl` connects to wrong/no cluster | Always run this immediately after `terraform apply` on EKS |
| Opening 0.0.0.0/0 on worker node security groups | EKS nodes exposed directly to internet | Nodes in private subnets should only accept traffic from ALB and within VPC |

---

### Phase 2 — Exit Criteria
- [ ] `terraform output` shows VPC ID and subnet IDs
- [ ] `kubectl get nodes` shows 3 nodes in `Ready` state
- [ ] `kubectl get pods -A` shows all system pods `Running`
- [ ] AWS Load Balancer Controller pod is `Running`
- [ ] Karpenter pod is `Running`

---

## Phase 3 — Database Integration

### Objective
Provision all five AWS managed data services (RDS MySQL, RDS PostgreSQL, DynamoDB, ElastiCache Redis, SQS), store all credentials securely in AWS Secrets Manager, and wire each microservice to its database using Kubernetes ExternalName Services — with zero credentials stored in Git or Kubernetes Secrets.

---

### Concepts to Learn

| Concept | Why It Matters |
|---|---|
| **AWS RDS Multi-AZ** | Synchronous standby replica — automatic failover < 60 seconds |
| **RDS Subnet Groups** | Tells RDS which subnets to use — must be private subnets |
| **RDS Security Groups** | Controls which resources can connect to the DB — only EKS pods |
| **DynamoDB on-demand capacity** | No provisioned throughput — pay per request, scales automatically |
| **ElastiCache Subnet Groups** | Same principle as RDS — Redis must be in private subnets |
| **SQS Standard vs FIFO** | Standard = at-least-once, unordered; FIFO = exactly-once, ordered |
| **AWS Secrets Manager** | Encrypted, audited, rotatable secret store — the correct place for DB passwords |
| **Kubernetes ExternalName Service** | Maps an in-cluster DNS name to an external hostname (RDS endpoint) |
| **EKS Pod Identity** | Grants IAM role to a pod via ServiceAccount — no access keys needed |
| **Secrets Store CSI Driver** | Kubernetes volume driver that fetches secrets from AWS Secrets Manager at pod start |
| **SecretProviderClass CRD** | Defines which secret to fetch and how to expose it (file or K8s Secret) |

---

### Tasks to Perform

#### 3.1 Provision AWS Managed Data Services (Terraform)

```bash
cd 14_AWS_Managed_Dataplane/terraform-manifests
terraform init && terraform apply
```

**Resources created:**
```
RDS MySQL 8.0        → catalog microservice
RDS PostgreSQL 15    → orders microservice
DynamoDB Table       → cart microservice
ElastiCache Redis 7  → checkout microservice
SQS Standard Queue   → orders async messaging
```

### 3.2 Store Database Credentials in AWS Secrets Manager

```bash
# Catalog MySQL credentials
aws secretsmanager create-secret \
  --name "prod/retail-store/catalog-db" \
  --secret-string '{
    "username": "catalog_user",
    "password": "YourSecurePassword123!",
    "host": "catalog.abc123.us-east-1.rds.amazonaws.com",
    "port": "3306",
    "dbname": "catalog"
  }'

# Orders PostgreSQL credentials
aws secretsmanager create-secret \
  --name "prod/retail-store/orders-db" \
  --secret-string '{
    "username": "orders_user",
    "password": "YourSecurePassword456!",
    "host": "orders.xyz789.us-east-1.rds.amazonaws.com",
    "port": "5432",
    "dbname": "orders"
  }'
```

### 3.3 Wire Microservices to Databases via ExternalName Services

```yaml
# catalog-mysql-externalname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-mysql        # app uses this DNS name — unchanged
  namespace: retail-store
spec:
  type: ExternalName
  externalName: catalog.abc123.us-east-1.rds.amazonaws.com
  ports:
  - port: 3306
```

```bash
kubectl apply -f catalog-mysql-externalname-service.yaml
kubectl apply -f orders-postgresql-externalname-service.yaml

# Verify DNS resolution from inside cluster
kubectl run dns-test --image=busybox --restart=Never -- \
  nslookup catalog-mysql.retail-store.svc.cluster.local
kubectl delete pod dns-test
```

### 3.4 Verify Database Connectivity

```bash
# Test catalog → RDS MySQL
kubectl exec -it deploy/catalog -n retail-store -- \
  curl http://localhost:8080/health

# Check catalog API can read products (proves DB connection works)
kubectl port-forward svc/catalog 8080:8080 -n retail-store &
curl http://localhost:8080/catalogue | jq length
```

### Expected Outcome
- All 5 AWS data services show `available` / `active` status in AWS console
- Each DB secret exists in AWS Secrets Manager with correct JSON structure
- ExternalName Services resolve correctly from inside the cluster
- Catalog API at `/catalogue` returns a JSON array of products (proves RDS MySQL working)
- Orders API health check returns 200 (proves RDS PostgreSQL working)
- DynamoDB table visible in AWS console with correct key schema

---

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| Putting RDS in public subnets | DB directly internet-accessible — critical security risk | RDS subnet group must use private subnets only |
| Not setting RDS security group to allow only EKS nodes | Anyone in VPC can connect to DB | Restrict inbound to the EKS worker node security group on port 3306/5432 |
| Wrong secret JSON structure in Secrets Manager | App fails to parse secret at startup → `CrashLoopBackOff` | Match exactly what the app expects: `{"username": ..., "password": ..., "host": ...}` |
| Using ClusterIP Service instead of ExternalName for RDS | ClusterIP tries to route to pods that don't exist | ExternalName is a DNS alias — no selector, no pods |
| Forgetting to create Pod Identity Association | CSI driver can't call Secrets Manager → pod stuck in `Init` | Associate ServiceAccount → IAM Role in EKS Pod Identity console |
| Hardcoding DB password in `values.yaml` or ConfigMap | Password in Git history → credential leak | Always use Secrets Manager + SecretProviderClass |
| DynamoDB table name mismatch | Cart service throws `ResourceNotFoundException` | Table name in Terraform output must exactly match the env var the app reads |

---

### Phase 3 — Exit Criteria
- [ ] All 5 AWS data services show `available` status in AWS console
- [ ] Secrets created in AWS Secrets Manager for each DB
- [ ] ExternalName Services created for RDS endpoints
- [ ] Catalog API returns product list (proves MySQL connection)
- [ ] Orders API health check passes (proves PostgreSQL connection)

---

## Phase 4 — APIs and Communication

### Objective
Deploy all 5 microservices as a complete Helm release, establish secure inter-service communication, expose the application via AWS ALB with HTTPS, and have Route53 automatically register the public DNS name — so the retail store is fully functional at a public URL.

---

### Concepts to Learn

| Concept | Why It Matters |
|---|---|
| **Helm chart structure** | `templates/` + `values.yaml` = parameterized Kubernetes manifests |
| **Helm values override** | `values-aws-dataplane.yaml` overrides defaults for AWS managed DB endpoints |
| **Kubernetes ClusterIP Service** | Stable internal DNS name for pod-to-pod communication |
| **Kubernetes Ingress** | Single entry point that routes HTTP/HTTPS to multiple services |
| **AWS ALB Ingress annotations** | Annotations tell the ALB controller how to configure the load balancer |
| **ALB target-type: ip** | Routes directly to pod IPs, bypasses kube-proxy — better performance |
| **External DNS** | Controller that watches Ingress objects and writes Route53 records automatically |
| **ACM certificates** | AWS Certificate Manager — free TLS certificates for your domain |
| **SecretProviderClass** | Per-microservice declaration of which Secrets Manager secret to mount |
| **Readiness vs Liveness probes** | Readiness: is pod ready for traffic? Liveness: is pod still alive? Both prevent bad deploys |
| **Rolling update strategy** | Kubernetes replaces pods one at a time — zero downtime if probes are correct |

---

### Tasks to Perform

### 4.1 Deploy EKS Pod Identity + SecretProviderClass

```yaml
# SecretProviderClass for catalog DB
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: catalog-db-secret
  namespace: retail-store
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/retail-store/catalog-db"
        objectType: "secretsmanager"
        jmesPath:
          - path: password
            objectAlias: db-password
          - path: host
            objectAlias: db-host
  secretObjects:
  - secretName: catalog-db-k8s-secret
    type: Opaque
    data:
    - objectName: db-password
      key: DB_PASSWORD
    - objectName: db-host
      key: DB_HOST
```

```bash
kubectl apply -f secret-provider-class-catalog.yaml
kubectl apply -f secret-provider-class-orders.yaml
```

### 4.2 Deploy All Microservices via Helm

```bash
helm install retail-store ./charts/retail-store \
  --namespace retail-store \
  --create-namespace \
  --values values-aws-dataplane.yaml \
  --set global.region=us-east-1

# Verify all pods are running
kubectl get pods -n retail-store
# Expected: catalog, carts, checkout, orders, ui — all Running

# Watch rollout
kubectl rollout status deployment/catalog -n retail-store
kubectl rollout status deployment/carts -n retail-store
kubectl rollout status deployment/checkout -n retail-store
kubectl rollout status deployment/orders -n retail-store
kubectl rollout status deployment/ui -n retail-store
```

### 4.3 Expose App via Ingress + ALB + Route53

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store-ingress
  namespace: retail-store
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    external-dns.alpha.kubernetes.io/hostname: retailstore.yourdomain.com
spec:
  rules:
  - host: retailstore.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ui
            port:
              number: 8080
```

```bash
kubectl apply -f ingress.yaml

# Watch ALB being created (takes ~2 minutes)
kubectl get ingress -n retail-store -w

# Verify DNS record in Route53
aws route53 list-resource-record-sets \
  --hosted-zone-id <YOUR_ZONE_ID> \
  --query "ResourceRecordSets[?Name=='retailstore.yourdomain.com.']"

# Test end-to-end
curl https://retailstore.yourdomain.com/health
```

### 4.4 Verify Inter-Service Communication

```bash
# Test each service endpoint from inside the cluster
kubectl run curl-test --image=curlimages/curl --restart=Never -n retail-store -- \
  curl -s http://catalog/catalogue | head -c 200
kubectl delete pod curl-test -n retail-store

# Check service endpoints are populated (proves pods are registered)
kubectl get endpoints -n retail-store

# Check secrets are mounted correctly
kubectl exec -it deploy/catalog -n retail-store -- \
  ls /mnt/secrets/
kubectl exec -it deploy/catalog -n retail-store -- \
  env | grep DB_
```

### Expected Outcome
- All 5 microservice Deployments show `READY 3/3` (or configured replica count)
- `kubectl get endpoints -n retail-store` shows populated IP:port pairs for every service
- `kubectl get ingress -n retail-store` shows an ALB DNS hostname (not `<pending>`)
- Route53 A record created automatically — `dig retailstore.yourdomain.com` resolves to ALB
- `https://retailstore.yourdomain.com` loads with valid TLS certificate
- Full user journey works: browse products → add to cart → checkout → order placed

---

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| Ingress stuck at `<pending>` | ALB not created | Check AWS LBC pod logs: `kubectl logs -n kube-system deploy/aws-load-balancer-controller` |
| ALB created but Route53 record not appearing | External DNS misconfigured or wrong hosted zone ID | Check External DNS pod logs: `kubectl logs deploy/external-dns` |
| Pods `Running` but endpoints empty | Pod failing readiness probe | `kubectl describe pod <name>` → look at Events section for probe failure reason |
| `CrashLoopBackOff` on microservice pod | Secret not mounted correctly | `kubectl logs <pod> --previous` and check for missing env vars or connection refused |
| Using `NodePort` service type instead of `ClusterIP` | Exposes ports directly on EC2 nodes — bypasses ALB | Only use `ClusterIP` for internal services; expose via Ingress |
| Not setting resource `requests` and `limits` | HPA cannot calculate utilization percentage | Always set `resources.requests.cpu` — required for HPA to work |
| Helm install fails midway and leaves partial resources | Second `helm install` fails with "already exists" | Run `helm uninstall retail-store` first, then reinstall cleanly |

---

### Phase 4 — Exit Criteria
- [ ] All 5 microservice pods in `Running` state
- [ ] All 5 services have populated `Endpoints`
- [ ] Ingress shows ALB DNS name
- [ ] Route53 A record exists for your domain
- [ ] `https://retailstore.yourdomain.com` loads the retail store UI
- [ ] Can add item to cart and complete checkout flow end-to-end

---

## Phase 5 — Testing

### Objective
Systematically validate every production concern before enabling automated deployments: functional correctness, autoscaling behaviour, resilience to node failure, secrets security, observability pipeline, and rollback capability.

---

### Concepts to Learn

| Concept | Why It Matters |
|---|---|
| **HPA scale-up trigger** | CPU must exceed target for 3 consecutive 15-second intervals before scaling |
| **HPA scale-down delay** | Default 5-minute cooldown prevents flapping — normal to wait |
| **kubectl cordon** | Marks node unschedulable — no new pods land on it |
| **kubectl drain** | Evicts all pods from node, respects PDB — used before node termination |
| **PDB eviction enforcement** | `kubectl drain` will block/wait if draining would violate `minAvailable` |
| **Kubernetes probe failure** | Readiness failure → pod removed from Service endpoints (no traffic) |
| **X-Ray trace sampling** | Not every request is traced by default — configure sampling rate |
| **CloudWatch Insights query syntax** | SQL-like language for querying structured log data |
| **Helm revision history** | Every `helm upgrade` creates a new revision — `helm rollback` restores previous |
| **Image pull errors** | `ErrImagePull` / `ImagePullBackOff` = wrong tag, wrong registry, or missing credentials |

---

### Tasks to Perform

#### 5.1 Functional Testing

```bash
# Smoke test all API endpoints
kubectl port-forward svc/catalog 8081:8080 -n retail-store &
kubectl port-forward svc/carts 8082:8080 -n retail-store &
kubectl port-forward svc/checkout 8083:8080 -n retail-store &
kubectl port-forward svc/orders 8084:8080 -n retail-store &

curl http://localhost:8081/catalogue          # should return product list
curl http://localhost:8082/carts/test-user    # should return empty cart
curl http://localhost:8083/health             # should return 200
curl http://localhost:8084/orders             # should return 200

kill %1 %2 %3 %4
```

### 5.2 HPA Autoscaling Test

```bash
# Apply HPA
kubectl apply -f catalog-hpa.yaml
kubectl get hpa -n retail-store

# Generate load
kubectl run load-generator \
  --image=busybox \
  --restart=Never \
  -n retail-store \
  -- /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue; done"

# Watch HPA scale up (takes ~1 minute to trigger)
kubectl get hpa catalog-hpa -n retail-store -w

# Stop load and watch scale down (takes ~5 minutes)
kubectl delete pod load-generator -n retail-store
kubectl get hpa catalog-hpa -n retail-store -w
```

### 5.3 Spot Interruption Resilience Test

```bash
# Check PDB is in place
kubectl get pdb -n retail-store

# Simulate node drain (mimics Spot interruption)
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl cordon $NODE
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

# Verify PDB enforced minimum availability during drain
kubectl get pods -n retail-store -w

# Restore node
kubectl uncordon $NODE
```

### 5.4 Secrets Security Test

```bash
# Verify secrets are NOT in plain Kubernetes Secrets (base64 only)
kubectl get secret -n retail-store

# Verify secrets ARE mounted from AWS Secrets Manager
kubectl exec -it deploy/catalog -n retail-store -- \
  cat /mnt/secrets/db-password
# Should print the actual password (confirm it matches Secrets Manager)

# Verify the Pod Identity is working (no access keys in pod)
kubectl exec -it deploy/catalog -n retail-store -- \
  env | grep AWS_ACCESS_KEY
# Should return NOTHING — credentials come from Pod Identity, not env vars
```

### 5.5 Observability Test

```bash
# Generate traffic to produce traces
for i in {1..20}; do
  curl -s https://retailstore.yourdomain.com/catalogue > /dev/null
done

# Check X-Ray traces (wait ~30 seconds for data)
aws xray get-trace-summaries \
  --start-time $(date -d '5 minutes ago' +%s) \
  --end-time $(date +%s) \
  --region us-east-1

# Check CloudWatch Logs
aws logs filter-log-events \
  --log-group-name /aws/containerinsights/retail-store-eks/application \
  --filter-pattern "catalog" \
  --limit 5

# Check metrics in Prometheus
kubectl port-forward svc/adot-collector-metrics 8889:8889 -n monitoring &
curl http://localhost:8889/metrics | grep http_server
```

### 5.6 Rollback Test

```bash
# Upgrade to a "bad" version
helm upgrade retail-store ./charts/retail-store \
  --namespace retail-store \
  --set ui.image.tag=broken-tag-that-doesnt-exist

# Watch pods fail
kubectl get pods -n retail-store -w

# Rollback immediately
helm rollback retail-store 1

# Verify rollback succeeded
kubectl get pods -n retail-store
helm history retail-store
```

### Expected Outcome
- Every microservice health endpoint returns HTTP 200
- HPA scales catalog pods from baseline to 8+ under sustained load, then back down after 5 minutes idle
- `kubectl drain` blocks at the PDB boundary and waits for replacement pods before continuing eviction
- `kubectl exec` into any pod confirms zero AWS access keys in environment; secrets come from `/mnt/secrets/`
- X-Ray service map renders all 5 microservices with latency data and dependency arrows
- CloudWatch Logs Insights query returns structured log lines from all running pods
- Broken Helm release rolled back to healthy previous revision in under 2 minutes

---

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| Expecting HPA to react instantly | HPA evaluates every 15s; scale-up needs 3 consecutive violations | Wait at least 1-2 minutes after load starts before concluding HPA isn't working |
| Waiting for immediate scale-down after removing load | Default scale-down cooldown is 5 minutes | This is intentional — prevents thrashing; just wait |
| Draining a node with no replacement capacity | All pods evicted but can't reschedule → outage | Ensure Karpenter has a NodePool that can provision replacement nodes |
| `helm rollback` to revision 0 | Revision 0 doesn't exist — fails | Use `helm history <release>` first to see valid revision numbers |
| Checking X-Ray immediately after deploy | No traces yet — need actual traffic first | Generate 20+ requests, then wait 30 seconds for data to appear |
| Confusing Liveness probe failure with Readiness probe failure | Liveness failure restarts the pod; Readiness just removes from LB | Check `kubectl describe pod` events to distinguish which probe is failing |
| Port-forwarding in background and forgetting to kill it | Port conflicts on next test | Always track background processes; `kill %1 %2 %3` when done |

---

### Phase 5 — Exit Criteria
- [ ] All API health endpoints return 200
- [ ] HPA scales catalog from 3 → 8+ pods under load, back to 3 at idle
- [ ] Node drain respects PDB — never drops below `minAvailable`
- [ ] Secrets come from AWS Secrets Manager, no plain-text in Git or K8s Secrets
- [ ] X-Ray trace map shows all 5 services with correct latency
- [ ] CloudWatch Logs shows container logs from all namespaces
- [ ] Helm rollback restores previous working version in < 2 minutes

---

## Phase 6 — Deployment (GitOps CI/CD Pipeline)

### Objective
Eliminate all manual deployment steps. Every code commit to `main` automatically builds a multi-platform Docker image, pushes it to AWS ECR, updates the Helm values file in Git, and triggers ArgoCD to deploy the new version to EKS — with full audit trail and automatic drift correction.

---

### Concepts to Learn

| Concept | Why It Matters |
|---|---|
| **GitHub Actions triggers** | `on: push / paths:` — only run CI when relevant files change |
| **OIDC federation** | GitHub Actions gets a temporary AWS token via trust policy — no stored secrets |
| **ECR image tagging** | `latest` is mutable (dangerous in prod); `sha-xxxx` is immutable and traceable |
| **Multi-platform Docker build** | `linux/amd64` + `linux/arm64` — required to run on both Intel and Graviton EKS nodes |
| **Git SHA as deployment identity** | Every running pod's image tag maps to an exact Git commit |
| **ArgoCD Application CRD** | Declarative definition of what to deploy, from where, to which cluster |
| **ArgoCD sync policy** | `automated` = auto-sync; `selfHeal` = revert drift; `prune` = delete removed resources |
| **Configuration drift** | Cluster state diverges from Git — happens when someone runs `kubectl apply` manually |
| **ArgoCD polling interval** | Default 3 minutes — use webhook for faster response in real production |
| **Helm chart + values separation** | Chart = structure; values files = environment config — never mix them |

---

### Tasks to Perform

#### 6.1 Create AWS ECR Repository

```bash
# Create private ECR repository
aws ecr create-repository \
  --repository-name retail-store/ui \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Get registry URL
aws ecr describe-repositories \
  --repository-names retail-store/ui \
  --query 'repositories[0].repositoryUri' \
  --output text
# Output: 123456789.dkr.ecr.us-east-1.amazonaws.com/retail-store/ui
```

### 6.2 Set Up GitHub OIDC Trust with AWS (No Stored Keys)

```bash
# Create OIDC provider for GitHub Actions (one-time per AWS account)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create IAM role trusted by GitHub Actions
# Trust policy: only your specific repo can assume this role
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name github-actions-ecr-role \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name github-actions-ecr-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

### 6.3 GitHub Actions CI Workflow

```yaml
# .github/workflows/build-push-ui.yaml
name: Build and Push UI Service to ECR

on:
  push:
    branches: [main]
    paths:
    - 'src/ui/**'           # only trigger when UI source changes

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  ECR_REPOSITORY: retail-store/ui
  HELM_VALUES_FILE: src/ui/chart/values-ui.yaml

jobs:
  build-and-push-ui:
    runs-on: ubuntu-latest
    permissions:
      id-token: write     # required for OIDC
      contents: write     # required to push Helm values update

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-ecr-role
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Define image tags
      id: tags
      run: |
        SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-8)
        echo "IMAGE_TAG=sha-${SHORT_SHA}" >> $GITHUB_ENV
        echo "tag=sha-${SHORT_SHA}" >> $GITHUB_OUTPUT

    - name: Build and push Docker images
      uses: docker/build-push-action@v5
      with:
        context: ./src/ui
        platforms: linux/amd64,linux/arm64
        push: true
        tags: |
          ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:latest
          ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}

    - name: Setup Git auth using GITHUB_TOKEN
      run: |
        git config user.email "ci-bot@github.com"
        git config user.name "ci-bot"
        git remote set-url origin \
          https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}

    - name: Update Helm values file with new image tag
      run: |
        sed -i "s|tag:.*|tag: \"${{ env.IMAGE_TAG }}\"|" ${{ env.HELM_VALUES_FILE }}
        git add ${{ env.HELM_VALUES_FILE }}
        git commit -m "ci: Update UI image tag to ${{ env.IMAGE_TAG }}"
        git push origin main

    - name: CI Complete
      run: echo "Image ${{ env.IMAGE_TAG }} pushed. ArgoCD will deploy within 3 minutes."
```

### 6.4 Install and Configure ArgoCD

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=300s

# Get initial admin password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Open: https://localhost:8080

# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password <initial-password> \
  --insecure
```

### 6.5 Create ArgoCD Application

```yaml
# argocd-retail-store-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: retail-store
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_ORG/YOUR_REPO
    targetRevision: main
    path: charts/retail-store
    helm:
      valueFiles:
      - values-aws-dataplane.yaml
      - src/ui/chart/values-ui.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: retail-store
  syncPolicy:
    automated:
      prune: true         # delete resources removed from Git
      selfHeal: true      # revert manual kubectl changes
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

```bash
kubectl apply -f argocd-retail-store-app.yaml

# Verify sync status
argocd app get retail-store
argocd app sync retail-store    # force first sync

# Watch deployment
kubectl get pods -n retail-store -w
```

### 6.6 End-to-End GitOps Verification

```bash
# Make a visible UI change
echo "<!-- v2 $(date) -->" >> src/ui/src/main/resources/templates/home.html
git add . && git commit -m "feat: add v2 marker to homepage"
git push origin main

# Watch GitHub Actions run (in GitHub UI → Actions tab)
# Expected: build-and-push-ui job completes in ~2 minutes

# Verify ECR has new image
aws ecr describe-images \
  --repository-name retail-store/ui \
  --query 'imageDetails[0].imageTags'

# Verify Git has updated Helm values
git log --oneline -3
# Should show: "ci: Update UI image tag to sha-xxxx"

# Watch ArgoCD detect and deploy (within 3 minutes)
argocd app get retail-store
# Status should change: Synced → OutOfSync → Syncing → Synced

# Verify new pods are running with new image
kubectl get pods -n retail-store
kubectl describe pod <ui-pod-name> -n retail-store | grep Image:

# Confirm change is live
curl -s https://retailstore.yourdomain.com | grep "v2 marker"
```

### Expected Outcome
- ECR repository exists, `scanOnPush=true`, no Docker Hub involved anywhere
- GitHub Actions workflow runs in ~2 minutes end-to-end: checkout → OIDC → ECR push → Git commit
- ECR shows exactly two tags per build: `latest` and `sha-xxxxxxxx`
- `ci-bot` commit appears in Git log updating `values-ui.yaml` with new image tag
- ArgoCD detects the Git change and transitions `OutOfSync → Syncing → Synced` within 3 minutes
- Running pod's image tag matches the Git SHA of the triggering commit — full traceability
- Change is visible on live HTTPS URL within 5 minutes of `git push`
- Manual `kubectl set image` is automatically reverted by ArgoCD within 3 minutes (drift correction)

---

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| Storing AWS access keys in GitHub Secrets | If repo is compromised, attacker gets AWS access | Use OIDC federation — GitHub Actions gets temporary credentials via trust policy |
| Using `latest` as the only image tag | `latest` is mutable — ArgoCD can't detect a change if tag doesn't change | Always use `sha-xxxx` as the image tag in `values-ui.yaml` |
| `ci-bot` commit triggers another CI run (infinite loop) | Build queue fills up | Add `paths:` filter — CI only runs on `src/ui/**` changes, not on `chart/**` changes |
| ArgoCD stuck in `OutOfSync` forever | Chart structure issue or Helm render error | Run `argocd app diff retail-store` to see exactly what's different |
| Building only `linux/amd64` image | Pod fails to schedule on ARM/Graviton nodes | Always use `--platform linux/amd64,linux/arm64` in `docker buildx build` |
| `git push` in CI fails with 403 | `GITHUB_TOKEN` lacks write permission | Add `permissions: contents: write` to the job in the workflow file |
| ArgoCD `selfHeal` not enabled | Manual `kubectl apply` creates permanent drift | Set `syncPolicy.automated.selfHeal: true` in Application CRD |
| Not setting `prune: true` | Deleted resources in Git remain in cluster as orphans | Set `syncPolicy.automated.prune: true` to clean up removed resources |

---

### Phase 6 — Exit Criteria
- [ ] ECR repository exists with `scanOnPush=true`
- [ ] GitHub OIDC trust established — no AWS access keys stored in GitHub
- [ ] GitHub Actions workflow completes successfully in < 2 minutes
- [ ] ECR shows two tags: `latest` and `sha-xxxxxxxx`
- [ ] Git shows automated commit from `ci-bot` updating `values-ui.yaml`
- [ ] ArgoCD shows `Synced` status after detecting the Git change
- [ ] New image tag visible in running pod's `Image:` field
- [ ] Code change is visible in live application at public URL
- [ ] ArgoCD `selfHeal` reverts a manual `kubectl set image` within 3 minutes

---

## Phase Summary

| Phase | Duration | Key Output |
|---|---|---|
| **1 — Environment Setup** | Day 1–2 | All tools installed, app running locally |
| **2 — Core Services** | Day 3–5 | EKS cluster + all add-ons running |
| **3 — Database Integration** | Day 6–8 | All AWS data services wired to microservices |
| **4 — APIs and Communication** | Day 9–11 | Full app live at public HTTPS URL |
| **5 — Testing** | Day 12–14 | HPA, PDB, secrets, traces all verified |
| **6 — Deployment** | Day 15–17 | Full GitOps: code push → live in < 5 minutes |

---

# Hands-On Exercises

> Each exercise has a **Task**, **What you will learn**, **Success criteria**, and **Hint** if you get stuck.
> Do NOT read the hint until you have tried for at least 20 minutes.

---

## Level 1 — Beginner Exercises

---

### Exercise B-01 — Run a Single Container and Inspect It

**Task**
```bash
docker pull nginx:alpine
docker run -d --name my-nginx -p 8080:80 nginx:alpine
```
1. Open `http://localhost:8080` in your browser — confirm you see the Nginx welcome page
2. Get the container's internal IP address using only `docker` commands
3. View the access logs in real time while refreshing the browser
4. Copy the default `index.html` out of the container to your local machine
5. Stop and remove the container and image completely

**What You Will Learn**
- `docker inspect`, `docker logs -f`, `docker cp`, `docker stop`, `docker rm`, `docker rmi`
- The difference between a running container and a stopped container
- How port mapping works: `-p host:container`

**Success Criteria**
- [ ] Browser shows Nginx page
- [ ] You can name the container's IP address from `docker inspect`
- [ ] `docker logs -f my-nginx` shows your browser requests in real time
- [ ] `index.html` exists on your local machine after `docker cp`
- [ ] `docker ps -a` shows no containers; `docker images` shows no nginx

**Hint**
`docker inspect my-nginx | grep IPAddress` and `docker cp my-nginx:/usr/share/nginx/html/index.html .`

---

### Exercise B-02 — Write a Dockerfile for a Simple App

**Task**
Create a Python web server that returns `"Hello from DevOps!"` on port 5000.

1. Create a file `app.py`:
   ```python
   from http.server import HTTPServer, BaseHTTPRequestHandler
   class Handler(BaseHTTPRequestHandler):
       def do_GET(self):
           self.send_response(200)
           self.end_headers()
           self.wfile.write(b"Hello from DevOps!")
   HTTPServer(("", 5000), Handler).serve_forever()
   ```
2. Write a `Dockerfile` using `python:3.11-slim` as the base image
3. Build the image with tag `devops-hello:v1`
4. Run it on port `8080` and verify with `curl http://localhost:8080`
5. Rebuild it with `python:3.11-alpine` — compare the image sizes

**What You Will Learn**
- `FROM`, `WORKDIR`, `COPY`, `RUN`, `EXPOSE`, `CMD` instructions
- Why Alpine images are smaller than Slim images
- `docker build -t`, `docker image ls` to compare sizes

**Success Criteria**
- [ ] `curl http://localhost:8080` returns `Hello from DevOps!`
- [ ] `docker image ls` shows `devops-hello:v1`
- [ ] Alpine version is at least 30MB smaller than Slim version

**Hint**
Your Dockerfile needs at minimum: `FROM`, `WORKDIR /app`, `COPY app.py .`, `CMD ["python", "app.py"]`

---

### Exercise B-03 — Docker Compose with Dependency Ordering

**Task**
Write a `docker-compose.yaml` that runs:
- A PostgreSQL database (`postgres:15-alpine`) with a named volume
- A simple web app that waits for the DB to be healthy before starting
- Use `healthcheck` on the DB container
- Use `depends_on: condition: service_healthy` on the web container

1. Start the stack — confirm DB starts before the app
2. Stop just the app container without stopping the DB
3. Bring everything down including the volume
4. Bring it back up — confirm data in the volume is gone (ephemeral volume test)

**What You Will Learn**
- `healthcheck`, `depends_on` with `condition: service_healthy`
- Named volumes vs anonymous volumes
- `docker compose stop <service>` vs `docker compose down -v`

**Success Criteria**
- [ ] DB container reaches `healthy` before app container starts
- [ ] `docker compose stop web` leaves DB running
- [ ] `docker compose down -v` removes the volume
- [ ] After restart, DB is empty (volume was removed)

**Hint**
PostgreSQL healthcheck: `test: ["CMD-SHELL", "pg_isready -U postgres"]`

---

### Exercise B-04 — Terraform State Basics

**Task**
1. Write a minimal Terraform file that creates an S3 bucket with versioning enabled
2. Run `terraform plan` and read every line of the output — identify what will be created
3. Run `terraform apply` — note the state file created locally
4. Open `terraform.tfstate` in a text editor — find where the bucket name is stored
5. Run `terraform apply` again without any changes — observe that the plan shows `0 changes`
6. Add a tag `{"Environment": "dev"}` to the bucket — run `plan` and `apply` again
7. Run `terraform destroy`

**What You Will Learn**
- How Terraform detects drift (real vs state vs config)
- What the state file actually contains
- Why `terraform plan` must be read carefully before `apply`
- The `+`, `~`, `-` symbols in plan output (create, update, destroy)

**Success Criteria**
- [ ] Bucket exists in S3 after `apply`
- [ ] Second `apply` with no config changes shows `No changes`
- [ ] Adding a tag shows `~ update in-place` (not destroy/recreate)
- [ ] Bucket is gone after `destroy`

**Hint**
`resource "aws_s3_bucket_versioning"` is a separate resource from `aws_s3_bucket` in AWS provider v4+

---

### Exercise B-05 — kubectl Basics on a Running Cluster

**Task**
Using the EKS cluster from Phase 2:
1. List all pods across all namespaces — how many are in `kube-system`?
2. Describe the `coredns` pod — what node is it running on?
3. Get the logs of the `aws-load-balancer-controller` pod
4. Run a temporary `busybox` pod interactively and run `nslookup kubernetes` from inside it
5. Delete the temporary pod
6. Get all resources (pods, services, deployments, configmaps) in `kube-system` in one command

**What You Will Learn**
- `kubectl get pods -A`, `kubectl describe`, `kubectl logs`
- `kubectl run --rm -it` for temporary debug pods
- In-cluster DNS: `<service>.<namespace>.svc.cluster.local`
- `kubectl get all -n <namespace>`

**Success Criteria**
- [ ] You can name how many pods are in `kube-system`
- [ ] You know which node `coredns` runs on
- [ ] You see LBC startup logs
- [ ] `nslookup kubernetes` returns an IP from inside the busybox pod
- [ ] Busybox pod is gone after exit

**Hint**
`kubectl run tmp --image=busybox --rm -it --restart=Never -- sh`

---

## Level 2 — Intermediate Exercises

---

### Exercise I-01 — Deploy, Break, and Fix a Microservice

**Task**
1. Deploy the catalog microservice with a **wrong DB hostname** in its ConfigMap
2. Observe the pod behaviour — what state does it enter?
3. Read the pod logs to identify the exact error message
4. Fix the ConfigMap with the correct hostname and redeploy
5. Verify the pod becomes `Running` and the `/catalogue` endpoint returns data

**What You Will Learn**
- `CrashLoopBackOff` vs `Error` vs `Pending` pod states
- `kubectl describe pod` Events section for root cause
- `kubectl logs --previous` to read logs from a crashed container
- `kubectl rollout restart deployment` to force a redeploy after ConfigMap fix

**Success Criteria**
- [ ] You can identify `CrashLoopBackOff` from `kubectl get pods`
- [ ] `kubectl describe pod` shows the DB connection error in Events
- [ ] After ConfigMap fix, pod transitions to `Running` within 60 seconds
- [ ] `curl http://localhost:8080/catalogue` returns JSON product list

**Hint**
After editing a ConfigMap, pods do NOT automatically restart. Use:
`kubectl rollout restart deployment/catalog -n retail-store`

---

### Exercise I-02 — Write a Terraform Module from Scratch

**Task**
Create a Terraform module called `modules/s3-static-site` that:
1. Accepts inputs: `bucket_name`, `environment`, `tags`
2. Creates an S3 bucket with versioning and encryption enabled
3. Outputs: `bucket_name`, `bucket_arn`, `bucket_domain_name`
4. Call the module from a root module with two instances: `dev` and `staging`
5. Both should create different buckets from the same module code

**What You Will Learn**
- Module directory structure: `variables.tf`, `main.tf`, `outputs.tf`
- `module` block in root module, passing input variables
- How to use `module.<name>.<output>` in root outputs
- Code reuse: two environments from one module

**Success Criteria**
- [ ] `terraform plan` shows 2 S3 buckets to be created
- [ ] Both buckets created after `apply`
- [ ] Changing `tags` in one instance doesn't affect the other
- [ ] `terraform output` shows both bucket ARNs

**Hint**
Module path: `source = "./modules/s3-static-site"`. Each module instance needs a unique `bucket_name`.

---

### Exercise I-03 — HPA in Action with Custom Thresholds

**Task**
1. Deploy the catalog microservice with `resources.requests.cpu: 100m`
2. Create an HPA: `minReplicas: 2`, `maxReplicas: 8`, `targetCPUUtilization: 40%`
3. Confirm baseline is 2 replicas
4. Generate CPU load:
   ```bash
   kubectl run load --image=busybox -n retail-store --restart=Never -- \
     /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue; done"
   ```
5. Watch HPA every 15 seconds — record exact replica count at T+1min, T+2min, T+3min
6. Kill the load generator — record how long scale-down takes
7. Change `targetCPUUtilization` to `70%` and repeat — observe the difference

**What You Will Learn**
- How CPU target percentage is calculated against `requests.cpu`
- Why `resources.requests.cpu` is mandatory for HPA
- The 3-minute scale-up vs 5-minute scale-down asymmetry
- How changing the target threshold affects sensitivity

**Success Criteria**
- [ ] HPA stays at 2 replicas with no load
- [ ] HPA reaches at least 5 replicas under sustained load
- [ ] Scale-down takes approximately 5 minutes after load stops
- [ ] With 70% target, scale-up is less aggressive than with 40% target

**Hint**
`kubectl get hpa catalog-hpa -n retail-store -w` — the `-w` flag watches for changes every few seconds.

---

### Exercise I-04 — Secrets Manager + Pod Identity End-to-End

**Task**
1. Create a secret in AWS Secrets Manager:
   ```bash
   aws secretsmanager create-secret \
     --name "exercise/my-app/config" \
     --secret-string '{"api_key":"super-secret-key-123","db_pass":"mypassword"}'
   ```
2. Create a Kubernetes ServiceAccount, an IAM Role, and a Pod Identity Association
3. Write a `SecretProviderClass` that fetches `api_key` and `db_pass` separately
4. Deploy a pod that mounts the secret as both a file and environment variables
5. `kubectl exec` into the pod and confirm:
   - `/mnt/secrets/api_key` contains `super-secret-key-123`
   - `echo $API_KEY` prints `super-secret-key-123`
6. Update the secret value in AWS Secrets Manager — observe how long it takes to refresh in the pod

**What You Will Learn**
- Full Pod Identity trust chain: ServiceAccount → IAM Role → Secrets Manager
- `jmesPath` in SecretProviderClass for extracting individual keys
- `secretObjects` for syncing to Kubernetes Secrets (env vars)
- Secret rotation behaviour — CSI driver caches; pod needs restart to pick up new value

**Success Criteria**
- [ ] Pod starts without errors and mounts the secret
- [ ] File and env var both contain the correct secret value
- [ ] No `AWS_ACCESS_KEY_ID` in pod environment
- [ ] After secret update in Secrets Manager, pod shows old value until restarted

**Hint**
The `SecretProviderClass` needs `provider: aws` and the IAM role needs `secretsmanager:GetSecretValue` permission on the specific secret ARN.

---

### Exercise I-05 — Helm Chart Customisation

**Task**
Take the retail store Helm chart and:
1. Add a new value `catalog.replicaCount` that defaults to `2`
2. Wire it to the catalog Deployment's `replicas` field using `{{ .Values.catalog.replicaCount }}`
3. Override it at install time to `5` using `--set`
4. Create a `values-prod.yaml` that sets it to `10` and apply using `-f`
5. Add a conditional block: if `catalog.hpa.enabled` is `true`, render the HPA resource; otherwise skip it
6. Test both `enabled: true` and `enabled: false` with `helm template` (dry run)

**What You Will Learn**
- Helm template syntax: `{{ .Values.x }}`, `{{ if }}`, `{{ end }}`
- `helm template` for rendering manifests without deploying
- `--set` vs `-f values.yaml` precedence
- Conditional resource rendering

**Success Criteria**
- [ ] `helm install --set catalog.replicaCount=5` deploys 5 catalog pods
- [ ] `helm template -f values-prod.yaml` shows 10 replicas in the rendered YAML
- [ ] `helm template` with `hpa.enabled: false` produces no HPA resource
- [ ] `helm template` with `hpa.enabled: true` produces a valid HPA resource

**Hint**
`helm template retail-store ./charts/retail-store --set catalog.replicaCount=5 | grep replicas`

---

### Exercise I-06 — Observe a Slow Endpoint with X-Ray

**Task**
1. Artificially slow down the catalog service by adding a sleep in the response handler (or use a throttle proxy)
2. Generate 50 requests to `https://retailstore.yourdomain.com/catalogue`
3. Open the X-Ray Service Map in AWS console
4. Identify which node in the trace map has the highest average latency
5. Click into a trace — find the exact span where time is being spent
6. Remove the artificial slowdown — confirm latency returns to baseline on the trace map

**What You Will Learn**
- How to navigate the X-Ray console: Service Map vs Trace List vs Trace Detail
- Reading a trace waterfall: parent span vs child spans
- How to identify the root cause of latency (app code vs DB vs network)
- The difference between p50, p95, p99 latency percentiles

**Success Criteria**
- [ ] X-Ray service map shows elevated latency on the catalog node
- [ ] You can identify the specific span (HTTP handler vs DB query) that is slow
- [ ] After fix, p95 latency drops back to baseline within 2 minutes

**Hint**
AWS Console → X-Ray → Service Map. Click a node to see its latency histogram. Click "View traces" to drill into individual requests.

---

## Level 3 — Advanced Real-World Exercises

---

### Exercise A-01 — Implement Zero-Downtime Blue/Green Deployment

**Task**
Implement a blue/green deployment for the UI service without ArgoCD automation:
1. Current deployment is `ui-blue` (label: `version: blue`) — receiving 100% traffic
2. Deploy `ui-green` (label: `version: green`) with a new image tag
3. Wait for all `ui-green` pods to pass readiness probes
4. Switch the Service selector from `version: blue` to `version: green` — observe zero dropped requests
5. Run `curl` in a loop during the switchover and count HTTP errors (should be 0)
6. If green has issues, switch back to blue instantly
7. After validation, scale blue to 0 replicas (keep it for quick rollback)

**What You Will Learn**
- Blue/green vs rolling update trade-offs
- How Kubernetes Service `selector` controls traffic routing
- Why blue/green needs 2x resources but gives instant rollback
- Measuring zero-downtime with `curl` loop + HTTP error counting

**Success Criteria**
- [ ] Zero HTTP errors during the selector switch (verify with curl loop)
- [ ] Green pods are fully ready before any traffic is switched
- [ ] Switching back to blue takes under 10 seconds
- [ ] Blue replicas can be scaled to 0 while green handles all traffic

**Hint**
```bash
# Run this loop during the switchover to count errors
for i in {1..1000}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://retailstore.yourdomain.com)
  [[ "$STATUS" != "200" ]] && echo "ERROR: $STATUS at request $i"
done
```

---

### Exercise A-02 — Build a Self-Healing Alerting System

**Task**
Build an end-to-end alerting pipeline:
1. Create a Prometheus alerting rule that fires when `catalog` pod count < 2 for > 1 minute
2. Configure Alertmanager to route the alert to a webhook (use `https://webhook.site` as a test receiver)
3. Manually scale catalog to 1 replica — confirm alert fires and webhook receives the notification
4. Scale back to 3 replicas — confirm alert resolves and a resolution notification is sent
5. Extend the rule to also alert when p95 latency > 500ms for any microservice

**What You Will Learn**
- Prometheus alerting rule syntax: `expr`, `for`, `labels`, `annotations`
- Alertmanager routing: `route`, `receiver`, `group_by`, `repeat_interval`
- Alert states: `Pending` → `Firing` → `Resolved`
- PromQL for latency: `histogram_quantile(0.95, rate(...))`

**Success Criteria**
- [ ] Alert transitions to `Pending` within 30 seconds of scaling down
- [ ] Alert transitions to `Firing` after 1 minute
- [ ] Webhook.site receives the firing notification with correct labels
- [ ] Alert resolves and webhook receives resolution notification after scaling back up

**Hint**
```yaml
# Prometheus alerting rule
groups:
- name: retail-store
  rules:
  - alert: CatalogPodsLow
    expr: kube_deployment_status_replicas_ready{deployment="catalog"} < 2
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Catalog has fewer than 2 ready replicas"
```

---

### Exercise A-03 — Implement Spot Interruption Handling from Scratch

**Task**
Set up the complete Spot interruption handling pipeline without relying on pre-built Terraform:
1. Create a Karpenter `NodePool` that uses only Spot instances (`capacity-type: spot`)
2. Create an EventBridge rule that captures `EC2 Spot Instance Interruption Warning` events
3. Route the EventBridge rule to an SQS queue
4. Configure Karpenter to watch the SQS queue (via `interruptionQueue` setting)
5. Create a PDB for the catalog service with `minAvailable: 2`
6. Force a Spot interruption simulation:
   ```bash
   # Use AWS FIS (Fault Injection Simulator) or manually terminate a Spot node
   aws ec2 terminate-instances --instance-ids <spot-node-instance-id>
   ```
7. Verify that:
   - Karpenter provisions a replacement node
   - Catalog pods migrate without breaching the PDB
   - Zero downtime (curl loop shows no errors)

**What You Will Learn**
- Full Karpenter interruption handling architecture: EventBridge → SQS → Karpenter
- The timing window: 2-minute warning before actual termination
- PDB enforcement during pod eviction
- How to use AWS FIS for chaos engineering

**Success Criteria**
- [ ] Spot node terminates — Karpenter provisions a replacement within 2 minutes
- [ ] PDB is never violated: at least 2 catalog pods running at all times
- [ ] Zero HTTP 5xx errors during the node termination event
- [ ] New node is also Spot (Karpenter re-uses the NodePool constraints)

**Hint**
Check Karpenter logs during interruption: `kubectl logs -n karpenter deploy/karpenter -f | grep -i interrupt`

---

### Exercise A-04 — Add a New Microservice to the GitOps Pipeline

**Task**
Add a completely new `recommendations` microservice to the system end-to-end:
1. Write a minimal Node.js Express app that returns `["product-1", "product-2", "product-3"]` on `GET /recommendations`
2. Write a multi-stage Dockerfile (build stage + runtime stage)
3. Add the microservice to the existing Helm chart as a new template
4. Add a GitHub Actions workflow that builds and pushes it to a new ECR repository
5. Update the `values.yaml` with a `recommendations` section
6. Update `ArgoCD Application` to include the new service
7. Verify the new service is deployed and accessible at `/recommendations` via the existing Ingress

**What You Will Learn**
- Full GitOps lifecycle for a brand new service
- Adding a service to an existing Helm chart
- Multi-stage Dockerfile for Node.js (build deps vs runtime)
- How ArgoCD handles new resources added to a chart

**Success Criteria**
- [ ] New ECR repository exists: `retail-store/recommendations`
- [ ] Docker image built for `linux/amd64` and `linux/arm64`
- [ ] `helm template` shows all existing resources PLUS the new recommendations resources
- [ ] ArgoCD syncs and deploys recommendations pod automatically
- [ ] `curl https://retailstore.yourdomain.com/recommendations` returns product list

**Hint**
Copy an existing service template (e.g., catalog) and modify names, ports, and image reference. Keep the Node.js app minimal — the focus is the pipeline, not the app code.

---

### Exercise A-05 — Implement Multi-Environment GitOps with Separate Namespaces

**Task**
Extend the GitOps setup to support `dev` and `prod` environments from the same Git repo:

```
charts/
└── retail-store/
    ├── templates/
    ├── values.yaml           ← shared defaults
    ├── values-dev.yaml       ← dev overrides (1 replica, small instances)
    └── values-prod.yaml      ← prod overrides (3 replicas, HPA enabled, RDS)
```

1. Create two ArgoCD Applications: `retail-store-dev` and `retail-store-prod`
2. `retail-store-dev` deploys to namespace `retail-store-dev` using `values-dev.yaml`
3. `retail-store-prod` deploys to namespace `retail-store-prod` using `values-prod.yaml`
4. Configure `retail-store-dev` to auto-sync on every push to `main`
5. Configure `retail-store-prod` to require **manual sync approval** (disable `automated`)
6. Test: push a change — confirm dev auto-deploys, prod stays at old version until manually synced
7. Manually sync prod — confirm it deploys the same image that is already in dev

**What You Will Learn**
- Multi-environment GitOps patterns with a single chart
- ArgoCD Application per environment with different values files
- Promotion workflow: auto-deploy to dev, manual approval for prod
- Namespace isolation for different environments in the same cluster

**Success Criteria**
- [ ] `kubectl get pods -n retail-store-dev` and `-n retail-store-prod` both show running pods
- [ ] Dev has 1 replica per service; prod has 3
- [ ] Code push auto-deploys to dev within 3 minutes
- [ ] Prod remains at old version until you click "Sync" in ArgoCD UI
- [ ] After manual prod sync, both environments run the same image tag

**Hint**
Disable automated sync for prod:
```yaml
syncPolicy:
  automated: null    # no auto-sync
```
And enable it for dev:
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

---

## Exercise Quick Reference

| Exercise | Level | Topic | Time Estimate |
|---|---|---|---|
| B-01 | Beginner | Docker run + inspect | 30 min |
| B-02 | Beginner | Write a Dockerfile | 45 min |
| B-03 | Beginner | Docker Compose healthchecks | 45 min |
| B-04 | Beginner | Terraform state basics | 60 min |
| B-05 | Beginner | kubectl fundamentals | 45 min |
| I-01 | Intermediate | Debug CrashLoopBackOff | 60 min |
| I-02 | Intermediate | Write a Terraform module | 90 min |
| I-03 | Intermediate | HPA load testing | 90 min |
| I-04 | Intermediate | Secrets Manager + Pod Identity | 120 min |
| I-05 | Intermediate | Helm chart customisation | 90 min |
| I-06 | Intermediate | X-Ray trace analysis | 60 min |
| A-01 | Advanced | Blue/green zero-downtime deploy | 120 min |
| A-02 | Advanced | Prometheus alerting pipeline | 180 min |
| A-03 | Advanced | Spot interruption from scratch | 180 min |
| A-04 | Advanced | Add new service to GitOps | 240 min |
| A-05 | Advanced | Multi-environment GitOps | 240 min |

---

# Real-World Analogies

> Every concept in this architecture has a direct parallel in the physical world.
> If you understand the analogy, you understand the technology.

---

## The Whole System — A Modern Shopping Mall

The entire Retail Store platform is like a **modern shopping mall**:

- The **VPC** is the mall building — everything inside is protected from the outside world
- The **public subnets** are the entrance lobby — customers can enter here
- The **private subnets** are the back offices and stockrooms — customers never go there
- The **ALB** is the information desk at the entrance — it directs you to the right shop
- The **microservices** are the individual shops (Catalog = clothing store, Cart = shopping basket, Checkout = cashier)
- **AWS managed databases** are the mall's shared cold storage — shops don't each run their own refrigerators
- **Karpenter** is the mall management team that opens and closes parking spaces based on how busy the mall is

---

## Analogy 1 — Docker: Shipping Containers

Before Docker, deploying software was like shipping goods in **loose, unlabelled boxes**. Every delivery truck, warehouse, and destination handled things differently. Things got lost, broken, or mixed up.

Docker is the **standardised shipping container** (like the ones on cargo ships):

| Docker Concept | Shipping Analogy |
|---|---|
| **Docker Image** | The container blueprint / packing list |
| **Docker Container** | A filled, sealed shipping container |
| **Dockerfile** | The instructions for packing the container |
| **Docker Hub / ECR** | The shipping port / container depot |
| **docker run** | Loading the container onto a truck and driving it |
| **Port mapping `-p 8080:80`** | The dock number where you unload the container |
| **Multi-stage build** | Pack only finished goods, not the factory equipment |

> **Key insight:** The same container runs identically on your laptop, in CI, and in production. No more "it works on my machine."

---

## Analogy 2 — Kubernetes: A Smart Airport

Running containers manually with Docker is like having one small airfield — fine for a hobby. Kubernetes is a **major international airport** with thousands of flights.

| Kubernetes Concept | Airport Analogy |
|---|---|
| **Kubernetes Cluster** | The entire airport complex |
| **Control Plane** | Air Traffic Control tower — sees everything, gives instructions |
| **Worker Nodes** | Runways — where planes actually land and take off |
| **Pod** | A plane — the actual unit doing the work |
| **Deployment** | A scheduled airline route — maintains N planes on that route at all times |
| **Service (ClusterIP)** | The gate number — always the same, even if the planes change |
| **Ingress** | The airport terminal — single entry point that routes you to the right gate |
| **Namespace** | Separate terminals (T1 = dev, T2 = prod) — same airport, isolated zones |
| **kube-scheduler** | The controller who assigns which runway each plane uses |
| **kubelet** | The ground crew on each runway — ensures planes actually take off and land |

> **Key insight:** When a plane (pod) crashes, ATC (Kubernetes) immediately schedules a replacement. You don't manage individual planes — you manage flight routes (Deployments).

---

## Analogy 3 — Terraform: An Architect's Blueprint

Building AWS infrastructure manually in the console is like a **contractor building a house by memory** — works once, impossible to reproduce exactly, impossible to inspect, and disastrous when something goes wrong.

Terraform is the **architect's blueprint**:

| Terraform Concept | Construction Analogy |
|---|---|
| **`.tf` files** | Architectural blueprints — the desired design on paper |
| **`terraform apply`** | Construction crew building exactly what's on the blueprints |
| **`terraform.tfstate`** | The as-built record — documents what was actually built |
| **`terraform plan`** | The foreman's review: "Here's what will change before we start" |
| **Remote state (S3)** | Blueprints stored in a fireproof vault — shared, versioned, backed up |
| **DynamoDB state lock** | Only one contractor can make changes at a time — no two builders modifying the same wall |
| **Terraform modules** | Standard room designs — a "kitchen module" used in every house |
| **`terraform destroy`** | Demolishing the building according to the blueprint |

> **Key insight:** If your data centre burns down, you run `terraform apply` and the entire infrastructure is rebuilt identically from the blueprints. That's the power of IaC.

---

## Analogy 4 — VPC Networking: A Secure Office Building

A VPC is a **corporate office building**:

```
Office Building (VPC: 10.0.0.0/16)
│
├── Public Lobby (Public Subnets)
│   ├── Reception desk (ALB) — visitors come here
│   ├── Security checkpoint (Security Groups) — ID check before entry
│   └── Building entrance (Internet Gateway) — connected to the street
│
└── Private Office Floors (Private Subnets)
    ├── Employee workstations (EKS Worker Nodes)
    ├── Server room (Databases)
    └── Back door to courier (NAT Gateway) — employees can send packages out,
                                              but couriers can't walk in uninvited
```

| Network Concept | Office Analogy |
|---|---|
| **Internet Gateway** | The main building entrance facing the street |
| **NAT Gateway** | The building's mailroom — you send mail out, but no one can walk in through it |
| **Public Subnet** | The lobby floor — anyone can enter with a badge check |
| **Private Subnet** | Restricted floors — only employees with access cards |
| **Security Group** | The access card system — defines who can enter which room |
| **Route Table** | The building directory — tells you which elevator goes to which floor |

> **Key insight:** Hackers cannot reach your databases directly because there is no route from the internet to the private subnets. The only way in is through the lobby (ALB).

---

## Analogy 5 — Microservices: A Restaurant Kitchen

A monolithic application is like a **single chef** who takes your order, cooks everything, does the dishes, and handles billing. Fast when the restaurant is empty, catastrophic when busy.

Microservices are a **professional restaurant kitchen** with specialists:

| Microservice | Kitchen Role |
|---|---|
| **UI Service** | The waiter — takes the customer's full order, coordinates with all departments |
| **Catalog Service** | The menu board team — knows every dish, ingredient, and price |
| **Cart Service** | The order ticket — tracks what each table has ordered so far |
| **Checkout Service** | The cashier — finalises the bill and processes payment |
| **Orders Service** | The kitchen expeditor — confirms order, tracks it, notifies when ready |
| **RDS MySQL** | The recipe book stored in a vault — persistent, authoritative data |
| **Redis Cache** | The chef's counter — frequently used ingredients kept close at hand |
| **SQS Queue** | The kitchen printer — orders queue up and are processed one by one |

> **Key insight:** If the dessert chef (Checkout) is sick, the main course chef (Catalog) keeps working. You scale only the cashier station during a rush — not the entire restaurant.

---

## Analogy 6 — Helm: IKEA Furniture

Deploying Kubernetes apps by writing raw YAML for every environment is like **building custom furniture from scratch** each time. Helm charts are **IKEA furniture**:

| Helm Concept | IKEA Analogy |
|---|---|
| **Helm Chart** | The IKEA furniture box — all parts, one package |
| **`templates/`** | The generic assembly instructions with variable hole sizes |
| **`values.yaml`** | The specific measurements for YOUR room |
| **`helm install`** | Opening the box and assembling the furniture |
| **`helm upgrade`** | Replacing a drawer with a bigger one without rebuilding the whole wardrobe |
| **`helm rollback`** | Reverting to the previous drawer configuration |
| **`--set replicas=5`** | Asking IKEA to ship extra shelf boards for your specific needs |
| **Helm Repository** | The IKEA catalogue — browse and install pre-built furniture |

> **Key insight:** The same Helm chart deploys to dev (1 replica, small) and prod (3 replicas, HPA enabled, RDS) by swapping one values file. One blueprint, infinite configurations.

---

## Analogy 7 — HPA + Karpenter: A Smart Ride-Share Company

Manual scaling is like a taxi company that decides how many cabs to run every Monday morning and never changes it. Autoscaling is **Uber's dynamic dispatch**:

### HPA = Surge Pricing Dispatch (Pod Level)
> When demand spikes in an area, Uber asks more nearby drivers to head there.

- HPA watches CPU/memory (demand signals)
- When utilisation exceeds the target, it increases pod replicas (dispatches more drivers)
- When demand drops, it reduces replicas after a cooldown (drivers go back to standby)

### Karpenter = Fleet Size Management (Node Level)
> When Uber needs more drivers than are available, it approves new driver sign-ups instantly.

- Karpenter watches for `Pending` pods (unmet demand)
- Provisions new EC2 nodes instantly (brings new drivers online)
- Removes nodes when they're idle (lets drivers go home)

### PDB = Minimum Service Guarantee
> Uber guarantees at least 3 cars are always available in an area, even during driver shift changes.

- PDB sets `minAvailable: 3`
- Kubernetes refuses to evict pods that would drop below this minimum
- Ensures the service never fully drops during node recycling

### Spot Instances = Part-Time Drivers
> Cheaper than full-time drivers, but they can leave with 2 minutes notice.

- Spot instances cost 70-90% less
- AWS can reclaim them with a 2-minute Spot interruption warning
- Karpenter handles the warning gracefully — migrates work before the driver leaves

---

## Analogy 8 — AWS Secrets Manager: A Bank Vault for Keys

Storing passwords in environment variables or code is like **writing your house keys under the doormat** — convenient but catastrophic if found.

AWS Secrets Manager is a **bank vault with access logs**:

| Secrets Manager Concept | Bank Vault Analogy |
|---|---|
| **Secret** | A lockbox in the vault containing your valuables |
| **Secret rotation** | The bank automatically changes your vault combination on a schedule |
| **Access policy (IAM)** | Only specific people (IAM roles) can open specific lockboxes |
| **Audit trail (CloudTrail)** | Every vault access is logged: who opened it, when, from which IP |
| **Secrets Store CSI Driver** | The bank courier who delivers the contents to your desk each morning |
| **Pod Identity** | Your employee ID badge — proves who you are before the courier hands over the contents |

> **Key insight:** If an employee (pod) leaves (is deleted), their access badge (Pod Identity) is revoked. The secret in the vault is never touched — only access is removed.

---

## Analogy 9 — GitOps + ArgoCD: A Central Bank's Ledger System

Manual `kubectl apply` deployments are like a bank where **tellers can change account balances by hand** with no record. One bad day and the books don't balance.

GitOps is a **double-entry bookkeeping system**:

| GitOps Concept | Banking Analogy |
|---|---|
| **Git repository** | The official ledger — the authoritative record of all transactions |
| **ArgoCD** | The auditor who continuously checks the bank's actual holdings against the ledger |
| **`git commit`** | Filing a new transaction in the ledger |
| **ArgoCD sync** | The auditor reconciles the bank's holdings to match the ledger |
| **Drift (`OutOfSync`)** | Someone changed an account balance without a ledger entry — flagged immediately |
| **`selfHeal: true`** | The auditor automatically reverses unauthorised changes |
| **Git history** | The complete audit trail — who made every transaction, when, and why |
| **`git revert`** | Issuing a reversal transaction — the cleanest way to undo an error |

> **Key insight:** In GitOps, the only way to change the cluster is through a Git commit. `kubectl apply` directly to production is like a bank teller changing a balance by hand — ArgoCD will detect and reverse it.

---

## Analogy 10 — Observability: A Hospital Monitoring System

Running a production system without observability is like a hospital with **no patient monitors** — you only know something is wrong when the patient stops breathing.

The three pillars of observability are like three different hospital monitoring systems:

### Traces = Patient Journey Records
> Every patient (request) gets a wristband with a unique ID. Every department (microservice) scans it and adds notes. The full record shows where time was spent.

- A trace is the complete record of one request's journey through all microservices
- Each span is one department's entry: "patient arrived at radiology at 2:05pm, left at 2:12pm"
- X-Ray is the records management system — you can pull up any patient's full journey

### Logs = Nurses' Station Notes
> Nurses write detailed notes about everything: patient complained of pain at 3pm, medication given at 3:05pm, temperature checked at 4pm.

- Logs are the raw timestamped notes from each service
- CloudWatch Logs is the centralised filing system for all wards
- CloudWatch Insights is the search system: "find all patients who complained of pain in the last hour"

### Metrics = Vital Signs Monitors
> The heart rate monitor shows a number every second. You set an alarm threshold: "alert if heart rate > 120 for > 5 minutes."

- Metrics are numerical measurements over time (CPU%, request rate, error rate)
- Prometheus stores the time series data
- Grafana is the dashboard wall of monitors
- Alertmanager is the alarm system that pages the on-call doctor (engineer)

> **Key insight:** Logs tell you WHAT happened. Traces tell you WHERE it happened across services. Metrics tell you the PATTERN — is this getting better or worse over time?

---

## Analogy 11 — CI/CD GitOps Pipeline: A Car Factory Assembly Line

Manual deployments are like **artisanal car building** — one craftsman does everything. Quality varies, speed is slow, and no two cars are identical.

The GitOps CI/CD pipeline is a **modern automated assembly line**:

```
Developer writes code
    │
    │ (Pulls the trigger — drops raw material onto the conveyor)
    ▼
GitHub Actions CI  =  Quality Control Station
    │  Inspects the material (builds image)
    │  Stamps with serial number (git SHA tag)
    │  Stores in warehouse (pushes to ECR)
    │  Updates the manifest (writes new part number to the bill of materials)
    ▼
Git Repository  =  Bill of Materials
    │  The official record of exactly which parts go into which car
    │  Every change is version-controlled and auditable
    ▼
ArgoCD CD  =  The Robot Assembly Arm
    │  Reads the bill of materials every 3 minutes
    │  Detects when a part number has changed
    │  Installs the new part (deploys new pods) without stopping the line
    ▼
EKS Cluster  =  The Finished Car
    The running product — always matches the bill of materials
```

| Pipeline Concept | Factory Analogy |
|---|---|
| **Code commit** | Dropping raw material onto the conveyor belt |
| **GitHub Actions** | The quality control + stamping station |
| **AWS ECR** | The parts warehouse with part numbers (image tags) |
| **Git SHA image tag** | The unique serial number stamped on every part |
| **Helm values file update** | Updating the bill of materials with the new part number |
| **ArgoCD polling** | The robot arm checking the bill of materials every 3 minutes |
| **Rolling update** | Replacing tyres while the car keeps moving — no factory shutdown |
| **Helm rollback** | The recall process — swap back to the previous certified part |

> **Key insight:** No human touches the production line after the initial code commit. The pipeline is the assembly line — consistent, fast, auditable, and repeatable every single time.

---

# Rebuild From Scratch — Complete Development Plan

> This plan rebuilds the entire system from zero in strict dependency order.
> Every step has a **verification command** — do not proceed until it passes.
> Estimated total time: **17 working days** for a focused learner.

---

## Pre-Requisites Checklist

Before Day 1, ensure you have:
- [ ] AWS account with billing alerts configured at $50 and $100
- [ ] IAM user (not root) with `AdministratorAccess` — download access keys
- [ ] A registered domain in Route53 (or transfer one) — needed for HTTPS in Step 11
- [ ] GitHub account with a repository named `aws-devops-retail-store`
- [ ] A Linux machine or EC2 instance (Amazon Linux 2023 recommended)

---

## Step 1 — Install All CLI Tools
**Day 1 | ~2 hours**

```bash
# ── AWS CLI ──────────────────────────────────────────────
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
aws --version                          # must print aws-cli/2.x.x

# ── Terraform ────────────────────────────────────────────
sudo yum-config-manager --add-repo \
  https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo dnf install terraform -y
terraform -version                     # must print Terraform v1.6+

# ── kubectl ──────────────────────────────────────────────
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client              # must print Client Version

# ── Helm ─────────────────────────────────────────────────
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version                           # must print v3.x.x

# ── Docker ───────────────────────────────────────────────
sudo dnf install docker -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER && newgrp docker
docker --version                       # must print Docker version 24+

# ── Git ──────────────────────────────────────────────────
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

**✅ Verify:** `aws --version && terraform -version && kubectl version --client && helm version && docker --version`

---

## Step 2 — Configure AWS Access
**Day 1 | ~30 minutes**

```bash
aws configure
# AWS Access Key ID:     <your-iam-user-key>
# AWS Secret Access Key: <your-iam-user-secret>
# Default region:        us-east-1
# Default output:        json

# Verify identity
aws sts get-caller-identity
# Must return your Account ID and IAM user ARN — NOT root
```

**✅ Verify:** `aws sts get-caller-identity` returns your account ID

---

## Step 3 — Bootstrap Terraform Remote State
**Day 1 | ~30 minutes**

```bash
# Unique bucket name
BUCKET="tfstate-retail-$(openssl rand -hex 4)"
echo "Bucket: $BUCKET"   # save this — you will use it in every Terraform backend

# Create S3 bucket
aws s3api create-bucket --bucket $BUCKET --region us-east-1

# Enable versioning (mandatory — enables state recovery)
aws s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket $BUCKET \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Block all public access
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# DynamoDB lock table
aws dynamodb create-table \
  --table-name retail-tf-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**✅ Verify:**
```bash
aws s3api get-bucket-versioning --bucket $BUCKET  # Status: Enabled
aws dynamodb describe-table --table-name retail-tf-state-lock --query 'Table.TableStatus'
# "ACTIVE"
```

---

## Step 4 — Run Application Locally
**Day 1–2 | ~2 hours**

```bash
git clone https://github.com/aws-containers/retail-store-sample-app
cd retail-store-sample-app

# Start all 10 containers
docker compose up -d

# Wait for all containers to be healthy (~2 minutes)
docker compose ps

# Test the app
curl -s http://localhost:8888 | grep -i "retail"   # should find text in HTML
```

**✅ Verify:**
```bash
docker compose ps   # ALL containers show "healthy" or "running"
curl http://localhost:8888/catalogue   # returns JSON product array
```

Explore the app — add items to cart, complete a checkout. Understand what each container does.

```bash
# Clean up when done
docker compose down -v
cd ..
```

---

## Step 5 — Provision VPC with Terraform
**Day 2–3 | ~3 hours**

```bash
mkdir -p retail-store-infra/01_vpc && cd retail-store-infra/01_vpc
```

Create `c1-versions.tf`:
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "tfstate-retail-XXXX"   # your bucket from Step 3
    key            = "vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "retail-tf-state-lock"
    encrypt        = true
  }
}
provider "aws" { region = "us-east-1" }
```

Create `c2-vpc.tf` using the `terraform-aws-modules/vpc/aws` module:
```hcl
data "aws_availability_zones" "available" { state = "available" }

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "retail-store-vpc"
  cidr = "10.0.0.0/16"
  azs  = slice(data.aws_availability_zones.available.names, 0, 3)

  public_subnets  = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false   # one NAT GW per AZ for HA
  enable_dns_hostnames   = true

  # Required tags for EKS to discover subnets
  public_subnet_tags  = { "kubernetes.io/role/elb" = "1" }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = "1" }
}
```

Create `c3-outputs.tf`:
```hcl
output "vpc_id"             { value = module.vpc.vpc_id }
output "public_subnet_ids"  { value = module.vpc.public_subnets }
output "private_subnet_ids" { value = module.vpc.private_subnets }
```

```bash
terraform init && terraform validate
terraform plan -out=vpc.tfplan
terraform apply vpc.tfplan
```

**✅ Verify:**
```bash
terraform output vpc_id          # vpc-xxxxxxxxxxxxxxxxx
terraform output private_subnet_ids   # 3 subnet IDs
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=retail-store-vpc" \
  --query 'Vpcs[0].State'        # "available"
```

---

## Step 6 — Provision EKS Cluster with Terraform
**Day 3–4 | ~4 hours (including 15 min wait)**

```bash
mkdir -p ../02_eks && cd ../02_eks
```

Create `c1-versions.tf` (same backend config, different key: `eks/terraform.tfstate`)

Create `c2-remote-state.tf`:
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "tfstate-retail-XXXX"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Create `c3-eks.tf` using `terraform-aws-modules/eks/aws`:
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "retail-store-eks"
  cluster_version = "1.29"
  vpc_id          = data.terraform_remote_state.vpc.outputs.vpc_id
  subnet_ids      = data.terraform_remote_state.vpc.outputs.private_subnet_ids

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 5
      desired_size   = 3
    }
  }

  cluster_addons = {
    coredns                = {}
    kube-proxy             = {}
    vpc-cni                = {}
    eks-pod-identity-agent = {}
    aws-ebs-csi-driver     = {}
  }
}
```

```bash
terraform init && terraform plan -out=eks.tfplan
terraform apply eks.tfplan      # ~15 minutes

# Connect kubectl
aws eks update-kubeconfig --region us-east-1 --name retail-store-eks
```

**✅ Verify:**
```bash
kubectl get nodes              # 3 nodes, all Ready
kubectl get pods -A            # all system pods Running
kubectl get addon -A 2>/dev/null || aws eks list-addons --cluster-name retail-store-eks
```

---

## Step 7 — Install Helm Controllers
**Day 4 | ~2 hours**

```bash
# ── AWS Load Balancer Controller ──────────────────────────
helm repo add eks https://aws.github.io/eks-charts && helm repo update

# Create IAM role for LBC (using Pod Identity)
# [See course section 11_01 for the full IAM policy document]

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=retail-store-eks \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$(terraform -chdir=../02_eks output -raw vpc_id_FIXME)

# ── Secrets Store CSI Driver ──────────────────────────────
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  -n kube-system --set syncSecret.enabled=true

# ── AWS Secrets and Configuration Provider (ASCP) ─────────
kubectl apply -f \
  https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

# ── External DNS ──────────────────────────────────────────
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns \
  -n kube-system \
  --set provider=aws \
  --set aws.region=us-east-1
```

**✅ Verify:**
```bash
kubectl get deployment aws-load-balancer-controller -n kube-system
kubectl get daemonset secrets-store-csi-driver -n kube-system
kubectl get deployment external-dns -n kube-system
# All should show AVAILABLE = 1+
```

---

## Step 8 — Provision AWS Managed Data Services
**Day 5–6 | ~3 hours**

```bash
mkdir -p ../03_dataplane && cd ../03_dataplane
```

Create Terraform files for:
```hcl
# RDS MySQL (Catalog)
resource "aws_db_instance" "catalog" {
  identifier           = "retail-catalog"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  allocated_storage    = 20
  db_name              = "catalog"
  username             = "catalog_user"
  password             = random_password.catalog_db.result
  multi_az             = true
  db_subnet_group_name = aws_db_subnet_group.main.name
  skip_final_snapshot  = true
}

# RDS PostgreSQL (Orders)  — same pattern, engine = "postgres"
# ElastiCache Redis (Checkout)
# DynamoDB Table (Cart)
# SQS Queue (Orders messaging)
```

Store all credentials in Secrets Manager:
```bash
aws secretsmanager create-secret \
  --name "retail-store/catalog-db" \
  --secret-string "{
    \"username\": \"catalog_user\",
    \"password\": \"$(terraform output -raw catalog_db_password)\",
    \"host\": \"$(terraform output -raw catalog_db_endpoint)\",
    \"port\": \"3306\",
    \"dbname\": \"catalog\"
  }"
```

**✅ Verify:**
```bash
aws rds describe-db-instances \
  --query 'DBInstances[?DBInstanceIdentifier==`retail-catalog`].DBInstanceStatus'
# ["available"]
aws secretsmanager get-secret-value --secret-id retail-store/catalog-db \
  --query SecretString --output text | jq .host
# must return the RDS endpoint hostname
```

---

## Step 9 — Deploy Microservices with Helm
**Day 7–8 | ~3 hours**

```bash
# Clone the application repo
git clone https://github.com/aws-containers/retail-store-sample-app
cd retail-store-sample-app
```

Create `values-aws.yaml` that overrides the defaults to point to AWS managed services:
```yaml
catalog:
  image:
    tag: "1.0.0"
  db:
    endpoint: "retail-catalog.abc123.us-east-1.rds.amazonaws.com"
    secretName: retail-store-catalog-db-k8s-secret

carts:
  dynamodb:
    tableName: retail-store-cart
    region: us-east-1

checkout:
  redis:
    endpoint: "retail-checkout.xyz.cache.amazonaws.com"

orders:
  db:
    endpoint: "retail-orders.abc123.us-east-1.rds.amazonaws.com"
  sqs:
    queueUrl: "https://sqs.us-east-1.amazonaws.com/123456789/retail-orders"
```

```bash
kubectl create namespace retail-store

helm install retail-store ./deploy/helm/retailstore \
  --namespace retail-store \
  --values values-aws.yaml

# Watch rollout
kubectl rollout status deployment/ui -n retail-store
kubectl rollout status deployment/catalog -n retail-store
kubectl rollout status deployment/carts -n retail-store
kubectl rollout status deployment/checkout -n retail-store
kubectl rollout status deployment/orders -n retail-store
```

**✅ Verify:**
```bash
kubectl get pods -n retail-store    # all 5 Deployments: 1/1 Running
kubectl get svc -n retail-store     # all ClusterIP services exist
```

---

## Step 10 — Configure HPA + PDB
**Day 8 | ~1 hour**

```yaml
# catalog-hpa.yaml
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
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
---
# catalog-pdb.yaml
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
```

```bash
kubectl apply -f catalog-hpa.yaml
kubectl apply -f catalog-pdb.yaml
# Repeat for: carts, checkout, orders, ui
```

**✅ Verify:**
```bash
kubectl get hpa -n retail-store    # all HPAs show TARGETS and REPLICAS
kubectl get pdb -n retail-store    # all PDBs show ALLOWED DISRUPTIONS >= 1
```

---

## Step 11 — Expose via ALB + HTTPS + Route53
**Day 9 | ~2 hours**

```bash
# Create ACM certificate for your domain
CERT_ARN=$(aws acm request-certificate \
  --domain-name "retailstore.yourdomain.com" \
  --validation-method DNS \
  --query CertificateArn --output text)

# Complete DNS validation in Route53 (or follow console prompts)
aws acm wait certificate-validated --certificate-arn $CERT_ARN
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store
  namespace: retail-store
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: "<CERT_ARN>"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443},{"HTTP":80}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    external-dns.alpha.kubernetes.io/hostname: "retailstore.yourdomain.com"
spec:
  rules:
  - host: retailstore.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ui
            port:
              number: 8080
```

```bash
kubectl apply -f ingress.yaml

# Watch ALB creation
kubectl get ingress retail-store -n retail-store -w
# Wait until ADDRESS column shows an ALB hostname (not <pending>)
```

**✅ Verify:**
```bash
curl -L https://retailstore.yourdomain.com/catalogue | jq length
# returns product count — app is live end-to-end with HTTPS
```

---

## Step 12 — Install Karpenter + Spot NodePool
**Day 10 | ~2 hours**

```bash
# Install Karpenter via Helm
helm repo add karpenter https://charts.karpenter.sh/
helm install karpenter karpenter/karpenter \
  --namespace karpenter --create-namespace \
  --set settings.clusterName=retail-store-eks \
  --set settings.interruptionQueue=retail-store-karpenter-interruption
```

```yaml
# spot-nodepool.yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: [amd64, arm64]
      - key: karpenter.sh/capacity-type
        operator: In
        values: [spot]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: [t3.medium, t3.large, t3a.medium, m5.large]
  limits:
    cpu: "100"
    memory: 400Gi
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
```

```bash
kubectl apply -f spot-nodepool.yaml
```

**✅ Verify:**
```bash
kubectl get nodepools
kubectl get nodes -L karpenter.sh/capacity-type
# After first scale event, nodes provisioned by Karpenter appear
```

---

## Step 13 — Install ADOT Observability Stack
**Day 11–12 | ~4 hours**

```bash
# Install Cert-Manager (required by ADOT Operator)
kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
kubectl wait --for=condition=available deployment/cert-manager -n cert-manager --timeout=120s

# Install ADOT Operator
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace opentelemetry-operator-system --create-namespace \
  --set admissionWebhooks.certManager.enabled=true

# Install Kube State Metrics
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  -n monitoring --create-namespace

# Install Prometheus Node Exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter \
  -n monitoring
```

Deploy three ADOT Collectors (one per signal):
```bash
# Traces → X-Ray
kubectl apply -f adot-collector-traces.yaml      # Deployment mode, OTLP receiver, awsxray exporter

# Logs → CloudWatch
kubectl apply -f adot-collector-logs.yaml        # DaemonSet mode, filelog receiver, cloudwatchlogs exporter

# Metrics → AMP
kubectl apply -f adot-collector-metrics.yaml     # Deployment mode, prometheus receiver, prometheusremotewrite exporter
```

**✅ Verify:**
```bash
# Generate 10 requests then check X-Ray
for i in {1..10}; do curl -s https://retailstore.yourdomain.com/catalogue > /dev/null; done
sleep 30
aws xray get-trace-summaries \
  --start-time $(date -d '5 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'TraceSummaries[0].Id'    # returns a trace ID

# Check CloudWatch Logs
aws logs describe-log-groups \
  --log-group-name-prefix /aws/containerinsights/retail-store-eks
# Should show the application log group
```

---

## Step 14 — Build the CI Pipeline (GitHub Actions)
**Day 13–14 | ~3 hours**

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name retail-store/ui \
  --image-scanning-configuration scanOnPush=true

# Set up OIDC provider (one-time per AWS account)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create IAM role for GitHub Actions
aws iam create-role \
  --role-name retail-store-github-actions \
  --assume-role-policy-document file://github-trust-policy.json
aws iam attach-role-policy \
  --role-name retail-store-github-actions \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

Create `.github/workflows/build-push-ui.yaml` in your repo:
```yaml
name: Build and Push UI

on:
  push:
    branches: [main]
    paths: ['src/ui/**']

jobs:
  build-and-push-ui:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
    steps:
    - uses: actions/checkout@v4
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::ACCOUNT_ID:role/retail-store-github-actions
        aws-region: us-east-1
    - uses: aws-actions/amazon-ecr-login@v2
    - name: Define image tags
      run: echo "IMAGE_TAG=sha-$(echo $GITHUB_SHA | cut -c1-8)" >> $GITHUB_ENV
    - uses: docker/build-push-action@v5
      with:
        context: ./src/ui
        platforms: linux/amd64,linux/arm64
        push: true
        tags: |
          ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/retail-store/ui:latest
          ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/retail-store/ui:${{ env.IMAGE_TAG }}
    - name: Update Helm values
      run: |
        sed -i "s|tag:.*|tag: \"${{ env.IMAGE_TAG }}\"|" src/ui/chart/values-ui.yaml
        git config user.email "ci-bot@github.com"
        git config user.name "ci-bot"
        git add src/ui/chart/values-ui.yaml
        git commit -m "ci: update UI image to ${{ env.IMAGE_TAG }}"
        git push
```

```bash
# Test: make a small change and push
echo "<!-- rebuild $(date) -->" >> src/ui/src/main/resources/templates/home.html
git add . && git commit -m "test: trigger CI pipeline" && git push
```

**✅ Verify:**
```bash
# GitHub UI → Actions tab → should show green build in ~2 minutes
aws ecr describe-images --repository-name retail-store/ui \
  --query 'imageDetails[0].imageTags'   # ["latest", "sha-xxxxxxxx"]
```

---

## Step 15 — Install ArgoCD and Wire GitOps CD
**Day 15 | ~2 hours**

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Get initial password
ARGOCD_PWD=$(kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD password: $ARGOCD_PWD"

# Access UI (keep this running while you configure)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

Create the ArgoCD Application:
```yaml
# retail-store-argocd-app.yaml
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
      - values-aws.yaml
      - src/ui/chart/values-ui.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: retail-store
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

```bash
kubectl apply -f retail-store-argocd-app.yaml
kubectl get application -n argocd    # Status: Synced, Health: Healthy
```

**✅ Verify:**
```bash
# Make a code change and push
echo "<!-- v2 -->" >> src/ui/src/main/resources/templates/home.html
git add . && git commit -m "feat: v2 header" && git push

# Within 5 minutes:
# 1. GitHub Actions builds and pushes new image to ECR
# 2. CI commits updated image tag to values-ui.yaml
# 3. ArgoCD detects the change and deploys
kubectl get pods -n retail-store -w   # watch new UI pod replace old one
```

---

## Step 16 — Final End-to-End Validation
**Day 16–17 | ~4 hours**

Run every verification from all previous steps, plus:

```bash
# ── Full GitOps round-trip ────────────────────────────────
echo "<!-- final test $(date) -->" >> src/ui/src/main/resources/templates/home.html
git add . && git commit -m "test: final validation" && git push
# Expected: live at https://retailstore.yourdomain.com within 5 minutes

# ── HPA under load ────────────────────────────────────────
kubectl run load --image=busybox -n retail-store --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue; done"
sleep 120
kubectl get hpa -n retail-store    # replicas should have increased
kubectl delete pod load -n retail-store

# ── Drift correction ──────────────────────────────────────
kubectl scale deployment ui --replicas=1 -n retail-store
sleep 200    # ArgoCD detects and fixes within 3 minutes
kubectl get deployment ui -n retail-store    # replicas back to original value

# ── Spot resilience ───────────────────────────────────────
NODE=$(kubectl get nodes -l karpenter.sh/capacity-type=spot \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$NODE" ]; then
  kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
  kubectl get pods -n retail-store    # pods moved to other nodes, PDB respected
  kubectl uncordon $NODE
fi

# ── Observability ─────────────────────────────────────────
aws xray get-trace-summaries \
  --start-time $(date -d '10 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'length(TraceSummaries)'    # should be > 0
```

---

## Rebuild Sequence — At a Glance

```
Step  1  │ Install CLI tools          │ aws, terraform, kubectl, helm, docker
Step  2  │ Configure AWS access       │ IAM user, aws configure, sts verify
Step  3  │ Bootstrap Terraform state  │ S3 bucket + DynamoDB lock table
Step  4  │ Run app locally            │ docker compose up — 10 containers
Step  5  │ Provision VPC              │ terraform apply — subnets, IGW, NAT GW
Step  6  │ Provision EKS cluster      │ terraform apply — managed control plane + nodes
Step  7  │ Install Helm controllers   │ AWS LBC, Secrets Store CSI, External DNS
Step  8  │ Provision data services    │ RDS, DynamoDB, ElastiCache, SQS + Secrets Manager
Step  9  │ Deploy microservices       │ helm install retail-store
Step 10  │ Configure HPA + PDB        │ autoscaling + disruption budgets
Step 11  │ Expose via ALB + HTTPS     │ Ingress + ACM cert + Route53
Step 12  │ Install Karpenter          │ Spot NodePool + interruption handling
Step 13  │ Install ADOT stack         │ Traces → X-Ray, Logs → CW, Metrics → AMP
Step 14  │ Build CI pipeline          │ GitHub Actions + OIDC + ECR
Step 15  │ Wire ArgoCD CD             │ Application CRD + auto-sync
Step 16  │ Final validation           │ End-to-end GitOps + HPA + drift correction
```

---

## Cost Estimate While Building

| Resource | Approximate Daily Cost |
|---|---|
| EKS Cluster (control plane) | $2.40/day |
| 3x t3.medium worker nodes | ~$3.00/day |
| NAT Gateway (3x) | ~$3.30/day |
| RDS MySQL t3.micro Multi-AZ | ~$1.00/day |
| RDS PostgreSQL t3.micro Multi-AZ | ~$1.00/day |
| ElastiCache t3.micro | ~$0.50/day |
| ALB | ~$0.70/day |
| **Total** | **~$12/day (~$85 for 17-day build)** |

> **Cost saving tip:** Run `terraform destroy` on EKS and RDS at the end of each day. Only keep the VPC and S3/DynamoDB (cents per day). Re-apply EKS each morning — takes 15 minutes. Add the EKS and RDS destroy/apply to a startup/shutdown script.

---

# Learn by Doing — Every Component Explained

> Format for each component:
> **Simple** → **Technical** → **Practical Example** → **Debug & Troubleshoot**

---

## Component 1 — Docker

### Simple Terms
Think of Docker like a **lunchbox**. You pack everything your lunch needs — food, fork, napkin — into one box. Wherever you take it (your desk, a park, a friend's house), it works the same. Docker packs your app and everything it needs into one box (called an image). That box runs identically on your laptop, on a build server, and in production.

### Technical Explanation
Docker uses Linux kernel features — **namespaces** (process isolation) and **cgroups** (resource limits) — to create isolated environments called containers. A **Docker image** is a read-only layered filesystem built from a `Dockerfile`. Each `RUN`, `COPY`, and `ADD` instruction creates a new immutable layer. When you `docker run`, Docker adds a thin writable layer on top and starts a process inside the isolated namespace.

**Multi-stage builds** use multiple `FROM` statements to produce a lean final image: the first stage compiles the code (with all build tools), the second stage copies only the compiled binary (no build tools, no source code). Result: a 200MB production image instead of a 1GB build image.

### Practical Example

```bash
# ── See layers in an image ──────────────────────────────
docker pull stacksimplify/retail-store-sample-ui:1.0.0
docker history stacksimplify/retail-store-sample-ui:1.0.0
# Each row is one layer — notice how small the app layer is vs the base OS

# ── Run and get inside a container ─────────────────────
docker run -d --name ui-test -p 8888:8080 \
  stacksimplify/retail-store-sample-ui:1.0.0
docker exec -it ui-test /bin/sh
  # Inside the container:
  ls /app                  # see the app files
  ps aux                   # only 1-2 processes running
  env | grep SPRING        # see environment variables
  exit

# ── Watch resource usage live ───────────────────────────
docker stats ui-test       # CPU%, MEM USAGE, NET I/O in real time

# ── Build your own image ────────────────────────────────
cat > Dockerfile <<'EOF'
FROM public.ecr.aws/amazonlinux/amazonlinux:2023
RUN dnf install -y python3 && dnf clean all
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF
docker build -t my-app:v1 .
docker images | grep my-app      # see the image size
```

### Debug and Troubleshoot

| Symptom | Command | What to Look For |
|---|---|---|
| Container exits immediately | `docker logs <name>` | Error message in the last lines |
| Container starts but port not reachable | `docker ps` | Check PORTS column — must show `0.0.0.0:8888->8080/tcp` |
| "Port already in use" | `ss -tlnp \| grep 8888` | Another process using that port |
| "Image not found" | `docker images` | Check exact image name and tag |
| Container is slow | `docker stats` | Memory limit being hit — increase with `--memory` flag |
| "Permission denied" on docker command | `groups` | You're not in the `docker` group — run `newgrp docker` |

```bash
# Most useful debug workflow:
docker ps -a                        # see ALL containers including stopped ones
docker logs <container-id> --tail 50 # last 50 log lines
docker inspect <container-id>       # full JSON config including IP, mounts, env
docker exec -it <container-id> sh   # get inside and poke around
```

---

## Component 2 — Terraform

### Simple Terms
Imagine you need to set up 50 identical offices in different cities. You could fly to each city and set them up manually — slow, inconsistent, and if you make a mistake in city 30, you won't notice until something breaks. Terraform is the **instruction manual + robot** that sets up every office identically from the same blueprint, records what it built, and can update or demolish any office by updating the blueprint.

### Technical Explanation
Terraform is a **declarative IaC tool**: you declare the desired end state in `.tf` files (HCL syntax) and Terraform figures out the steps to reach that state. It operates in three phases:

1. **Refresh** — reads current real-world state from AWS
2. **Plan** — diffs desired state (`.tf` files) vs current state (`.tfstate`) → produces a change plan
3. **Apply** — executes the minimum API calls to reach desired state

The **state file** (`terraform.tfstate`) is Terraform's memory. It maps your HCL resource names to real AWS resource IDs. If the state is lost, Terraform loses track of what it created. Remote state on S3 + DynamoDB locking prevents concurrent applies and enables team collaboration.

### Practical Example

```bash
# ── Watch what Terraform is doing under the hood ────────
TF_LOG=DEBUG terraform plan 2>&1 | grep "aws_"
# Shows every AWS API call Terraform makes

# ── Understand the state file ───────────────────────────
terraform state list                    # all resources Terraform knows about
terraform state show aws_vpc.main       # full details of one resource
terraform state pull | jq .resources[0] # raw state JSON

# ── Import an existing resource into state ──────────────
# If someone created an S3 bucket manually, import it so Terraform manages it:
terraform import aws_s3_bucket.my_bucket my-bucket-name

# ── Move a resource within state (safe rename) ──────────
terraform state mv aws_instance.old aws_instance.new

# ── Force resource recreation ───────────────────────────
terraform taint aws_instance.web       # mark for destruction+recreation on next apply
terraform plan                         # shows -/+ replace

# ── Target a single resource (useful during debugging) ──
terraform apply -target=aws_vpc.main
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| `Error: state lock` | Another apply is running, or a previous one crashed | `terraform force-unlock <LOCK_ID>` |
| `Error: resource already exists` | Resource was created manually outside Terraform | `terraform import` the resource into state |
| `Error: Provider configuration not present` | Provider block missing or wrong region | Check `c1-versions.tf` provider block |
| `Plan shows destroy + recreate` for a minor change | Some AWS attributes are immutable (e.g. RDS engine version) | Accept the recreation or use `lifecycle { prevent_destroy = true }` |
| `terraform apply` succeeded but resource not visible in AWS console | Wrong region selected in console | Match `provider "aws" { region = ... }` with console region |
| State file shows resources that no longer exist | Resources deleted outside Terraform | `terraform refresh` to sync state with reality |

```bash
# Most useful debug workflow:
terraform validate                    # syntax check first
terraform plan 2>&1 | grep -E "Error|Warning"  # scan for issues
TF_LOG=INFO terraform apply           # verbose logging during apply
terraform show                        # human-readable view of current state
```

---

## Component 3 — VPC Networking

### Simple Terms
The internet is like a city with millions of buildings. Your VPC is a **private gated community** inside that city. You build a wall around your community (the VPC), put a gate at the entrance (Internet Gateway), hire a security guard at the gate (Security Groups), and keep your most valuable stuff in a locked back room (private subnets) that visitors can never reach directly. Only your own staff (EC2 instances) can go in and out of the back room — and only through a monitored side door (NAT Gateway).

### Technical Explanation
A **VPC** (Virtual Private Cloud) is an isolated virtual network in AWS with its own IP address space (CIDR block). Traffic routing is controlled by **Route Tables** attached to subnets:

- **Public subnet** route table: `0.0.0.0/0 → igw-xxx` (traffic to internet goes via IGW)
- **Private subnet** route table: `0.0.0.0/0 → nat-xxx` (outbound internet goes via NAT, no inbound)

**Security Groups** are stateful firewalls at the ENI (network interface) level. Stateful means: if you allow outbound traffic, the return traffic is automatically allowed without an explicit inbound rule.

EKS uses **VPC CNI** (Container Network Interface) — each pod gets a real VPC IP address from the subnet. This means pod-to-pod traffic is native VPC routing, not overlay networking, giving low latency and enabling VPC flow logs for pod-level visibility.

### Practical Example

```bash
# ── Explore your VPC ────────────────────────────────────
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=retail-store-vpc" \
  --query 'Vpcs[0].VpcId' --output text)
echo $VPC_ID

# List all subnets and their types
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}' \
  --output table

# See route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes[*].{Dest:DestinationCidrBlock,GW:GatewayId}}' \
  --output json

# ── Test connectivity from a pod ────────────────────────
kubectl run nettest --image=busybox --restart=Never -- sleep 3600
kubectl exec -it nettest -- sh
  # Test outbound internet (via NAT Gateway):
  wget -q -O- https://ifconfig.me     # returns the NAT Gateway's public IP
  # Test internal VPC:
  nslookup catalog.retail-store.svc.cluster.local
  # Test blocked inbound (should fail):
  wget -q -O- http://169.254.169.254/latest/meta-data/ --timeout=2
  exit
kubectl delete pod nettest
```

### Debug and Troubleshoot

| Symptom | Likely Cause | Fix |
|---|---|---|
| Pod can't reach the internet | NAT Gateway not in route table for private subnet | Check private subnet route table: `0.0.0.0/0 → nat-xxx` |
| ALB not reachable from internet | IGW missing or public subnet route wrong | Verify public subnet route: `0.0.0.0/0 → igw-xxx` |
| Pod can't reach RDS | Security Group on RDS doesn't allow EKS node SG | Add inbound rule: port 3306 from EKS node security group |
| Pods can't talk to each other | VPC CNI misconfigured | `kubectl get pods -n kube-system | grep aws-node` — VPC CNI must be running |
| DNS resolution failing in pod | CoreDNS down | `kubectl get pods -n kube-system | grep coredns` — both pods must be Running |

```bash
# Most useful debug workflow:
# 1. Test DNS from inside a pod:
kubectl run dns-test --image=busybox --restart=Never --rm -it -- nslookup kubernetes

# 2. Check Security Group rules:
aws ec2 describe-security-groups --group-ids sg-xxx \
  --query 'SecurityGroups[0].IpPermissions'

# 3. VPC Flow Logs (if enabled):
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs \
  --filter-pattern "REJECT" --limit 10
```

---

## Component 4 — Kubernetes

### Simple Terms
Imagine you have 100 workers and 50 tasks. You could assign tasks manually — but you'd spend all day managing assignments, dealing with workers who call in sick, and reallocating tasks when someone finishes early. Kubernetes is the **smart HR manager and task scheduler** that does this automatically. You say "I need 3 people always doing Task A" and Kubernetes makes sure that's always true — if one person quits, it immediately hires a replacement.

### Technical Explanation
Kubernetes implements a **control loop** (reconciliation loop) for every resource type:

```
Desired State (what you declared) → Controller watches → Actual State (what exists)
         ↑                                                          │
         └──────────────────── Reconcile ─────────────────────────┘
```

**Key objects:**
- **Pod**: smallest deployable unit — one or more containers sharing a network namespace and volumes
- **Deployment**: manages a ReplicaSet, handles rolling updates and rollbacks
- **ReplicaSet**: ensures N pod replicas exist at all times
- **Service**: stable virtual IP + DNS name that load-balances across matching pods (by label selector)
- **ConfigMap**: injects non-sensitive configuration into pods as env vars or files
- **Secret**: injects sensitive data (base64-encoded — NOT encrypted) as env vars or files

The **scheduler** assigns pods to nodes based on resource requests, affinity rules, and taints/tolerations. The **kubelet** on each node watches its assigned pods and starts/stops containers via the container runtime (containerd).

### Practical Example

```bash
# ── Watch the reconciliation loop live ──────────────────
# Scale a deployment to 0 — watch pods terminate
kubectl scale deployment catalog --replicas=0 -n retail-store
kubectl get pods -n retail-store -w    # watch pods Terminating

# Scale back — watch pods being created
kubectl scale deployment catalog --replicas=3 -n retail-store
kubectl get pods -n retail-store -w    # watch pods ContainerCreating → Running

# ── Understand labels and selectors ─────────────────────
# See what labels the catalog pods have
kubectl get pods -n retail-store -l app=catalog --show-labels

# See what selector the catalog service uses
kubectl get svc catalog -n retail-store -o jsonpath='{.spec.selector}'
# They MUST match for traffic to route to the pods

# ── Understand resource requests and limits ──────────────
kubectl describe pod -l app=catalog -n retail-store | grep -A4 "Limits\|Requests"
# Requests: reserved on the node
# Limits: hard ceiling — container killed if exceeded

# ── See scheduling decisions ─────────────────────────────
kubectl get pods -n retail-store -o wide   # shows which node each pod is on
kubectl describe node <node-name> | grep -A20 "Allocated resources"
```

### Debug and Troubleshoot

| Pod State | Meaning | First Command to Run |
|---|---|---|
| `Pending` | No node found to schedule it | `kubectl describe pod <name>` → look at Events |
| `ContainerCreating` | Image being pulled or volume being attached | `kubectl describe pod <name>` → look at Events |
| `CrashLoopBackOff` | Container starts and immediately crashes | `kubectl logs <name> --previous` |
| `OOMKilled` | Container exceeded memory limit | `kubectl describe pod <name>` → `Last State: OOMKilled` |
| `ImagePullBackOff` | Can't pull the Docker image | Wrong tag, wrong registry, or missing imagePullSecret |
| `Terminating` stuck | Finalizers blocking deletion | `kubectl patch pod <name> -p '{"metadata":{"finalizers":[]}}' --type=merge` |

```bash
# Complete pod debug workflow:
kubectl get pods -n retail-store          # identify the problem pod
kubectl describe pod <pod-name> -n retail-store   # read Events section first
kubectl logs <pod-name> -n retail-store           # current container logs
kubectl logs <pod-name> -n retail-store --previous # previous container logs (if crashed)
kubectl exec -it <pod-name> -n retail-store -- sh  # get inside if it's running

# Node-level issues:
kubectl get nodes                          # check node status
kubectl describe node <node-name>          # look at Conditions and Allocated resources
kubectl top nodes                          # CPU and memory usage per node
kubectl top pods -n retail-store           # CPU and memory usage per pod
```

---

## Component 5 — Helm

### Simple Terms
Writing raw Kubernetes YAML for every environment is like **writing the same letter 10 times** with only the name changed. Helm lets you write the letter once as a template with `{{name}}` placeholders, then fill in different names for each environment. One template file. Infinite personalised letters.

### Technical Explanation
Helm is a **Kubernetes package manager**. A chart is a directory of Go templates (`templates/*.yaml`) plus a `values.yaml` file. At install time, Helm renders the templates by injecting values using Go template syntax (`{{ .Values.image.tag }}`), then applies the resulting manifests to Kubernetes.

Helm tracks **releases** — each `helm install` creates a release, and each `helm upgrade` creates a new revision of that release. This enables atomic upgrades (all resources update together) and rollbacks (`helm rollback` re-applies the previous revision's manifests).

**Helm chart hierarchy:**
```
Chart.yaml          ← chart name, version, description
values.yaml         ← default values
templates/
  deployment.yaml   ← uses {{ .Values.image.tag }}
  service.yaml
  hpa.yaml
  _helpers.tpl      ← reusable template snippets
charts/             ← sub-charts (dependencies)
```

### Practical Example

```bash
# ── Inspect a chart before installing ───────────────────
helm show values ./charts/retail-store          # all available values
helm template retail-store ./charts/retail-store \
  --values values-aws.yaml | grep -A5 "kind: Deployment"
# Dry run — renders all YAML without deploying — great for review

# ── Override values three ways ───────────────────────────
# Method 1: --set (for single values)
helm upgrade retail-store ./charts/retail-store \
  --set catalog.replicaCount=5

# Method 2: -f values file (for many values)
helm upgrade retail-store ./charts/retail-store \
  -f values-prod.yaml

# Method 3: --set-string (for strings with special chars)
helm upgrade retail-store ./charts/retail-store \
  --set-string catalog.image.tag="sha-9d97a3d"

# ── Debug a failing upgrade ──────────────────────────────
helm upgrade retail-store ./charts/retail-store \
  --values values-aws.yaml \
  --dry-run --debug 2>&1 | head -100
# Renders all templates and shows errors without touching the cluster

# ── Track history and rollback ───────────────────────────
helm history retail-store -n retail-store
# REVISION  STATUS     DESCRIPTION
# 1         superseded install complete
# 2         deployed   upgrade complete
# 3         failed     upgrade failed

helm rollback retail-store 2 -n retail-store   # go back to revision 2
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| `helm install` fails with "already exists" | A previous partial install left resources | `helm uninstall retail-store` then reinstall |
| `helm upgrade` fails, leaves release in "failed" state | Bad template or resource conflict | `helm rollback retail-store <prev-revision>` then fix the template |
| Template renders empty value `""` | Wrong `.Values` path | `helm template` + search for the empty value in output |
| "too many open files" during upgrade | Too many Kubernetes objects updated at once | Increase system `ulimit -n` or use `--timeout` flag |
| Resource not updating after `helm upgrade` | Helm computes no diff (immutable field) | Force recreate: `helm upgrade --force` |

```bash
# Most useful Helm debug workflow:
helm status retail-store -n retail-store         # current release status
helm get manifest retail-store -n retail-store   # what's actually deployed
helm get values retail-store -n retail-store     # what values are in effect
helm diff upgrade retail-store ./charts/...      # shows diff (needs helm-diff plugin)
```

---

## Component 6 — AWS Load Balancer + Ingress

### Simple Terms
Imagine a hotel with 10 restaurants. Without a concierge, every guest would wander the building looking for their restaurant. The **concierge** (ALB) stands at the front door, asks "What are you looking for?", and directs you to the right restaurant. The **directory board** (Ingress) tells the concierge the rules: "guests asking for `/italian` go to Floor 2, guests asking for `/sushi` go to Floor 3."

### Technical Explanation
The **AWS Load Balancer Controller** (running as a Deployment in `kube-system`) watches for Kubernetes `Ingress` objects. When it finds one with the `kubernetes.io/ingress.class: alb` annotation, it calls the AWS API to create an **Application Load Balancer** with:
- Listeners (port 80 HTTP, port 443 HTTPS)
- Target Groups (one per service, with pod IPs as targets — `target-type: ip`)
- Rules (routing `/catalogue` to catalog target group, `/` to UI target group)

**External DNS** runs alongside and watches the same Ingress. When it finds `external-dns.alpha.kubernetes.io/hostname: retailstore.example.com`, it creates an A record in Route53 pointing to the ALB's DNS name.

Traffic path:
```
DNS lookup → Route53 A record → ALB DNS name → ALB Listener
→ Routing Rule matches host/path → Target Group → Pod IP:8080
```

### Practical Example

```bash
# ── Watch ALB creation in real time ──────────────────────
kubectl apply -f ingress.yaml
kubectl get ingress retail-store -n retail-store -w
# ADDRESS column starts empty, fills in when ALB is ready (~2 minutes)

# ── Inspect what the controller created in AWS ──────────
ALB_DNS=$(kubectl get ingress retail-store -n retail-store \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $ALB_DNS

aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?DNSName=='$ALB_DNS'].{ARN:LoadBalancerArn,State:State.Code}" \
  --output table

# ── Check target group health ────────────────────────────
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName, 'retail')].TargetGroupArn" \
  --output text | head -1)
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].{IP:Target.Id,Health:TargetHealth.State}'

# ── Test routing rules ────────────────────────────────────
curl -H "Host: retailstore.yourdomain.com" http://$ALB_DNS/catalogue
curl -H "Host: retailstore.yourdomain.com" http://$ALB_DNS/health
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| Ingress stuck at `<pending>` forever | LBC not running or missing IAM permissions | `kubectl logs -n kube-system deploy/aws-load-balancer-controller` |
| ALB created but returns 503 | Target group has no healthy targets | Check pod readiness probes + `describe-target-health` |
| ALB created but Route53 record not created | External DNS misconfigured or wrong hosted zone | `kubectl logs deploy/external-dns` — look for permission errors |
| HTTPS returns certificate error | ACM cert not validated or wrong ARN in annotation | Check cert status: `aws acm describe-certificate --certificate-arn <arn>` |
| 404 on specific path | Ingress path rule missing or wrong `pathType` | Check `kubectl get ingress -o yaml` routing rules |

```bash
# Most useful ALB debug workflow:
kubectl describe ingress retail-store -n retail-store   # events show creation errors
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=50
kubectl logs deploy/external-dns --tail=50
aws elbv2 describe-target-health --target-group-arn <arn>
```

---

## Component 7 — AWS Secrets Manager + Pod Identity

### Simple Terms
Imagine your app needs a house key (database password). You could tape the key to the front door (hardcode in code — terrible idea), keep a copy in your pocket (env var — lost if pocket is stolen), or store it in a bank vault and give specific people a signed note that lets them check out the key for one hour (AWS Secrets Manager + Pod Identity). The bank keeps a record of every time someone checks out the key. If an employee leaves, you revoke their note — the key never changes.

### Technical Explanation
**EKS Pod Identity** works through a chain of trust:
```
Pod ServiceAccount → Pod Identity Association (EKS API)
→ IAM Role → secretsmanager:GetSecretValue permission
→ Pod Identity Agent (DaemonSet on each node) intercepts AWS SDK calls
→ Returns temporary credentials (15-minute TTL)
```

The **Secrets Store CSI Driver** is a Kubernetes volume driver. When a pod mounts a `csi` volume with `driver: secrets-store.csi.k8s.io`, the driver calls the **ASCP** (AWS Secrets and Configuration Provider) which calls `GetSecretValue` using the pod's Pod Identity credentials. The secret value is written to an in-memory `tmpfs` volume (never touches disk) and optionally synced to a Kubernetes Secret for environment variable injection.

### Practical Example

```bash
# ── Create the full chain manually ──────────────────────

# 1. Create the secret
aws secretsmanager create-secret \
  --name "retail-store/catalog-db-password" \
  --secret-string '{"password":"MySecurePass123!"}'

# 2. Create IAM policy
cat > secretsmanager-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
    "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:retail-store/catalog-db-password-*"
  }]
}
EOF
aws iam create-policy \
  --policy-name retail-catalog-secrets-policy \
  --policy-document file://secretsmanager-policy.json

# 3. Create IAM role + Pod Identity Association (via EKS console or AWS CLI)
# 4. Apply SecretProviderClass
kubectl apply -f - <<EOF
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: catalog-db-secret
  namespace: retail-store
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "retail-store/catalog-db-password"
        objectType: "secretsmanager"
        jmesPath:
          - path: password
            objectAlias: db-password
  secretObjects:
  - secretName: catalog-db-k8s-secret
    type: Opaque
    data:
    - objectName: db-password
      key: DB_PASSWORD
EOF

# 5. Verify the secret is mounted
kubectl exec -it deploy/catalog -n retail-store -- cat /mnt/secrets/db-password
kubectl exec -it deploy/catalog -n retail-store -- env | grep DB_PASSWORD
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `Init:0/1` | CSI driver can't fetch secret | `kubectl describe pod <name>` → Events show the CSI error |
| "AccessDeniedException" in pod logs | IAM role missing the secretsmanager permission | Check IAM policy attached to the role in Pod Identity Association |
| "Secret not found" | Wrong secret name in SecretProviderClass | Exact name must match: `aws secretsmanager list-secrets` |
| Secret file exists but is empty | `jmesPath` expression wrong | Test: `aws secretsmanager get-secret-value --secret-id <name> \| jq '.SecretString \| fromjson .password'` |
| K8s Secret not created | `secretObjects` block missing from SecretProviderClass | Secret sync only happens after the volume is mounted by at least one pod |

```bash
# Most useful secrets debug workflow:
kubectl describe pod <pod-name> -n retail-store    # look at Volumes + Events
kubectl get secretproviderclass -n retail-store -o yaml   # verify spec
kubectl get secret catalog-db-k8s-secret -n retail-store  # should exist after pod starts
aws secretsmanager get-secret-value \
  --secret-id retail-store/catalog-db-password     # verify secret exists + value
```

---

## Component 8 — HPA and Autoscaling

### Simple Terms
Imagine a call centre. At 9am, 20 agents handle the morning rush. At 2pm, it's quiet — 5 agents are enough. At 5pm, the rush returns. Without automation, a manager watches the queues all day and moves agents around. HPA is the **automated queue monitor** that watches how busy agents are (CPU%), automatically calls in extra agents when it's busy, and sends them home when it's quiet.

### Technical Explanation
HPA runs as a control loop every **15 seconds**. It queries the `metrics.k8s.io` API (served by Metrics Server) for current CPU/memory across all pods in the target Deployment. It applies the formula:

$$\text{desired replicas} = \left\lceil \text{current replicas} \times \frac{\text{current metric value}}{\text{target metric value}} \right\rceil$$

Scale-up is aggressive (responds within ~1 minute). Scale-down is conservative (default 5-minute stabilisation window) to prevent thrashing.

**Critical dependency:** The pod's `spec.containers[].resources.requests.cpu` MUST be set. Without it, Metrics Server has no baseline to calculate utilisation percentage against.

### Practical Example

```bash
# ── See HPA working in real time ────────────────────────
kubectl get hpa -n retail-store -w
# Columns: NAME, REFERENCE, TARGETS, MINPODS, MAXPODS, REPLICAS, AGE
# TARGETS shows "current/target" — e.g., "23%/50%"

# ── Manually trigger a scale event ──────────────────────
# Scale up by reducing the target (makes current CPU seem high)
kubectl patch hpa catalog-hpa -n retail-store \
  --patch '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":10}}}]}}'
sleep 60
kubectl get hpa catalog-hpa -n retail-store   # replicas should have increased

# Restore
kubectl patch hpa catalog-hpa -n retail-store \
  --patch '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":50}}}]}}'

# ── Generate realistic load ──────────────────────────────
kubectl run load -n retail-store --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://catalog/catalogue > /dev/null; done"
watch -n 5 kubectl get hpa -n retail-store
kubectl delete pod load -n retail-store
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| `TARGETS` shows `<unknown>/50%` | Metrics Server not installed or pod has no CPU request | Install Metrics Server; add `resources.requests.cpu` to deployment |
| HPA never scales up even under load | `resources.requests.cpu` not set | CPU utilisation can't be calculated without a request value |
| HPA scales up but pods stay `Pending` | No nodes with capacity | Karpenter should provision nodes; check `kubectl describe pod` events |
| HPA scales down too aggressively | `stabilizationWindowSeconds` too short | Increase: `behavior.scaleDown.stabilizationWindowSeconds: 300` |
| HPA shows correct metrics but replicas don't change | Already at `maxReplicas` | Increase `maxReplicas` or fix the root performance issue |

```bash
# Most useful HPA debug workflow:
kubectl describe hpa catalog-hpa -n retail-store   # shows events + calculations
kubectl top pods -n retail-store                   # current CPU/memory usage
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods # raw metrics API response
```

---

## Component 9 — ADOT Observability

### Simple Terms
Running software without observability is like driving a car at night with no dashboard — you don't know your speed, fuel level, or if the engine is overheating until you break down. ADOT is the **car dashboard**: traces are the GPS showing your exact route, logs are the trip recorder writing down every event, and metrics are the gauges showing speed, fuel, and temperature in real time.

### Technical Explanation
**ADOT** (AWS Distro for OpenTelemetry) is AWS's supported distribution of the OpenTelemetry Collector. It implements the **pipeline model**:

```
Receivers → Processors → Exporters
(collect)   (transform)  (send)
```

- **Receivers**: listen for incoming telemetry (OTLP gRPC/HTTP) or scrape targets (Prometheus)
- **Processors**: filter, enrich, batch, and rate-limit telemetry before exporting
- **Exporters**: send to destination (awsxray, awscloudwatchlogs, prometheusremotewrite)

The `k8sattributes` processor is critical — it enriches every trace/log/metric with Kubernetes metadata (pod name, namespace, node name, deployment name) by querying the Kubernetes API. This lets you filter X-Ray traces by `k8s.deployment.name=catalog` in CloudWatch.

The ADOT Operator manages collector instances as CRDs (`OpenTelemetryCollector`). It creates and manages the underlying Deployment/DaemonSet/StatefulSet automatically.

### Practical Example

```bash
# ── Generate traffic and find the trace ─────────────────
for i in {1..20}; do
  curl -s https://retailstore.yourdomain.com/catalogue > /dev/null
done
sleep 30   # wait for traces to appear

# Find traces in X-Ray
aws xray get-trace-summaries \
  --start-time $(date -d '5 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'TraceSummaries[0].{Id:Id,Duration:Duration,HasError:HasError}'

# Get the full trace
TRACE_ID=$(aws xray get-trace-summaries \
  --start-time $(date -d '5 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'TraceSummaries[0].Id' --output text)
aws xray batch-get-traces --trace-ids $TRACE_ID \
  --query 'Traces[0].Segments[*].Document' \
  --output text | jq '.name, .start_time, .end_time'

# ── Query CloudWatch Logs ────────────────────────────────
aws logs start-query \
  --log-group-name /aws/containerinsights/retail-store-eks/application \
  --start-time $(date -d '30 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, kubernetes.pod_name, log
    | filter kubernetes.namespace_name = "retail-store"
    | sort @timestamp desc
    | limit 20'

# ── Check ADOT collector pipeline health ─────────────────
kubectl get opentelemetrycollector -n monitoring
kubectl logs -n monitoring deploy/retail-store-traces-collector | tail -20
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| No traces in X-Ray | App not instrumented or wrong OTLP endpoint | Check that app sends to `http://adot-collector:4317`; check collector logs |
| Collector pod in `CrashLoopBackOff` | Config syntax error in OpenTelemetryCollector CRD | `kubectl logs <collector-pod> --previous` — look for config parse error |
| Traces appear but no K8s metadata | `k8sattributes` processor missing or RBAC wrong | Collector ServiceAccount needs `get/list/watch` on pods |
| CloudWatch log group missing | ADOT log collector not running or wrong IAM permissions | Check collector pod has `logs:CreateLogGroup`, `logs:PutLogEvents` via Pod Identity |
| Metrics not appearing in Prometheus | `prometheusremotewrite` exporter failing | Check `sigv4auth` extension is configured; verify AMP workspace URL |

```bash
# Most useful ADOT debug workflow:
kubectl describe opentelemetrycollector -n monitoring    # check spec + status
kubectl logs -n monitoring -l app.kubernetes.io/component=opentelemetry-collector
kubectl port-forward -n monitoring svc/retail-store-traces-collector 8888:8888 &
curl http://localhost:8888/metrics | grep otelcol_exporter   # pipeline metrics
```

---

## Component 10 — GitOps CI/CD Pipeline

### Simple Terms
Old way: a developer finishes code, SSHs into the server, copies files, restarts the app, and hopes it works. If something breaks, nobody knows what changed. GitOps is the new way: the developer pushes code to Git. A robot (GitHub Actions) tests and packages it. Another robot (ArgoCD) notices the package and deploys it automatically. Every step is recorded in Git history forever. To undo a mistake, you just undo the Git commit.

### Technical Explanation
The pipeline has two distinct phases with a clean handoff through Git:

**CI (GitHub Actions):**
1. Triggered on `push` to `main` matching `paths: ['src/ui/**']`
2. Uses OIDC (not stored keys) to get temporary AWS credentials via `sts:AssumeRoleWithWebIdentity`
3. Builds a multi-platform image with `docker buildx` (QEMU emulation for ARM)
4. Pushes to ECR with two tags: `latest` (mutable reference) and `sha-xxxxxxxx` (immutable, traceable)
5. Updates `values-ui.yaml` with the new SHA tag → commits → pushes to Git

**CD (ArgoCD):**
1. Polls Git every 3 minutes (or receives a webhook push for faster response)
2. Detects diff: `values-ui.yaml` has new image tag that differs from what's in the cluster
3. Runs `helm upgrade` equivalent (server-side apply)
4. Kubernetes rolls out the new pods using `RollingUpdate` strategy
5. `selfHeal: true` → detects any manual cluster change and reverts to Git state

### Practical Example

```bash
# ── Trace a deployment end-to-end ───────────────────────

# Step 1: Make a code change
echo "<!-- rebuild $(date) -->" >> src/ui/src/main/resources/templates/home.html
git add . && git commit -m "test: trigger pipeline $(date +%s)" && git push

# Step 2: Watch GitHub Actions (in your browser at github.com/your-repo/actions)
# Or via CLI:
gh run list --limit 3   # needs GitHub CLI installed

# Step 3: Verify ECR has the new image
sleep 120    # wait for CI to finish
aws ecr describe-images \
  --repository-name retail-store/ui \
  --query 'sort_by(imageDetails, &imagePushedAt)[-1].imageTags'

# Step 4: Check Git was updated by ci-bot
git pull && git log --oneline -3
# Should show: "ci: update UI image to sha-xxxxxxxx"

# Step 5: Watch ArgoCD deploy
kubectl get pods -n retail-store -l app=ui -w
# Old pod Terminating + new pod Running = successful rolling update

# Step 6: Verify the image tag on the running pod
kubectl get pod -n retail-store -l app=ui \
  -o jsonpath='{.items[0].spec.containers[0].image}'
# Should contain sha-xxxxxxxx — matching the Git commit

# ── Test ArgoCD self-healing ─────────────────────────────
kubectl scale deployment ui --replicas=1 -n retail-store  # manual change
sleep 10
kubectl get deployment ui -n retail-store -o jsonpath='{.spec.replicas}'
# ArgoCD will restore this to original value within 3 minutes
```

### Debug and Troubleshoot

| Symptom | Cause | Fix |
|---|---|---|
| GitHub Actions fails at "Configure AWS credentials" | OIDC trust policy wrong — repo name mismatch | Check `sub` condition in trust policy matches your exact `org/repo` |
| GitHub Actions fails at "Push to ECR" | IAM role missing ECR push permissions | Attach `AmazonEC2ContainerRegistryPowerUser` to the GitHub Actions role |
| CI commit triggers another CI run (loop) | `paths` filter not excluding the chart directory | Add `paths-ignore: ['**/values-ui.yaml']` or move chart to separate repo |
| ArgoCD stuck in `OutOfSync` | Helm render error or resource conflict | `argocd app diff retail-store` to see what's different |
| ArgoCD shows `Synced` but old image is still running | Helm release not actually upgraded | Check `helm history retail-store -n retail-store` for failed revision |
| ArgoCD not self-healing drift | `selfHeal: false` or ArgoCD reconciliation paused | Check `argocd app get retail-store` → `Auto Sync` field |

```bash
# Most useful GitOps debug workflow:
argocd app get retail-store                    # full sync status
argocd app diff retail-store                   # shows what would change
argocd app logs retail-store                   # ArgoCD controller logs for this app
kubectl rollout history deployment/ui -n retail-store  # see all K8s rollout revisions
kubectl rollout undo deployment/ui -n retail-store     # emergency K8s-level rollback
```

---

## Quick Debugging Decision Tree

When something is wrong, follow this order:

```
Something is broken
│
├── Is a pod not running?
│   ├── kubectl get pods -A              → find the broken pod
│   ├── kubectl describe pod <name>      → read Events section
│   ├── kubectl logs <name> --previous   → read crash logs
│   └── kubectl exec -it <name> -- sh   → get inside if running
│
├── Is the app unreachable?
│   ├── kubectl get ingress              → check ALB DNS
│   ├── kubectl get endpoints            → check pods are registered
│   ├── curl -v http://alb-dns.com      → check ALB responds
│   └── kubectl logs deploy/aws-load-balancer-controller
│
├── Is a Terraform apply failing?
│   ├── terraform validate               → syntax errors
│   ├── TF_LOG=INFO terraform plan       → verbose AWS API errors
│   ├── terraform state list             → check what state knows
│   └── aws console                      → check if resource exists manually
│
├── Is ArgoCD not deploying?
│   ├── argocd app get retail-store      → check sync status
│   ├── argocd app diff retail-store     → see what's different
│   └── kubectl logs -n argocd deploy/argocd-application-controller
│
└── Is autoscaling not working?
    ├── kubectl get hpa                  → check TARGETS column
    ├── kubectl top pods                 → verify metrics are flowing
    └── kubectl describe hpa <name>      → read calculation details
```

---

# Master Checklists

> Use these checklists as daily references.
> Each item is a concrete, verifiable action — not a vague concept.
> Check off items only when the verification command passes.

---

## Checklist 1 — Understanding the Architecture

### 1.1 Application Layer
- [ ] I can name all 5 microservices, their language, and their database
- [ ] I can draw the request flow: `User → DNS → ALB → UI → Catalog/Cart/Checkout/Orders`
- [ ] I understand why UI is the only service with an Ingress (others are internal-only)
- [ ] I can explain why Cart uses DynamoDB instead of MySQL
- [ ] I can explain why Orders writes to both PostgreSQL AND SQS
- [ ] I understand what `ExternalName` Service type does and why it requires zero app code changes
- [ ] I can explain the difference between `ClusterIP`, `NodePort`, `LoadBalancer`, and `ExternalName`

### 1.2 Infrastructure Layer
- [ ] I can explain why VPC has both public and private subnets
- [ ] I can explain the exact path a packet takes from the internet to a pod
- [ ] I can explain what happens if the NAT Gateway is deleted (pods lose outbound internet)
- [ ] I understand why EKS control plane is in a separate AWS-managed VPC
- [ ] I can explain what `kubeconfig` is and where it lives on disk (`~/.kube/config`)
- [ ] I can explain the difference between a Terraform `resource` block and a `data` block

### 1.3 Security Layer
- [ ] I can explain why Kubernetes Secrets are NOT secure (base64 ≠ encryption)
- [ ] I can explain the Pod Identity credential chain: ServiceAccount → IAM Role → AWS API
- [ ] I can explain why OIDC is safer than storing AWS keys in GitHub Secrets
- [ ] I understand what `selfHeal: true` in ArgoCD prevents (manual drift)
- [ ] I can explain what happens to a secret in a pod when the pod restarts (re-fetches from Secrets Manager)

### 1.4 Scaling and Resilience Layer
- [ ] I can explain the HPA formula with a concrete example (e.g., 3 pods × 80% ÷ 50% = 5 pods)
- [ ] I understand what `minAvailable` in a PDB guarantees during a node drain
- [ ] I can explain all 6 stages of Spot interruption handling (EventBridge → SQS → Karpenter → drain → PDB → terminate)
- [ ] I understand why scale-down has a 5-minute delay but scale-up is fast

### 1.5 Observability Layer
- [ ] I can explain the difference between a trace, a span, a log line, and a metric
- [ ] I can explain why ADOT uses a DaemonSet for logs but a Deployment for traces
- [ ] I understand the ADOT pipeline: receiver → processor → exporter
- [ ] I can explain what the `k8sattributes` processor adds to every telemetry signal

### 1.6 CI/CD Layer
- [ ] I can draw the full GitOps flow from `git push` to running pod (7 steps)
- [ ] I can explain why `sha-xxxxxxxx` tag is used instead of `latest` in production
- [ ] I understand what ArgoCD `prune: true` does to removed resources
- [ ] I can explain the difference between CI (GitHub Actions) and CD (ArgoCD)

**Architecture Understanding Score:** ___/30 items checked

---

## Checklist 2 — Setting Up the Project

### 2.1 Tools and Access
- [ ] `aws --version` returns `aws-cli/2.x.x` or higher
- [ ] `terraform -version` returns `Terraform v1.6+`
- [ ] `kubectl version --client` returns client version
- [ ] `helm version` returns `v3.x.x`
- [ ] `docker --version` returns `Docker version 24+`
- [ ] `aws sts get-caller-identity` returns your Account ID (NOT root account)
- [ ] `git config --list` shows your name and email

### 2.2 AWS Account
- [ ] Billing alert created at $50
- [ ] Billing alert created at $100
- [ ] IAM user exists with `AdministratorAccess` (not root)
- [ ] MFA enabled on root account
- [ ] Default region confirmed as `us-east-1` in `aws configure`

### 2.3 Terraform State Bootstrap
- [ ] S3 bucket created with a unique name (hex suffix)
- [ ] S3 bucket versioning is `Enabled`
- [ ] S3 bucket encryption is `AES256`
- [ ] S3 bucket public access is fully blocked
- [ ] DynamoDB table `retail-tf-state-lock` exists with status `ACTIVE`
- [ ] Bucket name saved somewhere — you need it in every Terraform backend config

### 2.4 Repository Setup
- [ ] GitHub repository `aws-devops-retail-store` created
- [ ] Local git clone configured with correct remote URL
- [ ] `.gitignore` includes: `*.tfstate`, `*.tfstate.backup`, `.terraform/`, `.env`, `*.pem`
- [ ] Branch protection on `main` — direct pushes blocked (optional but best practice)

### 2.5 Infrastructure Provisioned
- [ ] `terraform apply` on VPC project succeeded — `terraform output` shows vpc_id
- [ ] `terraform apply` on EKS project succeeded — takes ~15 minutes
- [ ] `aws eks update-kubeconfig` run — `kubectl get nodes` shows 3 `Ready` nodes
- [ ] `kubectl get pods -A` shows zero `Pending` or `CrashLoopBackOff` pods
- [ ] AWS Load Balancer Controller deployment is `Available`
- [ ] Karpenter deployment is `Available`
- [ ] Secrets Store CSI Driver DaemonSet is `Available`
- [ ] External DNS deployment is `Available`

**Setup Completeness Score:** ___/27 items checked

---

## Checklist 3 — Running the Project

### 3.1 Local Run (Docker Compose)
- [ ] `docker compose up -d` starts all 10 containers without errors
- [ ] `docker compose ps` shows all containers as `healthy`
- [ ] `curl http://localhost:8888/catalogue` returns a JSON array
- [ ] Can add items to cart at `http://localhost:8888`
- [ ] Can complete a checkout flow end-to-end
- [ ] `docker compose down -v` stops everything cleanly

### 3.2 Kubernetes Run (EKS)
- [ ] `kubectl get pods -n retail-store` shows all 5 deployments `Running`
- [ ] `kubectl get svc -n retail-store` shows all ClusterIP services
- [ ] `kubectl get endpoints -n retail-store` shows populated IP:port pairs for every service
- [ ] `kubectl get ingress -n retail-store` shows an ALB hostname (not `<pending>`)
- [ ] `kubectl get hpa -n retail-store` shows correct `TARGETS` (not `<unknown>`)
- [ ] `kubectl get pdb -n retail-store` shows all PDBs with `ALLOWED DISRUPTIONS >= 1`

### 3.3 Public Access Verified
- [ ] Route53 A record exists for your domain pointing to the ALB
- [ ] `curl -L https://retailstore.yourdomain.com/catalogue | jq length` returns a number
- [ ] HTTPS certificate is valid (no browser warnings)
- [ ] Full user journey works: browse → add to cart → checkout → order placed

### 3.4 Observability Running
- [ ] `kubectl get opentelemetrycollector -n monitoring` shows collectors
- [ ] X-Ray service map shows at least 3 connected services after generating traffic
- [ ] CloudWatch log group `/aws/containerinsights/retail-store-eks/application` exists
- [ ] Amazon Managed Prometheus workspace is `ACTIVE`
- [ ] Grafana workspace is accessible and Prometheus datasource is configured

### 3.5 GitOps Running
- [ ] `argocd app get retail-store` shows `Status: Synced, Health: Healthy`
- [ ] ArgoCD UI accessible and shows green status for retail-store application
- [ ] Making a code change and pushing triggers a GitHub Actions run within 30 seconds
- [ ] GitHub Actions run completes successfully (all green checkmarks)
- [ ] New image tag visible in ECR after CI run
- [ ] ArgoCD detects and deploys the new image within 3 minutes of CI commit

**Running Verification Score:** ___/27 items checked

---

## Checklist 4 — Modifying the Project

> Before making ANY modification, complete these pre-flight steps:

### 4.1 Pre-Modification Safety
- [ ] Current `argocd app get retail-store` shows `Synced` — cluster matches Git
- [ ] `kubectl get pods -n retail-store` — all pods healthy before you start
- [ ] You are on a feature branch — NOT committing directly to `main`
- [ ] You have noted the current Helm release revision: `helm history retail-store`
- [ ] You know the rollback command: `helm rollback retail-store <prev-revision>`

### 4.2 Modifying Application Code (Microservice)
- [ ] Identify which microservice owns the change (catalog, carts, checkout, orders, or ui)
- [ ] Run the service locally with Docker Compose before changing EKS
- [ ] Write or update the relevant unit tests
- [ ] Build the image locally and confirm it starts: `docker build -t test:v1 . && docker run -p 8080:8080 test:v1`
- [ ] Push to feature branch — confirm CI pipeline builds successfully
- [ ] Check ECR for the new image tag
- [ ] Merge to `main` — ArgoCD deploys automatically
- [ ] Confirm new pods are `Running` and old pods are `Terminating`
- [ ] Smoke test the modified endpoint via `curl`
- [ ] Check X-Ray for any new errors in the trace map

### 4.3 Modifying Kubernetes Configuration
- [ ] Edit the relevant Helm template or `values.yaml`
- [ ] Dry-run first: `helm template retail-store ./charts --values values-aws.yaml`
- [ ] Review the rendered YAML diff — no unexpected changes
- [ ] Apply with `helm upgrade retail-store ./charts --values values-aws.yaml`
- [ ] Watch rollout: `kubectl rollout status deployment/<name> -n retail-store`
- [ ] Confirm `kubectl get pods -n retail-store` all healthy
- [ ] If broken: `helm rollback retail-store <prev-revision>`

### 4.4 Modifying Infrastructure (Terraform)
- [ ] Pull latest state: `terraform refresh` before making changes
- [ ] Make changes to `.tf` files only — never edit `.tfstate` directly
- [ ] Run `terraform validate` — no errors
- [ ] Run `terraform plan -out=change.tfplan` and READ every line carefully
- [ ] Confirm no unexpected `destroy` or `replace` operations in the plan
- [ ] Run `terraform apply change.tfplan`
- [ ] Verify the changed resource in AWS console matches expectations
- [ ] Run `kubectl get nodes` if EKS was modified — all nodes still `Ready`
- [ ] Run `kubectl get pods -A` — no new failures

### 4.5 Modifying Secrets
- [ ] Update the secret value in AWS Secrets Manager (never in code or Git)
- [ ] Restart the affected pod to pick up the new value: `kubectl rollout restart deployment/<name> -n retail-store`
- [ ] Verify the new secret is mounted: `kubectl exec -it <pod> -- cat /mnt/secrets/<key>`
- [ ] Confirm the app is working with the new secret value
- [ ] Document the rotation in your team's runbook (not in Git)

**Modification Safety Score:** ___/29 items checked

---

## Checklist 5 — Adding a New Feature

> Follow this checklist in order for every new feature, regardless of size.

### 5.1 Design Phase
- [ ] Clearly defined: what does the feature do?
- [ ] Clearly defined: which microservice(s) does it touch?
- [ ] Does it need a new API endpoint? What path? What HTTP method?
- [ ] Does it need a new database table or DynamoDB attribute?
- [ ] Does it need a new AWS resource? (queue, bucket, cache, secret)
- [ ] Does it introduce a new inter-service call? Document the dependency
- [ ] Is there a risk of breaking existing functionality? Plan a rollback

### 5.2 Local Development
- [ ] Create a feature branch: `git checkout -b feature/my-feature`
- [ ] Update `docker-compose.yaml` if new environment variables are needed
- [ ] Implement the feature code in the microservice
- [ ] Run `docker compose up -d` and test the feature locally
- [ ] Confirm existing endpoints still work (no regression)
- [ ] Write a test that verifies the new behaviour

### 5.3 Infrastructure (if new AWS resource needed)
- [ ] Add the Terraform resource definition in `03_dataplane/`
- [ ] Run `terraform plan` — review what new resources will be created
- [ ] Apply in dev environment first: `terraform apply -target=<new_resource>`
- [ ] Verify the resource in AWS console
- [ ] Create the secret in AWS Secrets Manager if credentials are needed
- [ ] Add IAM permissions to the relevant Pod Identity role for the new resource

### 5.4 Kubernetes Configuration
- [ ] Add new environment variables to the Helm chart `values.yaml`
- [ ] Add the new `SecretProviderClass` if new secrets are needed
- [ ] If it's a new microservice: create Deployment, Service, HPA, PDB templates
- [ ] Dry-run: `helm template retail-store ./charts --values values-aws.yaml`
- [ ] Check rendered YAML — new resources look correct, no existing resources broken
- [ ] If a new HTTP path is exposed externally: add Ingress rule for the path

### 5.5 CI/CD Pipeline Update
- [ ] If a new microservice was added: create a new ECR repository
- [ ] If a new microservice was added: add a new GitHub Actions workflow for it
- [ ] Update `values.yaml` with new image repository and default tag
- [ ] Add the new service to ArgoCD Application's Helm values list
- [ ] Test: push to feature branch, confirm CI builds successfully without deploying to prod

### 5.6 Deployment to Production
- [ ] All tests passing on feature branch
- [ ] Code reviewed and approved (PR merged to `main`)
- [ ] CI pipeline completes — new image in ECR with SHA tag
- [ ] ArgoCD shows `OutOfSync` → `Syncing` → `Synced`
- [ ] New pods are `Running` with correct image tag
- [ ] New feature works end-to-end via `https://retailstore.yourdomain.com`
- [ ] No new errors in X-Ray trace map
- [ ] No new ERROR lines in CloudWatch Logs
- [ ] HPA and PDB exist for the new deployment (if it's a new microservice)

### 5.7 Post-Deployment Validation
- [ ] Monitor X-Ray for 15 minutes — no elevated error rates
- [ ] Monitor Grafana for 15 minutes — no CPU/memory spikes
- [ ] Confirm existing features still work (regression check)
- [ ] If anything breaks: `argocd app rollback retail-store` or `helm rollback retail-store <n>`
- [ ] Update team runbook or documentation with the new feature details

**New Feature Readiness Score:** ___/43 items checked

---

## Combined Master Progress Tracker

| Checklist | Total Items | Your Score | Status |
|---|---|---|---|
| 1 — Architecture Understanding | 30 | ___/30 | ⬜ Not started |
| 2 — Project Setup | 27 | ___/27 | ⬜ Not started |
| 3 — Running the Project | 27 | ___/27 | ⬜ Not started |
| 4 — Modifying the Project | 29 | ___/29 | ⬜ Not started |
| 5 — Adding a New Feature | 43 | ___/43 | ⬜ Not started |
| **Total** | **156** | **___/156** | |

> **Scoring guide:**
> - 0–50: You are in the learning phase — focus on Checklists 1 and 2 first
> - 51–100: You can run and observe the system — move to Checklist 3 and 4
> - 101–130: You are productive — work on exercises and new features
> - 131–156: You are production-ready for this stack
