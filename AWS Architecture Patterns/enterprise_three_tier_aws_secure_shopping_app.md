# Enterprise Three-Tier Highly Secure Shopping Application on AWS

**Document version:** 1.0  
**Target audience:** Cloud architects, DevOps engineers, SREs, security engineers, platform teams  
**Reference application:** Enterprise shopping/e-commerce platform  
**Alternative mapping:** Streaming/media application notes are included where applicable  
**Deployment options covered:** ECS Fargate primary path, EKS alternative, EC2 Auto Scaling alternative  

---

## 1. Executive Summary

This document explains the design and implementation of an enterprise-grade, highly secure, highly available three-tier application deployed on AWS using cloud-native AWS services.

The reference workload is a **shopping application** similar to a production e-commerce platform. It includes web/mobile user access, catalog browsing, search, cart, checkout, payment orchestration, order processing, notifications, observability, security controls, caching, scaling, and disaster recovery.

The architecture follows AWS Well-Architected principles across reliability, security, operational excellence, performance efficiency, cost optimization, and sustainability.

### Main design goals

- High availability across multiple Availability Zones.
- Strong security using least privilege, private networking, encryption, WAF, IAM, Secrets Manager, and CloudTrail.
- Low-latency customer experience using CloudFront, ElastiCache, ALB, service-level scaling, and database read replicas.
- Scalable compute using ECS Fargate as the primary deployment model.
- Optional deployment patterns for Amazon EKS and EC2 Auto Scaling.
- Reliable data layer using Aurora, DynamoDB, S3, SQS, EventBridge, and backup strategies.
- Enterprise observability using CloudWatch, X-Ray, CloudTrail, VPC Flow Logs, GuardDuty, Security Hub, and AWS Config.
- Clear understanding of CAP theorem tradeoffs, latency, partition tolerance, caching, and consistency.

---

## 2. Application Overview

### 2.1 Business use case

The application supports these features:

- Customer registration and login.
- Product catalog browsing.
- Product search and filtering.
- Shopping cart management.
- Checkout and order placement.
- Payment initiation with external payment provider.
- Inventory reservation.
- Order confirmation.
- Email/SMS notification.
- Admin portal for catalog, pricing, inventory, promotions, and reporting.

### 2.2 Functional modules

| Module | Responsibility | Example AWS Services |
|---|---|---|
| Web frontend | Customer web interface | CloudFront, S3, AWS Amplify optional |
| API layer | Public API endpoint | ALB, API Gateway optional, ECS/EKS/EC2 |
| Auth | User authentication and authorization | Amazon Cognito, IAM |
| Catalog service | Product listing and product detail APIs | ECS, DynamoDB, OpenSearch optional |
| Cart service | Cart create/update/delete | ECS, ElastiCache, DynamoDB |
| Order service | Order creation and lifecycle | ECS, Aurora PostgreSQL/MySQL |
| Payment service | Payment integration | ECS, Secrets Manager, EventBridge |
| Inventory service | Stock reservation and update | ECS, Aurora/DynamoDB |
| Notification service | Email/SMS events | SNS, SES, SQS, Lambda |
| Admin service | Back-office APIs | ECS, ALB internal, Cognito groups |
| Observability | Logs, metrics, traces, audit | CloudWatch, X-Ray, CloudTrail |
| Security | Threat detection and compliance | WAF, Shield, GuardDuty, Security Hub, AWS Config |

---

## 3. Three-Tier Architecture

### 3.1 High-level architecture

```text
Users / Mobile Apps / Browsers
        |
        v
Amazon Route 53
        |
        v
Amazon CloudFront + AWS WAF + Shield
        |
        +-----------------------------+
        |                             |
        v                             v
Static Web Assets                  API Requests
Amazon S3                         Application Load Balancer
                                      |
                                      v
                         Private Subnets: Application Tier
                         ECS Fargate Services / EKS / EC2 ASG
                         - Web/API service
                         - Catalog service
                         - Cart service
                         - Order service
                         - Payment service
                         - Inventory service
                         - Notification worker
                                      |
              +-----------------------+-----------------------+
              |                       |                       |
              v                       v                       v
        ElastiCache Redis        Aurora Multi-AZ          DynamoDB
        caching/session/cart     orders/payments          catalog/cart/session
              |                       |                       |
              +-----------------------+-----------------------+
                                      |
                                      v
                           Async Event Backbone
                         SQS + SNS + EventBridge
                                      |
                                      v
                          Observability and Security
      CloudWatch, X-Ray, CloudTrail, VPC Flow Logs, GuardDuty, Security Hub
```

### 3.2 Three tiers explained

#### Tier 1: Presentation tier

The presentation tier provides customer-facing access.

**AWS services used:**

- **Route 53:** DNS routing and health checks.
- **CloudFront:** Global content delivery and edge caching.
- **S3:** Static website assets such as HTML, CSS, JavaScript, images, and product media.
- **AWS WAF:** Layer 7 protection for SQL injection, XSS, bot control, rate limiting, and managed rules.
- **AWS Shield:** DDoS protection.
- **ACM:** TLS certificates for HTTPS.

**Why this design:**

- Static content is cached close to users.
- WAF blocks malicious requests before they reach the application.
- CloudFront improves latency by serving objects from edge locations.
- CloudFront origin access control prevents direct public access to S3.

#### Tier 2: Application tier

The application tier contains API services and business logic.

**Primary deployment option:** Amazon ECS with Fargate.

**Alternative options:**

- Amazon EKS for Kubernetes-based organizations.
- EC2 Auto Scaling Groups for VM-based application hosting.

**AWS services used:**

- **Application Load Balancer:** Routes HTTP/HTTPS traffic to services.
- **ECS Fargate:** Serverless container runtime.
- **ECR:** Stores container images.
- **Cloud Map:** Optional service discovery.
- **App Mesh:** Optional service mesh for advanced traffic control.
- **Secrets Manager:** Stores database passwords, API tokens, and third-party payment credentials.
- **Parameter Store:** Stores non-secret configuration.
- **IAM roles for tasks:** Grants fine-grained AWS permissions to each service.

#### Tier 3: Data tier

The data tier stores persistent and temporary application data.

**AWS services used:**

- **Aurora PostgreSQL or Aurora MySQL:** Relational order, payment, inventory, and transaction data.
- **DynamoDB:** Product catalog, cart, session, idempotency records, and high-scale key-value workloads.
- **ElastiCache Redis:** Low-latency caching, session cache, cart cache, rate-limit counters, and distributed locks where appropriate.
- **S3:** Product images, invoices, exports, logs, and backups.
- **OpenSearch Service:** Optional product search and analytics.
- **SQS:** Reliable background processing.
- **SNS:** Fan-out notifications.
- **EventBridge:** Event-driven integrations.

---

## 4. AWS Native Services Used

### 4.1 Networking

| Requirement | AWS Service | Purpose |
|---|---|---|
| Isolated cloud network | VPC | Private application network |
| Public access layer | Public subnets | ALB, NAT Gateway, bastion optional |
| Private compute | Private subnets | ECS/EKS/EC2 application tasks |
| Private data layer | Isolated/private subnets | Aurora, ElastiCache |
| Outbound internet from private subnets | NAT Gateway | Patch, package, external API access |
| Private AWS API access | VPC endpoints | Private access to S3, ECR, CloudWatch, Secrets Manager |
| Network filtering | Security groups and NACLs | Restrict traffic |
| Traffic inspection optional | AWS Network Firewall | Advanced egress/ingress inspection |

### 4.2 Security

| Requirement | AWS Service | Purpose |
|---|---|---|
| Identity and access | IAM | Least privilege access |
| Customer identity | Cognito | User pools, tokens, federation |
| Secrets | Secrets Manager | Rotate and retrieve secrets securely |
| Encryption keys | KMS | Encrypt data at rest and manage keys |
| Web protection | WAF | Block malicious HTTP requests |
| DDoS protection | Shield | DDoS defense |
| Audit logs | CloudTrail | Record AWS API activity |
| Threat detection | GuardDuty | Detect suspicious activity |
| Security posture | Security Hub | Central security findings |
| Config compliance | AWS Config | Track resource configuration |
| Vulnerability scan | Inspector | Scan workloads and images |
| Private connectivity | PrivateLink/VPC endpoints | Reduce public internet exposure |

### 4.3 Compute

| Deployment model | Best for | AWS Services |
|---|---|---|
| ECS Fargate | Enterprise container apps without managing servers | ECS, Fargate, ECR, ALB |
| EKS | Kubernetes standardization, platform engineering | EKS, managed node groups/Fargate, ALB Controller |
| EC2 Auto Scaling | Legacy apps, full OS control, custom agents | EC2, Launch Templates, Auto Scaling Groups |

### 4.4 Data and messaging

| Data type | AWS Service | Example |
|---|---|---|
| Orders and payments | Aurora | ACID transactions |
| Product catalog | DynamoDB | High-scale product reads |
| Cart/session | DynamoDB and Redis | Fast cart access |
| Search index | OpenSearch | Text search and filters |
| Static media | S3 | Product images and invoices |
| Events | EventBridge | OrderCreated, PaymentAuthorized |
| Queue | SQS | Reliable async workers |
| Notification | SNS/SES | Email/SMS fan-out |

### 4.5 Observability

| Requirement | AWS Service |
|---|---|
| Application logs | CloudWatch Logs |
| Metrics and alarms | CloudWatch Metrics and Alarms |
| Distributed tracing | AWS X-Ray / OpenTelemetry |
| API and infrastructure audit | CloudTrail |
| Network inspection logs | VPC Flow Logs |
| Dashboarding | CloudWatch Dashboards / Managed Grafana |
| Container insights | CloudWatch Container Insights |

---

## 5. Production Reference Architecture

### 5.1 Recommended production topology

Use at least three Availability Zones in a production Region.

```text
Region: ap-south-1 / us-east-1 / eu-west-1

VPC: 10.0.0.0/16

AZ-a:
  Public subnet: 10.0.1.0/24
  Private app subnet: 10.0.11.0/24
  Private data subnet: 10.0.21.0/24

AZ-b:
  Public subnet: 10.0.2.0/24
  Private app subnet: 10.0.12.0/24
  Private data subnet: 10.0.22.0/24

AZ-c:
  Public subnet: 10.0.3.0/24
  Private app subnet: 10.0.13.0/24
  Private data subnet: 10.0.23.0/24
```

### 5.2 Recommended traffic flow

1. User resolves domain through Route 53.
2. User reaches CloudFront over HTTPS.
3. CloudFront serves static content from S3.
4. API requests go through CloudFront and WAF to ALB.
5. ALB forwards requests to ECS Fargate services in private subnets.
6. Services retrieve secrets from Secrets Manager using task IAM roles.
7. Services access Aurora, DynamoDB, ElastiCache, S3, SQS, SNS, and EventBridge.
8. Logs, metrics, traces, and audit events are sent to CloudWatch, X-Ray, CloudTrail, and security services.

---

## 6. System Design Explanation

### 6.1 Request flow: product browsing

```text
User -> CloudFront -> S3 static assets
User -> CloudFront -> WAF -> ALB -> Catalog Service -> Redis Cache -> DynamoDB/OpenSearch
```

Flow:

1. User opens application homepage.
2. CloudFront serves frontend files from S3.
3. Frontend calls `/products` API.
4. WAF validates and filters request.
5. ALB forwards to Catalog service.
6. Catalog service checks Redis cache.
7. If cache hit, response is returned quickly.
8. If cache miss, service reads DynamoDB or OpenSearch and updates Redis.

### 6.2 Request flow: cart update

```text
User -> ALB -> Cart Service -> Redis -> DynamoDB
```

Flow:

1. User adds product to cart.
2. Cart service writes to Redis for low-latency access.
3. Cart service persists cart state to DynamoDB.
4. DynamoDB TTL can expire abandoned carts.
5. Cart updates are idempotent using request IDs.

### 6.3 Request flow: checkout and order placement

```text
User -> ALB -> Checkout Service
             -> Order Service -> Aurora transaction
             -> Inventory Service -> reserve stock
             -> Payment Service -> payment provider
             -> EventBridge -> SQS -> Notification Worker
```

Flow:

1. User submits checkout request.
2. Checkout service validates cart and customer identity.
3. Order service creates order in Aurora.
4. Inventory service reserves stock.
5. Payment service calls payment provider using secure credentials from Secrets Manager.
6. OrderCreated event is published to EventBridge.
7. EventBridge routes to SQS queues.
8. Notification worker sends confirmation through SES/SNS.

### 6.4 Request flow: admin update product price

```text
Admin -> Cognito Group -> CloudFront/WAF -> Internal Admin API -> Catalog Service -> DynamoDB -> EventBridge -> Cache invalidation
```

Flow:

1. Admin authenticates with Cognito.
2. API authorizes admin role/group.
3. Product data is updated in DynamoDB.
4. ProductUpdated event is emitted.
5. Cache invalidation job clears Redis and optionally CloudFront cache.

---

## 7. High Availability Design

### 7.1 HA principles

- Deploy across at least two Availability Zones; three AZs are preferred for enterprise production.
- Keep compute stateless where possible.
- Store state in managed multi-AZ data services.
- Use health checks at every layer.
- Use autoscaling for compute and data throughput.
- Use queues to absorb traffic spikes.
- Prefer managed AWS services for control-plane availability.
- Use backup, restore, and disaster recovery testing.

### 7.2 HA by layer

| Layer | HA strategy |
|---|---|
| DNS | Route 53 health checks and failover routing |
| Edge | CloudFront global edge network |
| Web security | AWS WAF attached to CloudFront/ALB |
| Load balancing | ALB across multiple AZs |
| Compute | ECS service with tasks spread across private subnets in multiple AZs |
| Database | Aurora Multi-AZ cluster with reader endpoints |
| NoSQL | DynamoDB on-demand or provisioned autoscaling; optional global tables |
| Cache | ElastiCache Redis with Multi-AZ and automatic failover |
| Queue | SQS managed availability and DLQ |
| Object storage | S3 with versioning, lifecycle, replication optional |
| Observability | CloudWatch alarms and dashboards |

### 7.3 Multi-Region DR options

| DR pattern | RTO | RPO | Cost | Example |
|---|---:|---:|---:|---|
| Backup and restore | Hours | Hours | Low | Nightly Aurora snapshots |
| Pilot light | 30-120 min | Minutes-hours | Medium | Minimal infra in DR Region |
| Warm standby | 5-30 min | Seconds-minutes | Higher | Scaled-down active DR stack |
| Active-active | Near zero | Near zero / eventual | Highest | Global DynamoDB, Aurora Global Database, Route 53 ARC |

For most enterprise shopping apps, start with **multi-AZ production** plus **warm standby DR** for critical services. Use active-active only when the business requires regional fault isolation and global low latency.

---

## 8. CAP Theorem, Consistency, Latency, and Partition Tolerance

### 8.1 CAP theorem basics

CAP theorem says that during a network partition, a distributed system must choose between:

- **Consistency:** Every read gets the latest write or an error.
- **Availability:** Every request receives a non-error response, though it may not contain the latest data.
- **Partition tolerance:** The system continues operating despite network communication failure between nodes.

In practical cloud architecture, partition tolerance is mandatory. Therefore, each subsystem must intentionally choose consistency or availability during failures.

### 8.2 CAP decisions in this shopping app

| Component | Preferred CAP behavior | Reason |
|---|---|---|
| Product catalog | Availability + partition tolerance | Slightly stale product info is acceptable |
| Product price | Consistency preferred for checkout | Incorrect price can cause financial/compliance issues |
| Cart | Availability + eventual consistency | User experience matters; can reconcile at checkout |
| Inventory | Consistency preferred | Overselling must be minimized |
| Order creation | Consistency preferred | Duplicate/incorrect orders are harmful |
| Payment | Strong consistency and idempotency | Financial correctness is critical |
| Search | Availability + eventual consistency | Search index can lag source of truth |
| Analytics | Availability + eventual consistency | Real-time exactness usually not required |
| Notifications | Availability + retry | Duplicate-safe notifications are acceptable with idempotency |

### 8.3 Latency design

| Technique | Purpose |
|---|---|
| CloudFront | Reduce global static/dynamic content latency |
| Redis cache | Reduce repeated database reads |
| DynamoDB | Single-digit millisecond key-value access pattern when modeled correctly |
| Aurora read replicas | Scale read queries away from writer |
| SQS async processing | Keep user-facing requests fast |
| EventBridge events | Decouple services and avoid synchronous chains |
| Connection pooling | Reduce DB connection overhead |
| Regional deployment | Keep compute and data close to users |
| OpenSearch | Fast product search and filters |

### 8.4 Partition tolerance strategy

During partial failures:

- Product catalog continues serving cached data.
- Cart service accepts writes with idempotency and retries persistence.
- Checkout validates price, inventory, and payment using source-of-truth stores.
- Orders use a transactional database and idempotency keys.
- Queues buffer work until downstream services recover.
- DLQs capture failed async messages for replay.
- Circuit breakers stop cascading failures.
- Bulkheads isolate service failure domains.

---

## 9. Security Architecture

### 9.1 Defense in depth

Security controls are applied at every layer:

```text
User -> TLS -> CloudFront -> WAF/Shield -> ALB -> Private Services -> Private Data Stores
                                  |             |                  |
                              CloudTrail     IAM Roles       KMS Encryption
                              GuardDuty      Secrets         VPC Endpoints
                              Config         Logging         SG Rules
```

### 9.2 Identity and access management

Best practices:

- Use IAM roles, not long-term access keys.
- Use IAM Identity Center for workforce access.
- Use least privilege policies for ECS task roles.
- Separate roles per microservice.
- Use permission boundaries and SCPs in AWS Organizations.
- Enforce MFA for privileged users.
- Disable root user access keys.
- Use break-glass accounts with alerting.

### 9.3 Network security

Recommended controls:

- ALB in public subnets only.
- ECS/EKS/EC2 application workloads in private subnets.
- Databases and cache in isolated private subnets.
- Security groups allow only required ports.
- No direct public SSH/RDP to instances.
- Use Session Manager instead of bastion hosts where possible.
- Use VPC endpoints for AWS services.
- Use NAT Gateway egress restrictions and optionally AWS Network Firewall.

### 9.4 Data protection

| Data | Protection |
|---|---|
| S3 objects | SSE-KMS, bucket policy, block public access, versioning |
| Aurora | KMS encryption, TLS, Secrets Manager rotation |
| DynamoDB | KMS encryption, IAM fine-grained access |
| ElastiCache | Encryption in transit and at rest, auth token |
| EBS | KMS encryption |
| Logs | KMS encryption and retention policies |
| Secrets | Secrets Manager with rotation |

### 9.5 Application security

- Validate all API inputs.
- Use OAuth2/OIDC through Cognito or enterprise IdP.
- Implement JWT validation and scopes.
- Use idempotency keys for payment and order APIs.
- Use rate limiting in WAF and application layer.
- Avoid secrets in environment variables where possible; use runtime secret injection carefully.
- Scan images in ECR and Inspector.
- Use signed images and CI/CD security gates.
- Use dependency scanning.
- Use strict CORS policies.

### 9.6 Logging and audit

Required logs:

- CloudTrail management events.
- CloudTrail data events for sensitive S3 buckets and Lambda if used.
- ALB access logs.
- CloudFront logs.
- WAF logs.
- VPC Flow Logs.
- ECS application logs.
- Aurora audit logs where applicable.
- GuardDuty findings.
- AWS Config compliance history.

---

## 10. Caching Strategy

### 10.1 Cache layers

| Layer | Cache type | Example |
|---|---|---|
| Browser | Client cache | Static assets |
| CloudFront | CDN cache | Images, JS, CSS, API GET responses |
| Application | In-memory cache | Feature flags, local config |
| Redis | Distributed cache | Catalog, sessions, carts, rate limits |
| Database | Read replicas/query cache equivalent | Aurora reader endpoint |

### 10.2 Cache patterns

#### Cache-aside

Application checks Redis first. On miss, it reads database and writes Redis.

Best for:

- Product details.
- Category pages.
- Promotion metadata.

#### Write-through

Application writes cache and database together.

Best for:

- Session metadata.
- Frequently accessed customer preferences.

#### Write-behind

Application writes cache first and asynchronously persists data.

Use carefully. Not recommended for payments or orders.

#### Cache invalidation

Use events:

```text
ProductUpdated -> EventBridge -> CacheInvalidationWorker -> Redis delete + CloudFront invalidation
```

### 10.3 TTL recommendations

| Data type | TTL |
|---|---:|
| Static assets | 1 day to 1 year with versioned filenames |
| Product catalog | 5-30 minutes |
| Product detail | 1-10 minutes |
| Cart cache | 15-60 minutes |
| Session cache | Depends on token/session design |
| Inventory count | Very short TTL or no cache at checkout |
| Price at checkout | Validate from source of truth |

---

## 11. Scaling Strategy

### 11.1 Compute scaling

For ECS Fargate:

- Scale services by CPU utilization.
- Scale services by memory utilization.
- Scale API services by ALB request count per target.
- Scale worker services by SQS queue depth.
- Use minimum task count across multiple AZs.

Example:

| Service | Min tasks | Max tasks | Scaling signal |
|---|---:|---:|---|
| Web/API | 6 | 100 | ALB request count/CPU |
| Catalog | 6 | 200 | CPU/cache miss rate |
| Cart | 6 | 100 | CPU/DynamoDB latency |
| Order | 6 | 80 | CPU/Aurora latency |
| Worker | 3 | 200 | SQS queue depth |

### 11.2 Database scaling

Aurora:

- Use writer instance for transactions.
- Use reader endpoint for read-only queries.
- Use Aurora replicas for read scaling.
- Use RDS Proxy for connection pooling.
- Use indexes and query optimization.
- Use partitioning/sharding only when required.

DynamoDB:

- Use on-demand for unpredictable traffic.
- Use provisioned capacity with autoscaling for stable workloads.
- Design partition keys carefully.
- Avoid hot partitions.
- Use GSIs for query patterns.
- Use TTL for temporary data.

ElastiCache:

- Enable cluster mode for horizontal scaling.
- Use Multi-AZ automatic failover.
- Monitor memory, evictions, CPU, and replication lag.

---

## 12. Deployment Option 1: ECS Fargate Recommended Architecture

### 12.1 Why ECS Fargate

ECS Fargate is recommended for many enterprise teams because it runs containers without managing EC2 worker nodes. It reduces operational overhead while supporting autoscaling, IAM roles for tasks, ALB integration, private networking, and CloudWatch logging.

### 12.2 ECS components

| Component | Purpose |
|---|---|
| ECS cluster | Logical grouping of services |
| Task definition | Container specification |
| ECS service | Desired running task count |
| Fargate capacity | Serverless container compute |
| ECR repository | Container image registry |
| Task execution role | Pull images and write logs |
| Task role | Application AWS permissions |
| ALB target group | Routes traffic to tasks |
| CloudWatch logs | Container logs |

### 12.3 ECS production design

- One ECS cluster per environment or workload domain.
- Separate ECS services per microservice.
- Private subnets only for ECS tasks.
- ALB target type `ip`.
- Health checks configured at container and ALB level.
- Service autoscaling enabled.
- Blue/green deployment using CodeDeploy or rolling deployment with circuit breaker.
- ECR image scanning enabled.
- CloudWatch Container Insights enabled.

---

## 13. Deployment Option 2: EKS Architecture

Use EKS when your organization standardizes on Kubernetes or requires Kubernetes-native tooling.

### 13.1 EKS components

| Component | Purpose |
|---|---|
| EKS cluster | Managed Kubernetes control plane |
| Managed node groups or Fargate profiles | Worker compute |
| AWS Load Balancer Controller | Creates ALB/NLB from Kubernetes ingress/service |
| IRSA / Pod Identity | Fine-grained IAM access for pods |
| ECR | Container registry |
| CloudWatch Container Insights | Logs and metrics |
| Karpenter | Node autoscaling optional |

### 13.2 EKS security practices

- Private cluster endpoint where possible.
- Use Kubernetes RBAC with least privilege.
- Use IRSA or EKS Pod Identity for AWS permissions.
- Use network policies.
- Use image scanning and admission controls.
- Store secrets in Secrets Manager or External Secrets Operator.
- Use managed node groups or Fargate for reduced operations.

---

## 14. Deployment Option 3: EC2 Auto Scaling Architecture

Use EC2 when you require full host control, custom agents, legacy apps, or specialized runtime dependencies.

### 14.1 EC2 components

| Component | Purpose |
|---|---|
| Launch template | EC2 AMI, instance type, security groups, user data |
| Auto Scaling Group | Maintains desired instance capacity |
| ALB target group | Routes traffic to EC2 instances |
| Systems Manager | Patch, session access, inventory |
| CloudWatch Agent | OS and app metrics/logs |
| IAM instance profile | AWS permissions for instances |

### 14.2 EC2 security practices

- No public IPs on application instances.
- Use Session Manager instead of SSH.
- Patch through Systems Manager Patch Manager.
- Use IMDSv2 only.
- Use encrypted EBS volumes.
- Use hardened AMIs.
- Use Auto Scaling health checks.

---

## 15. Step-by-Step Implementation Guide Using AWS CLI

> Notes:
> - Replace example values before running commands.
> - The commands use `us-east-1` as an example Region.
> - Production infrastructure should usually be deployed through IaC such as AWS CDK, Terraform, or CloudFormation. CLI commands are included for learning and implementation clarity.

### 15.1 Configure environment variables

```bash
export AWS_REGION="us-east-1"
export APP_NAME="enterprise-shop"
export ENV="prod"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VPC_CIDR="10.0.0.0/16"
```

### 15.2 Create VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$APP_NAME-$ENV-vpc}]" \
  --query 'Vpc.VpcId' \
  --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

echo $VPC_ID
```

### 15.3 Create subnets across three AZs

```bash
AZ1="${AWS_REGION}a"
AZ2="${AWS_REGION}b"
AZ3="${AWS_REGION}c"

PUB_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone $AZ1 --query 'Subnet.SubnetId' --output text)
PUB_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone $AZ2 --query 'Subnet.SubnetId' --output text)
PUB_SUBNET_3=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone $AZ3 --query 'Subnet.SubnetId' --output text)

APP_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone $AZ1 --query 'Subnet.SubnetId' --output text)
APP_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone $AZ2 --query 'Subnet.SubnetId' --output text)
APP_SUBNET_3=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.13.0/24 --availability-zone $AZ3 --query 'Subnet.SubnetId' --output text)

DATA_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.21.0/24 --availability-zone $AZ1 --query 'Subnet.SubnetId' --output text)
DATA_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.22.0/24 --availability-zone $AZ2 --query 'Subnet.SubnetId' --output text)
DATA_SUBNET_3=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.23.0/24 --availability-zone $AZ3 --query 'Subnet.SubnetId' --output text)
```

### 15.4 Create internet gateway and public route table

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

PUBLIC_RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUBLIC_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUBLIC_RT_ID --subnet-id $PUB_SUBNET_1
aws ec2 associate-route-table --route-table-id $PUBLIC_RT_ID --subnet-id $PUB_SUBNET_2
aws ec2 associate-route-table --route-table-id $PUBLIC_RT_ID --subnet-id $PUB_SUBNET_3
```

### 15.5 Create NAT Gateways for private outbound access

For production, deploy one NAT Gateway per AZ.

```bash
EIP_ALLOC_1=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_GW_1=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_1 \
  --allocation-id $EIP_ALLOC_1 \
  --query 'NatGateway.NatGatewayId' \
  --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_1

PRIVATE_RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PRIVATE_RT_ID --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_1

aws ec2 associate-route-table --route-table-id $PRIVATE_RT_ID --subnet-id $APP_SUBNET_1
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_ID --subnet-id $APP_SUBNET_2
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_ID --subnet-id $APP_SUBNET_3
```

### 15.6 Create security groups

```bash
ALB_SG=$(aws ec2 create-security-group \
  --group-name $APP_NAME-$ENV-alb-sg \
  --description "ALB security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

APP_SG=$(aws ec2 create-security-group \
  --group-name $APP_NAME-$ENV-app-sg \
  --description "Application task security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

DB_SG=$(aws ec2 create-security-group \
  --group-name $APP_NAME-$ENV-db-sg \
  --description "Database security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

CACHE_SG=$(aws ec2 create-security-group \
  --group-name $APP_NAME-$ENV-cache-sg \
  --description "Cache security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 8080 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 5432 --source-group $APP_SG
aws ec2 authorize-security-group-ingress --group-id $CACHE_SG --protocol tcp --port 6379 --source-group $APP_SG
```

### 15.7 Create KMS key

```bash
KMS_KEY_ID=$(aws kms create-key \
  --description "$APP_NAME $ENV application encryption key" \
  --query 'KeyMetadata.KeyId' \
  --output text)

aws kms create-alias \
  --alias-name alias/$APP_NAME-$ENV-key \
  --target-key-id $KMS_KEY_ID
```

### 15.8 Create S3 buckets for static assets and logs

```bash
STATIC_BUCKET="$APP_NAME-$ENV-static-$ACCOUNT_ID"
LOG_BUCKET="$APP_NAME-$ENV-logs-$ACCOUNT_ID"

aws s3api create-bucket --bucket $STATIC_BUCKET --region $AWS_REGION
aws s3api create-bucket --bucket $LOG_BUCKET --region $AWS_REGION

aws s3api put-public-access-block \
  --bucket $STATIC_BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-public-access-block \
  --bucket $LOG_BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-versioning \
  --bucket $STATIC_BUCKET \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket $STATIC_BUCKET \
  --server-side-encryption-configuration "{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"aws:kms\",\"KMSMasterKeyID\":\"$KMS_KEY_ID\"}}]}"
```

### 15.9 Create Aurora subnet group

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name $APP_NAME-$ENV-db-subnet-group \
  --db-subnet-group-description "DB subnet group for $APP_NAME $ENV" \
  --subnet-ids $DATA_SUBNET_1 $DATA_SUBNET_2 $DATA_SUBNET_3
```

### 15.10 Store database credentials in Secrets Manager

```bash
DB_SECRET_ARN=$(aws secretsmanager create-secret \
  --name $APP_NAME/$ENV/aurora \
  --kms-key-id $KMS_KEY_ID \
  --secret-string '{"username":"app_user","password":"REPLACE_WITH_STRONG_PASSWORD"}' \
  --query 'ARN' \
  --output text)
```

### 15.11 Create Aurora PostgreSQL cluster

```bash
aws rds create-db-cluster \
  --db-cluster-identifier $APP_NAME-$ENV-aurora \
  --engine aurora-postgresql \
  --engine-version 16.4 \
  --master-username app_user \
  --master-user-password 'REPLACE_WITH_STRONG_PASSWORD' \
  --db-subnet-group-name $APP_NAME-$ENV-db-subnet-group \
  --vpc-security-group-ids $DB_SG \
  --storage-encrypted \
  --kms-key-id $KMS_KEY_ID \
  --backup-retention-period 7 \
  --deletion-protection

aws rds create-db-instance \
  --db-instance-identifier $APP_NAME-$ENV-aurora-writer-1 \
  --db-cluster-identifier $APP_NAME-$ENV-aurora \
  --engine aurora-postgresql \
  --db-instance-class db.r7g.large

aws rds create-db-instance \
  --db-instance-identifier $APP_NAME-$ENV-aurora-reader-1 \
  --db-cluster-identifier $APP_NAME-$ENV-aurora \
  --engine aurora-postgresql \
  --db-instance-class db.r7g.large
```

### 15.12 Create DynamoDB tables

```bash
aws dynamodb create-table \
  --table-name $APP_NAME-$ENV-catalog \
  --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=$KMS_KEY_ID

aws dynamodb create-table \
  --table-name $APP_NAME-$ENV-cart \
  --attribute-definitions AttributeName=CustomerId,AttributeType=S AttributeName=CartId,AttributeType=S \
  --key-schema AttributeName=CustomerId,KeyType=HASH AttributeName=CartId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=$KMS_KEY_ID
```

### 15.13 Create ElastiCache Redis subnet group and replication group

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name $APP_NAME-$ENV-cache-subnet-group \
  --cache-subnet-group-description "Cache subnet group" \
  --subnet-ids $DATA_SUBNET_1 $DATA_SUBNET_2 $DATA_SUBNET_3

aws elasticache create-replication-group \
  --replication-group-id $APP_NAME-$ENV-redis \
  --replication-group-description "Redis cache for $APP_NAME $ENV" \
  --engine redis \
  --cache-node-type cache.r7g.large \
  --num-cache-clusters 3 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --cache-subnet-group-name $APP_NAME-$ENV-cache-subnet-group \
  --security-group-ids $CACHE_SG \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled
```

### 15.14 Create SQS queues and DLQ

```bash
DLQ_URL=$(aws sqs create-queue \
  --queue-name $APP_NAME-$ENV-order-dlq \
  --attributes KmsMasterKeyId=alias/$APP_NAME-$ENV-key \
  --query 'QueueUrl' \
  --output text)

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url $DLQ_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

QUEUE_URL=$(aws sqs create-queue \
  --queue-name $APP_NAME-$ENV-order-events \
  --attributes "KmsMasterKeyId=alias/$APP_NAME-$ENV-key,RedrivePolicy={\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"5\"}" \
  --query 'QueueUrl' \
  --output text)
```

### 15.15 Create EventBridge bus

```bash
aws events create-event-bus --name $APP_NAME-$ENV-events
```

### 15.16 Create ECR repositories

```bash
aws ecr create-repository --repository-name $APP_NAME/api --image-scanning-configuration scanOnPush=true
aws ecr create-repository --repository-name $APP_NAME/catalog --image-scanning-configuration scanOnPush=true
aws ecr create-repository --repository-name $APP_NAME/order --image-scanning-configuration scanOnPush=true
aws ecr create-repository --repository-name $APP_NAME/worker --image-scanning-configuration scanOnPush=true
```

### 15.17 Build and push container image

```bash
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

docker build -t $APP_NAME/api:latest ./api

docker tag $APP_NAME/api:latest \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$APP_NAME/api:latest

docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$APP_NAME/api:latest
```

### 15.18 Create ECS cluster

```bash
aws ecs create-cluster \
  --cluster-name $APP_NAME-$ENV-cluster \
  --settings name=containerInsights,value=enabled
```

### 15.19 Create IAM roles for ECS

Create `ecs-task-execution-trust.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name $APP_NAME-$ENV-ecs-execution-role \
  --assume-role-policy-document file://ecs-task-execution-trust.json

aws iam attach-role-policy \
  --role-name $APP_NAME-$ENV-ecs-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam create-role \
  --role-name $APP_NAME-$ENV-api-task-role \
  --assume-role-policy-document file://ecs-task-execution-trust.json
```

Attach least-privilege policies to the task role for DynamoDB, Secrets Manager, SQS, EventBridge, and CloudWatch as needed.

### 15.20 Create CloudWatch log group

```bash
aws logs create-log-group --log-group-name /ecs/$APP_NAME/$ENV/api
aws logs put-retention-policy --log-group-name /ecs/$APP_NAME/$ENV/api --retention-in-days 90
```

### 15.21 Register ECS task definition

Create `api-taskdef.json`:

```json
{
  "family": "enterprise-shop-prod-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/enterprise-shop-prod-ecs-execution-role",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/enterprise-shop-prod-api-task-role",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/enterprise-shop/api:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        {"name": "ENV", "value": "prod"},
        {"name": "AWS_REGION", "value": "us-east-1"}
      ],
      "secrets": [
        {
          "name": "DB_SECRET_ARN",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:enterprise-shop/prod/aurora"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/enterprise-shop/prod/api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

Replace placeholders:

```bash
sed -i "s/ACCOUNT_ID/$ACCOUNT_ID/g" api-taskdef.json

aws ecs register-task-definition --cli-input-json file://api-taskdef.json
```

### 15.22 Create ALB and target group

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name $APP_NAME-$ENV-alb \
  --subnets $PUB_SUBNET_1 $PUB_SUBNET_2 $PUB_SUBNET_3 \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

TG_ARN=$(aws elbv2 create-target-group \
  --name $APP_NAME-$ENV-api-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)
```

Create HTTPS listener after requesting an ACM certificate.

```bash
CERT_ARN="REPLACE_WITH_ACM_CERTIFICATE_ARN"

aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### 15.23 Create ECS service

```bash
aws ecs create-service \
  --cluster $APP_NAME-$ENV-cluster \
  --service-name api \
  --task-definition enterprise-shop-prod-api \
  --desired-count 6 \
  --launch-type FARGATE \
  --platform-version LATEST \
  --network-configuration "awsvpcConfiguration={subnets=[$APP_SUBNET_1,$APP_SUBNET_2,$APP_SUBNET_3],securityGroups=[$APP_SG],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=api,containerPort=8080" \
  --health-check-grace-period-seconds 120 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}"
```

### 15.24 Configure ECS autoscaling

```bash
SERVICE_NAMESPACE="ecs"
RESOURCE_ID="service/$APP_NAME-$ENV-cluster/api"
SCALABLE_DIMENSION="ecs:service:DesiredCount"

aws application-autoscaling register-scalable-target \
  --service-namespace $SERVICE_NAMESPACE \
  --resource-id $RESOURCE_ID \
  --scalable-dimension $SCALABLE_DIMENSION \
  --min-capacity 6 \
  --max-capacity 100

aws application-autoscaling put-scaling-policy \
  --service-namespace $SERVICE_NAMESPACE \
  --resource-id $RESOURCE_ID \
  --scalable-dimension $SCALABLE_DIMENSION \
  --policy-name api-cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{"TargetValue":60.0,"PredefinedMetricSpecification":{"PredefinedMetricType":"ECSServiceAverageCPUUtilization"},"ScaleInCooldown":120,"ScaleOutCooldown":60}'
```

### 15.25 Create WAF Web ACL

Create `waf-web-acl.json`:

```json
{
  "Name": "enterprise-shop-prod-web-acl",
  "Scope": "REGIONAL",
  "DefaultAction": {"Allow": {}},
  "Description": "WAF for enterprise shopping app",
  "Rules": [
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSet"
      }
    },
    {
      "Name": "AWSManagedRulesSQLiRuleSet",
      "Priority": 2,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesSQLiRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "SQLiRuleSet"
      }
    }
  ],
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "enterprise-shop-prod-web-acl"
  }
}
```

```bash
WEB_ACL_ARN=$(aws wafv2 create-web-acl \
  --cli-input-json file://waf-web-acl.json \
  --query 'Summary.ARN' \
  --output text)

aws wafv2 associate-web-acl \
  --web-acl-arn $WEB_ACL_ARN \
  --resource-arn $ALB_ARN
```

### 15.26 Enable CloudTrail

```bash
TRAIL_BUCKET="$APP_NAME-$ENV-cloudtrail-$ACCOUNT_ID"
aws s3api create-bucket --bucket $TRAIL_BUCKET --region $AWS_REGION

aws cloudtrail create-trail \
  --name $APP_NAME-$ENV-trail \
  --s3-bucket-name $TRAIL_BUCKET \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name $APP_NAME-$ENV-trail
```

### 15.27 Enable GuardDuty

```bash
aws guardduty create-detector --enable
```

### 15.28 Enable AWS Config

AWS Config requires a recorder role and delivery channel. In production, deploy with CloudFormation/CDK for consistency.

```bash
aws configservice describe-configuration-recorders
```

### 15.29 Create CloudWatch alarms

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "$APP_NAME-$ENV-alb-5xx-high" \
  --metric-name HTTPCode_ELB_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=$(echo $ALB_ARN | awk -F'loadbalancer/' '{print $2}')
```

### 15.30 Create CloudFront distribution

CloudFront CLI JSON can be lengthy. Recommended production settings:

- Viewer protocol policy: redirect HTTP to HTTPS.
- Origin 1: S3 static bucket with Origin Access Control.
- Origin 2: ALB for API requests.
- WAF attached at CloudFront scope for global protection.
- Cache policy for static assets.
- Disabled or short TTL for dynamic APIs.
- Response headers policy for HSTS and security headers.
- Access logging enabled.

Example skeleton:

```bash
aws cloudfront create-distribution --distribution-config file://cloudfront-config.json
```

---

## 16. Optional EKS CLI Implementation Path

### 16.1 Create EKS cluster with eksctl

```bash
eksctl create cluster \
  --name $APP_NAME-$ENV-eks \
  --region $AWS_REGION \
  --vpc-private-subnets $APP_SUBNET_1,$APP_SUBNET_2,$APP_SUBNET_3 \
  --without-nodegroup
```

### 16.2 Create managed node group

```bash
eksctl create nodegroup \
  --cluster $APP_NAME-$ENV-eks \
  --region $AWS_REGION \
  --name app-workers \
  --node-type m7g.large \
  --nodes 6 \
  --nodes-min 6 \
  --nodes-max 50 \
  --managed \
  --private-networking
```

### 16.3 Install AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$APP_NAME-$ENV-eks \
  --set region=$AWS_REGION \
  --set vpcId=$VPC_ID
```

### 16.4 Example Kubernetes deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 6
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/enterprise-shop/api:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
```

---

## 17. Optional EC2 Auto Scaling CLI Implementation Path

### 17.1 Create launch template

```bash
AMI_ID="ami-REPLACE"
INSTANCE_PROFILE="enterprise-shop-prod-ec2-instance-profile"

aws ec2 create-launch-template \
  --launch-template-name $APP_NAME-$ENV-lt \
  --launch-template-data "{\"ImageId\":\"$AMI_ID\",\"InstanceType\":\"m7i.large\",\"IamInstanceProfile\":{\"Name\":\"$INSTANCE_PROFILE\"},\"SecurityGroupIds\":[\"$APP_SG\"],\"MetadataOptions\":{\"HttpTokens\":\"required\"},\"BlockDeviceMappings\":[{\"DeviceName\":\"/dev/xvda\",\"Ebs\":{\"Encrypted\":true,\"KmsKeyId\":\"$KMS_KEY_ID\",\"VolumeSize\":50}}]}"
```

### 17.2 Create Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name $APP_NAME-$ENV-asg \
  --launch-template LaunchTemplateName=$APP_NAME-$ENV-lt,Version='$Latest' \
  --min-size 6 \
  --max-size 100 \
  --desired-capacity 6 \
  --vpc-zone-identifier "$APP_SUBNET_1,$APP_SUBNET_2,$APP_SUBNET_3" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 300
```

### 17.3 Attach scaling policy

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name $APP_NAME-$ENV-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":60.0}'
```

---

## 18. CI/CD Production Pipeline

### 18.1 Recommended AWS-native CI/CD

```text
Developer -> CodeCommit/GitHub -> CodePipeline
          -> CodeBuild: test, scan, build image
          -> ECR: store image
          -> CodeDeploy/ECS deployment
          -> CloudWatch alarms verify deployment
```

### 18.2 Deployment controls

- Branch protection.
- Pull request review.
- Unit and integration tests.
- Container image vulnerability scan.
- Infrastructure policy scan.
- Secrets scan.
- Blue/green or canary deployment.
- Automatic rollback on failed health checks.
- Manual approval for production.

---

## 19. Observability and SRE Operations

### 19.1 Golden signals

Track these signals for every service:

- Latency: p50, p90, p95, p99.
- Traffic: requests per second.
- Errors: 4xx, 5xx, business errors.
- Saturation: CPU, memory, queue depth, DB connections.

### 19.2 Key CloudWatch alarms

| Alarm | Threshold example |
|---|---|
| ALB 5xx | > 1% for 5 minutes |
| Target 5xx | > 1% for 5 minutes |
| API p95 latency | > 500 ms for 10 minutes |
| ECS CPU | > 75% for 10 minutes |
| ECS memory | > 80% for 10 minutes |
| Aurora CPU | > 75% for 10 minutes |
| Aurora connections | > 80% max connections |
| Redis evictions | > 0 sustained |
| SQS queue age | > business SLA |
| DLQ messages | > 0 |
| WAF blocked spike | sudden anomaly |

### 19.3 Tracing

Use AWS X-Ray or OpenTelemetry to trace:

- API Gateway/ALB request path.
- ECS service calls.
- DynamoDB calls.
- Aurora queries.
- External payment API calls.
- SQS/EventBridge async handoff.

### 19.4 Log strategy

Use structured JSON logs with fields:

```json
{
  "timestamp": "2026-06-26T10:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "abc123",
  "customer_id_hash": "hash-value",
  "order_id": "ord-123",
  "message": "Order created"
}
```

Do not log:

- Passwords.
- Full payment card details.
- Secrets or tokens.
- Sensitive personally identifiable information unless masked and approved.

---

## 20. Failure Scenarios and Recovery

### 20.1 AZ failure

Expected behavior:

- ALB stops routing to unhealthy targets.
- ECS replaces failed tasks in healthy AZs.
- Aurora fails over if writer is affected.
- Redis promotes a replica if primary fails.
- Queues continue buffering messages.

### 20.2 Database writer failure

Expected behavior:

- Aurora promotes a reader to writer.
- Application retries with exponential backoff.
- RDS Proxy reduces connection storm impact.
- Idempotency prevents duplicate orders.

### 20.3 Redis failure

Expected behavior:

- Redis automatic failover promotes replica.
- Application degrades to database reads for critical flows.
- Cache miss rate increases temporarily.
- Non-critical cache-dependent features may degrade gracefully.

### 20.4 Payment provider failure

Expected behavior:

- Circuit breaker opens after repeated failures.
- Checkout returns a user-friendly retry message.
- Payment intent is stored with pending status.
- Retry worker processes payment later if business allows.

### 20.5 Regional failure

Expected behavior depends on DR strategy:

- Backup/restore: restore in DR Region.
- Warm standby: scale DR compute and route traffic using Route 53.
- Active-active: Route 53/ARC shifts traffic to healthy Region.

---

## 21. Industry Use Cases

### 21.1 E-commerce

- Product browsing at high scale.
- Flash sales and seasonal events.
- Cart and checkout resiliency.
- Fraud detection event streams.
- Order and inventory consistency.

### 21.2 Streaming application mapping

For a streaming app, replace shopping-specific modules:

| Shopping app module | Streaming app equivalent |
|---|---|
| Product catalog | Video catalog |
| Cart | Watchlist |
| Checkout/payment | Subscription billing |
| Inventory | Content entitlement |
| Product media in S3 | Video assets in S3 |
| CloudFront static assets | CloudFront media delivery |
| OpenSearch product search | Content search |
| Order events | Playback/subscription events |
| Notifications | New episode/recommendation alerts |

Additional AWS services for streaming:

- AWS Elemental MediaConvert for transcoding.
- AWS Elemental MediaPackage for video packaging.
- CloudFront signed URLs/cookies for protected streaming.
- S3 lifecycle policies for media assets.
- Kinesis Data Streams or Firehose for playback analytics.

### 21.3 Banking and fintech

- Strong audit with CloudTrail.
- Encryption with KMS.
- Transaction correctness using Aurora.
- Idempotent payment APIs.
- Fraud detection using event streams.

### 21.4 Healthcare

- Private networking.
- Strict access control.
- Full audit logging.
- Encryption at rest and in transit.
- Sensitive data masking in logs.

### 21.5 Travel and ticketing

- High read scale for search and availability.
- Consistent booking flow to avoid double booking.
- Queue-based processing for confirmations.
- Regional latency optimization.

---

## 22. Production Readiness Checklist

### 22.1 Security checklist

- [ ] Root account secured with MFA.
- [ ] IAM Identity Center enabled.
- [ ] Least privilege IAM policies reviewed.
- [ ] No public access to S3 buckets.
- [ ] WAF enabled with managed rules.
- [ ] CloudTrail multi-Region trail enabled.
- [ ] GuardDuty enabled.
- [ ] Security Hub enabled.
- [ ] AWS Config enabled.
- [ ] KMS encryption enabled for data stores.
- [ ] Secrets stored in Secrets Manager.
- [ ] Secrets rotation configured where possible.
- [ ] ECR image scanning enabled.
- [ ] VPC endpoints configured for private AWS service access.

### 22.2 Reliability checklist

- [ ] Workload deployed across at least two AZs; three preferred.
- [ ] ECS/EKS/EC2 autoscaling configured.
- [ ] ALB health checks configured.
- [ ] Aurora Multi-AZ configured.
- [ ] Redis Multi-AZ failover configured.
- [ ] SQS DLQs configured.
- [ ] Backups enabled and tested.
- [ ] DR runbook documented.
- [ ] Game days conducted.

### 22.3 Performance checklist

- [ ] CloudFront enabled.
- [ ] Redis caching implemented.
- [ ] DynamoDB keys tested for hot partitions.
- [ ] Aurora indexes optimized.
- [ ] Connection pooling enabled.
- [ ] p95 and p99 latency dashboards created.
- [ ] Load testing completed.

### 22.4 Operations checklist

- [ ] Structured logs enabled.
- [ ] CloudWatch dashboards created.
- [ ] Alarms connected to incident channel.
- [ ] Runbooks created.
- [ ] Deployment rollback tested.
- [ ] On-call escalation path defined.
- [ ] Cost budgets and anomaly detection enabled.

---

## 23. Recommended Production Enhancements

- Use AWS Organizations and Control Tower for multi-account governance.
- Separate accounts for dev, test, staging, prod, security, logging, and shared services.
- Centralize logs in a dedicated log archive account.
- Use AWS Firewall Manager to manage WAF across accounts.
- Use Route 53 ARC for advanced multi-Region recovery control.
- Use AWS Backup for centralized backup policies.
- Use Service Control Policies to restrict risky actions.
- Use CloudFormation/CDK/Terraform instead of manual CLI for repeatability.
- Use canary deployments for high-risk services.
- Use feature flags for controlled rollout.
- Use chaos testing and resilience testing.

---

## 24. Service Selection Summary

| Concern | Recommended AWS service |
|---|---|
| DNS | Route 53 |
| CDN | CloudFront |
| Static hosting | S3 |
| WAF | AWS WAF |
| DDoS | AWS Shield |
| TLS | ACM |
| Load balancing | ALB |
| Containers | ECS Fargate |
| Kubernetes optional | EKS |
| VM optional | EC2 Auto Scaling |
| Container registry | ECR |
| Relational DB | Aurora PostgreSQL/MySQL |
| Key-value DB | DynamoDB |
| Cache | ElastiCache Redis |
| Search | OpenSearch |
| Queue | SQS |
| Pub/sub | SNS |
| Event bus | EventBridge |
| Secrets | Secrets Manager |
| Encryption | KMS |
| Logs/metrics | CloudWatch |
| Tracing | X-Ray/OpenTelemetry |
| Audit | CloudTrail |
| Threat detection | GuardDuty |
| Security posture | Security Hub |
| Compliance config | AWS Config |

---

## 25. AWS Documentation References

These official AWS references were used while preparing this guide:

- AWS Well-Architected Reliability Pillar: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html
- Elastic Load Balancing documentation: https://docs.aws.amazon.com/elasticloadbalancing/
- Application Load Balancer documentation: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html
- Amazon ECS and AWS Fargate documentation: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html
- Amazon EKS architecture documentation: https://docs.aws.amazon.com/eks/latest/userguide/eks-architecture.html
- Amazon Aurora high availability documentation: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html
- Amazon DynamoDB global tables documentation: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html
- Amazon ElastiCache Multi-AZ documentation: https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/AutoFailover.html
- AWS WAF documentation: https://docs.aws.amazon.com/waf/
- Amazon CloudFront caching documentation: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ConfiguringCaching.html
- AWS Secrets Manager documentation: https://docs.aws.amazon.com/secretsmanager/
- AWS KMS documentation: https://docs.aws.amazon.com/kms/
- AWS CloudTrail documentation: https://docs.aws.amazon.com/cloudtrail/

---

## 26. Final Architecture Recommendation

For most enterprise shopping applications on AWS, the recommended starting point is:

```text
Route 53
  -> CloudFront + WAF + Shield
  -> S3 for static frontend
  -> ALB for APIs
  -> ECS Fargate services in private subnets
  -> Aurora Multi-AZ for transactional data
  -> DynamoDB for catalog/cart/session/idempotency data
  -> ElastiCache Redis for low-latency caching
  -> SQS/SNS/EventBridge for async workflows
  -> CloudWatch/X-Ray/CloudTrail/GuardDuty/Security Hub for operations and security
```

This architecture provides strong security, high availability, horizontal scalability, low latency, operational observability, and enterprise governance while minimizing infrastructure management overhead.
