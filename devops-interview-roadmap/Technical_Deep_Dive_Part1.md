# Technical Deep Dive - Interview Question Answers

## AWS LAMBDA

### Q: Explain Lambda cold starts and optimization strategies

**Answer:**
A cold start occurs when AWS Lambda needs to initialize a new execution environment. This happens when:
1. Function is invoked for the first time
2. Scaling up (all warm containers are busy)
3. After code/configuration updates
4. After ~15 minutes of inactivity

**Cold Start Components (Total: 100-300ms):**
1. **Download code** (~50-100ms)
2. **Start execution environment** (~50-100ms)
3. **Initialize runtime** (~50-100ms)
4. **Run initialization code** (variable)

**Optimization Strategies:**

**1. Provisioned Concurrency**
```python
# Keep N instances always warm
# Cost: $0.015 per GB-hour provisioned
# Use for: Critical APIs, scheduled events
```
- Eliminates cold starts completely
- Instances are always initialized and ready
- More expensive but predictable latency

**2. Reserved Concurrency**
```python
# Guarantees max concurrent executions
# Prevents function from consuming all account concurrency
# Doesn't prevent cold starts, just reserves capacity
```

**3. Reduce Package Size**
```python
# Before: 50MB package = 200ms cold start
# After: 10MB package = 80ms cold start

# Use Lambda Layers for shared dependencies
# Exclude unnecessary files (tests, docs)
# Use webpack/bundlers to minimize code
```

**4. Optimize Init Code**
```python
# BAD - DB connection in handler
def lambda_handler(event, context):
    db = connect_to_database()  # Runs every invocation!
    return db.query()

# GOOD - DB connection outside handler
db = connect_to_database()  # Runs once during cold start

def lambda_handler(event, context):
    return db.query()  # Reuses connection
```

**5. Choose Right Runtime**
- Interpreted (Python, Node.js): 100-200ms cold start
- Compiled (Go, Rust): 20-50ms cold start
- Java: 300-600ms cold start (use Snap Start for Java)

**6. VPC Optimization**
```python
# Old VPC Lambda: +10 seconds cold start (ENI creation)
# New Hyperplane: +100ms cold start

# Best practice: Use VPC endpoints instead of NAT gateway
# Reduces latency and cost
```

**7. Memory Allocation**
```python
# More memory = More CPU = Faster initialization
# Find sweet spot with Power Tuning tool

# Example: 512MB vs 1024MB
# Cold start: 300ms vs 180ms
# Cost: 2x memory but execution time halved
```

**8. Keep Functions Warm**
```python
# Periodic pings every 5 minutes
# CloudWatch Events → Lambda (light invocation)
# Cheap way to reduce cold starts for low-traffic functions
```

**Your Drishya Example:**
"At Drishya, I optimized our Lambda-based API:
- Reduced package size from 45MB to 8MB using layers
- Moved database connection to init code
- Used Python 3.11 (20% faster than 3.8)
- Implemented provisioned concurrency for critical endpoints
- Result: P95 latency from 400ms to <180ms, 55% reduction"

---

### Q: Lambda concurrency limits and how to handle them

**Answer:**

**Concurrency Limits:**
1. **Account-level default:** 1,000 concurrent executions (per region)
2. **Can request increase** up to tens of thousands
3. **Burst concurrency:**
   - Initial burst: 500-3000 (region dependent)
   - After burst: +500/minute

**Types of Concurrency:**

**1. Unreserved Concurrency**
- Default pool shared by all functions
- If one function uses 900, others get only 100

**2. Reserved Concurrency**
```python
# Guarantees this function can use up to N concurrent executions
# Subtracts from unreserved pool
# Example: Set to 200
# - This function: max 200 concurrent
# - Other functions: max 800 concurrent (1000 - 200)
```

**3. Provisioned Concurrency**
```python
# Pre-initialized execution environments
# Always warm, no cold starts
# Cost: ~$0.015 per GB-hour
# Use for: Latency-sensitive workloads
```

**Handling Throttling:**

**1. Asynchronous Invocation**
```python
# Automatic retry with exponential backoff
# Max retry attempts: 2
# Event goes to DLQ (Dead Letter Queue) after failures

import boto3
lambda_client = boto3.client('lambda')

lambda_client.invoke(
    FunctionName='my-function',
    InvocationType='Event',  # Asynchronous
    Payload=json.dumps(event)
)
```

**2. SQS Integration**
```python
# SQS acts as buffer
# Lambda polls queue (1-1000 messages per poll)
# Automatic throttling protection
# Messages stay in queue if function throttled

API Gateway → SQS → Lambda
# Decouples ingestion from processing
# Handles traffic spikes gracefully
```

**3. API Gateway Throttling**
```python
# Set at API Gateway level
# Prevents overwhelming Lambda
# Returns 429 (Too Many Requests) to client

# Default: 10,000 req/sec, 5,000 burst
# Can be configured per stage/method
```

**4. Step Functions**
```python
# Orchestrate long-running workflows
# Built-in retry and error handling
# Doesn't consume Lambda concurrency while waiting

Step Functions → Lambda (short tasks)
# Better than one long-running Lambda
```

**Your Drishya Example:**
"We had a data processing pipeline handling 10M+ events/day:
- Used SQS to buffer incoming events (peak: 50K/min)
- Set Lambda reserved concurrency to 800 (leaving 200 for other functions)
- Implemented DLQ for failed events
- Added CloudWatch alarm at 90% concurrency usage
- Result: Zero throttling issues, 99.9% success rate"

---

## TERRAFORM

### Q: Terraform state management best practices

**Answer:**

**State File Basics:**
```hcl
# State stores:
# 1. Resource mappings (Terraform ID ↔ Cloud Resource ID)
# 2. Resource metadata (dependencies, outputs)
# 3. Performance cache (avoids API calls every time)
```

**Remote State (Production Best Practice):**

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                    # Encryption at rest
    dynamodb_table = "terraform-state-lock" # State locking
    
    # Optional: Versioning enabled on S3 bucket
    # Optional: MFA delete on S3 bucket
  }
}

# DynamoDB table for locking (prevents concurrent modifications)
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "terraform-state-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**State Locking:**
```bash
# When you run 'terraform apply':
# 1. Terraform acquires lock in DynamoDB
# 2. Runs plan/apply
# 3. Releases lock

# If another user tries to apply simultaneously:
# Error: Error locking state: ConditionalCheckFailedException
# Prevents state corruption
```

**Best Practices:**

**1. Never Store State in Git**
```bash
# .gitignore
*.tfstate
*.tfstate.backup
.terraform/
```

**2. Use State File Versioning**
```hcl
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Allows rollback if state corrupted
# aws s3api list-object-versions --bucket my-terraform-state
```

**3. Separate State Files by Environment**
```
terraform/
├── prod/
│   ├── backend.tf (key = "prod/terraform.tfstate")
│   └── main.tf
├── staging/
│   ├── backend.tf (key = "staging/terraform.tfstate")
│   └── main.tf
└── dev/
    ├── backend.tf (key = "dev/terraform.tfstate")
    └── main.tf

# Prevents accidental prod changes
# Blast radius containment
```

**4. Workspaces (Alternative to Multiple State Files)**
```bash
# Create workspace
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select prod

# State stored as:
# prod: terraform.tfstate.d/prod/terraform.tfstate
# staging: terraform.tfstate.d/staging/terraform.tfstate

# Use Case: Same infrastructure, different environments
# Variable based on workspace:
resource "aws_instance" "app" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.small"
}
```

**5. State Migration**
```bash
# From local to remote:
# 1. Configure backend in code
# 2. Run terraform init -migrate-state
# 3. Confirm migration
# 4. Verify state in S3

# Between backends:
terraform init -migrate-state -backend-config=new-backend.hcl
```

**6. State Backup**
```bash
# Manual backup
terraform state pull > backup-$(date +%Y%m%d).tfstate

# S3 versioning provides automatic backups
# Can restore:
aws s3api get-object \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate \
  --version-id <VERSION_ID> \
  terraform.tfstate
```

**7. State Operations (Use with Caution)**

```bash
# List resources
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Move resource (refactoring)
terraform state mv aws_instance.old aws_instance.new

# Remove resource from state (doesn't delete actual resource)
terraform state rm aws_instance.old

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0

# Replace resource
terraform apply -replace="aws_instance.web"
```

**8. Drift Detection**
```bash
# Detect drift (state vs actual infrastructure)
terraform plan -refresh-only

# Shows what changed outside Terraform
# Options:
# 1. Update state: terraform apply -refresh-only
# 2. Fix infrastructure: terraform apply
# 3. Investigate: Who changed it? Why?
```

**Your Drishya Example:**
"At Drishya, I managed Terraform for 15+ AWS accounts:
- S3 backend with versioning and encryption
- DynamoDB for state locking
- Separate state files per account and environment (45 total state files)
- Automated state backup to separate bucket (daily)
- Implemented Terraform Cloud for team collaboration
- Result: Zero state corruption issues, 90% reduction in provisioning errors"

---

### Q: Terraform modules - design and best practices

**Answer:**

**Module Structure:**

```
terraform-aws-vpc/
├── README.md
├── main.tf        # Primary resources
├── variables.tf   # Input variables
├── outputs.tf     # Output values
├── versions.tf    # Provider version constraints
├── examples/      # Example usage
│   └── basic/
│       ├── main.tf
│       └── README.md
└── tests/         # Automated tests
    └── vpc_test.go
```

**Good Module Design:**

**1. Single Responsibility**
```hcl
# BAD: One module for everything
module "infrastructure" {
  # Creates VPC, EKS, RDS, ElastiCache, ALB, etc.
}

# GOOD: Separate modules
module "vpc" { }
module "eks" { }
module "rds" { }
```

**2. Required vs Optional Variables**
```hcl
# variables.tf
variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  # No default = REQUIRED
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway"
  type        = bool
  default     = true  # Optional with sensible default
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

**3. Output Useful Values**
```hcl
# outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Private subnet IDs for application deployment"
  value       = aws_subnet.private[*].id
}

output "database_subnet_group" {
  description = "DB subnet group for RDS"
  value       = aws_db_subnet_group.main.name
}

# These outputs are used by other modules
```

**4. Module Composition**
```hcl
# Root module uses child modules
module "vpc" {
  source = "./modules/vpc"
  
  vpc_cidr = "10.0.0.0/16"
  environment = "prod"
}

module "eks" {
  source = "./modules/eks"
  
  vpc_id            = module.vpc.vpc_id           # Use VPC output
  private_subnets   = module.vpc.private_subnet_ids
  cluster_version   = "1.28"
}

module "rds" {
  source = "./modules/rds"
  
  subnet_group_name = module.vpc.database_subnet_group
  vpc_id            = module.vpc.vpc_id
}
```

**5. Module Versioning**
```hcl
# Use Git tags for versioning
module "vpc" {
  source = "git::https://github.com/company/terraform-aws-vpc.git?ref=v2.1.0"
  
  # Terraform Registry (better)
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"  # Semantic versioning
}

# Version constraints
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # Any 5.x version >= 5.0
}
```

**6. Conditional Resources**
```hcl
# Create resource only if condition met
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(var.availability_zones) : 0
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

# for_each (better for maps)
resource "aws_subnet" "private" {
  for_each = var.private_subnets
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  
  tags = {
    Name = each.key
  }
}
```

**7. Locals for Computed Values**
```hcl
locals {
  # Computed values used multiple times
  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      ManagedBy   = "Terraform"
      CreatedDate = timestamp()
    }
  )
  
  # Conditional logic
  vpc_cidr = var.vpc_cidr != "" ? var.vpc_cidr : "10.0.0.0/16"
  
  # List transformations
  availability_zones = slice(data.aws_availability_zones.available.names, 0, var.az_count)
}

resource "aws_vpc" "main" {
  cidr_block = local.vpc_cidr
  tags       = local.common_tags
}
```

**Your Drishya Example:**
"At Drishya, I created reusable Terraform modules for:

**VPC Module:**
- Inputs: CIDR, AZ count, NAT gateway (optional)
- Outputs: VPC ID, subnet IDs, route table IDs
- Used across 15+ AWS accounts

**EKS Module:**
- Inputs: K8s version, node instance types, autoscaling config
- Outputs: Cluster endpoint, OIDC provider ARN
- Integrated with VPC module outputs

**Application Module:**
- Composite module: ALB + ECS Fargate + RDS
- Single call deploys entire application stack
- Version pinned to Git tags (e.g., v1.2.3)

**Result:**
- 90% reduction in provisioning errors (consistent configurations)
- 70% faster infrastructure deployment (reusable modules)
- New environment in 1 hour vs 1 day previously"

---

## KUBERNETES

### Q: Explain Kubernetes pod scheduling and how you optimize it

**Answer:**

**Default Scheduling Process:**

```
1. User creates Pod
2. API Server validates and stores in etcd
3. Scheduler watches for unscheduled Pods
4. Scheduler evaluates all nodes (filtering + scoring)
5. Scheduler binds Pod to best node
6. Kubelet on that node starts containers
```

**Filtering Phase (Hard Constraints):**
```yaml
# Scheduler removes nodes that don't meet:
1. Resource requests (CPU, memory)
2. Node selectors
3. Affinity/anti-affinity rules
4. Taints and tolerations
5. Volume availability
```

**Scoring Phase (Soft Preferences):**
```yaml
# Remaining nodes scored based on:
1. Resource balance (least allocated vs balanced)
2. Affinity preferences
3. Spreading (across zones/nodes)
4. Custom plugins
```

**Optimization Techniques:**

**1. Resource Requests and Limits**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:         # Minimum guaranteed
        memory: "256Mi"
        cpu: "250m"     # 0.25 CPU cores
      limits:           # Maximum allowed
        memory: "512Mi"
        cpu: "500m"

# Best Practice: Set requests based on actual usage (P95)
# Limits: 1.5-2x requests (allows bursting)
```

**2. Node Selectors (Simple)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  nodeSelector:
    gpu: "true"           # Only schedule on nodes with this label
    instance-type: "p3.2xlarge"
  containers:
  - name: ml-training
    image: tensorflow
```

**3. Node Affinity (Advanced)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: instance-type
            operator: In
            values:
            - c5.xlarge  # Prefer this type but not required
```

**4. Pod Affinity/Anti-Affinity**
```yaml
# Anti-affinity: Spread pods across nodes/zones
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-app
            topologyKey: kubernetes.io/hostname  # Different nodes
            
# Affinity: Co-locate pods (e.g., app with cache)
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - redis-cache
        topologyKey: kubernetes.io/hostname  # Same node as Redis
```

**5. Taints and Tolerations**
```yaml
# Taint node (repel pods)
kubectl taint nodes node1 key=value:NoSchedule

# Pod tolerates taint (can be scheduled)
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"

# Use Cases:
# - Dedicated nodes for specific workloads (e.g., GPU nodes)
# - Node draining (taint with NoExecute to evict pods)
# - High-priority pods only
```

**6. Priority Classes**
```yaml
# Create priority class
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority workloads"

---
# Use in pod
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  # This pod can preempt (evict) lower-priority pods if needed
```

**7. Topology Spread Constraints**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1              # Max difference between zones
        topologyKey: kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
      # Result: 2 pods in each of 3 zones (balanced)
```

**Your Drishya Example:**
"At Drishya, I optimized pod scheduling for our ML platform:

**Problem:** GPU nodes ($3/hour) were underutilized, CPU pods scheduled randomly

**Solution:**
1. Tainted GPU nodes: `gpu=true:NoSchedule`
2. ML pods tolerate taint + node selector for GPU
3. Implemented pod anti-affinity for API pods (spread across 3 AZs)
4. Set resource requests based on monitoring data (CPU: 250m, Mem: 512Mi)
5. Priority classes: Critical (API) = 1000, Batch (ML training) = 500
6. Topology spread for web tier (maxSkew: 1 across zones)

**Result:**
- GPU utilization: 40% → 85%
- Reduced GPU costs by $2K/month
- Zero single-zone failures (proper spreading)
- High-priority pods never wait for resources"

---

## CI/CD

### Q: Explain your GitHub Actions pipeline optimization

**Answer:**

**Before State:**
```yaml
# Single job, sequential steps
# Total time: ~4 hours

name: Deploy
on: push
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
    - name: Lint (20 min)
    - name: Test (60 min)
    - name: Build (60 min)
    - name: Security Scan (30 min)
    - name: Deploy (10 min)
    - name: Smoke Tests (10 min)
```

**After Optimization:**

**1. Parallel Jobs**
```yaml
name: Optimized Pipeline
on: push

jobs:
  quality-checks:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        check: [lint, security, unit-tests]
    steps:
    - uses: actions/checkout@v3
    - name: Run ${{ matrix.check }}
      run: make ${{ matrix.check }}
    # Runs in parallel: 5 minutes total (was 30 min sequential)

  build:
    needs: quality-checks  # Only after checks pass
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Build Docker Image
      uses: docker/build-push-action@v4
      with:
        cache-from: type=gha
        cache-to: type=gha,mode=max
    # 15 min (was 60 min with caching)

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to ECS
      run: |
        aws ecs update-service --force-new-deployment
    - name: Smoke Tests
      run: make smoke-tests
    # 10 min
```

**2. Docker Build Caching**
```yaml
# Before: 60-minute builds every time
# After: 5-minute builds (cache hit), 15-minute (cache miss)

- name: Build and Push
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE_TAG }}
    cache-from: type=gha          # GitHub Actions cache
    cache-to: type=gha,mode=max   # Store all layers
    build-args: |
      BUILDKIT_INLINE_CACHE=1

# Multi-stage Dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production    # Cached if package.json unchanged
COPY . .
RUN npm run build               # Cached if source unchanged

FROM node:18-slim
COPY --from=builder /app/dist /app
# Final image: 150MB (was 800MB)
```

**3. Quality Gates (Fail Fast)**
```yaml
jobs:
  quality-gates:
    runs-on: ubuntu-latest
    steps:
    - name: Code Coverage
      run: |
        coverage=$(pytest --cov --cov-report=term | grep TOTAL | awk '{print $4}' | sed 's/%//')
        if [ $coverage -lt 80 ]; then
          echo "Coverage $coverage% below 80% threshold"
          exit 1
        fi
    
    - name: Security Scan
      uses: aquasecurity/trivy-action@master
      with:
        severity: 'CRITICAL,HIGH'
        exit-code: '1'  # Fail build on vulnerabilities
    
    - name: Linting
      run: |
        eslint . --max-warnings=0  # No warnings allowed
        
# Catches 92% of issues before deploy
# Fails in 2-3 minutes instead of deploying bad code
```

**4. Reusable Workflows**
```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
jobs:
  deploy:
    steps:
      # Standard deployment steps

# .github/workflows/main.yml
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
  
  deploy-prod:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
```

**5. Conditional Execution**
```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'  # Only deploy from main
    steps:
      - name: Deploy to Staging
        if: github.event_name == 'push'
      
      - name: Deploy to Production
        if: github.event_name == 'release'
```

**6. Self-Hosted Runners (for speed)**
```yaml
# GitHub-hosted: 2-core, 7GB RAM, cold start 30s
# Self-hosted: 8-core, 32GB RAM, warm, Docker cache

jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    steps:
      # Faster builds with persistent Docker layer cache
      # 60 min → 8 min builds
```

**7. Monitoring & Metrics**
```yaml
jobs:
  deploy:
    steps:
      - name: Record Deployment
        run: |
          curl -X POST https://metrics-api/deployments \
            -d '{
              "service": "web-app",
              "version": "${{ github.sha }}",
              "duration": "${{ job.duration }}",
              "status": "success"
            }'
```

**Final Pipeline (30 minutes total):**
```
Trigger (push to main)
  ↓
[Parallel] Quality Checks (5 min)
  ├─ Lint
  ├─ Security Scan
  └─ Unit Tests
  ↓
[Parallel] Build & Test (15 min)
  ├─ Build Docker (with cache)
  └─ Integration Tests
  ↓
Deploy to Staging (5 min)
  ├─ ECS Update
  └─ Smoke Tests
  ↓
[Manual Approval]
  ↓
Deploy to Production (5 min)
  ├─ ECS Update
  └─ Smoke Tests
```

**Your Results:**
- **Time:** 4 hours → 30 minutes (87.5% reduction)
- **Frequency:** 2/week → 10/week (300% increase)
- **Success Rate:** 99.2% (quality gates catch issues early)
- **Rollbacks:** 85% reduction (better testing)
- **Cost:** $200/month GitHub Actions (saved thousands in developer time)

---

**THIS IS PART 1 OF THE TECHNICAL CHEAT SHEET**
**CONTINUED IN NEXT SECTION...**
