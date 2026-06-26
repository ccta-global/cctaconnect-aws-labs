# Advanced Architecture Vocabulary Guide

## Frequently Used Terms in Senior Architect, Cloud Architect, and System Design Discussions

---

# 21. CAP Theorem

## Definition

CAP Theorem states that a distributed system can guarantee only **two out of three** properties simultaneously:

* **C** = Consistency
* **A** = Availability
* **P** = Partition Tolerance

Since network failures (partitions) are inevitable in distributed systems, architects usually choose between Consistency and Availability.

---

## Architecture Thinking

Question:

> During a network failure, should the system prioritize correct data or continuous service?

---

## Examples

### Example 1: Banking System

Prioritizes Consistency over Availability.

Wrong account balance is unacceptable.

---

### Example 2: Social Media

Prioritizes Availability.

Temporary inconsistency is acceptable.

---

### Example 3: DNS

Availability is more important than immediate consistency.

---

# 22. Partition Tolerance

## Definition

Ability of a distributed system to continue operating despite network communication failures.

---

## Architecture Thinking

Question:

> What happens when two servers cannot communicate?

---

## Examples

### Example 1

Multi-region AWS deployment.

---

### Example 2

Kubernetes cluster node isolation.

---

### Example 3

Distributed database node failure.

---

# 23. ACID Transactions

## Definition

Properties that ensure reliable database transactions.

### A = Atomicity

All or nothing.

### C = Consistency

Valid state maintained.

### I = Isolation

Transactions don't interfere.

### D = Durability

Committed data survives failures.

---

## Architecture Thinking

Question:

> Does the business process require strict transactional integrity?

---

## Examples

### Example 1

Bank money transfer.

---

### Example 2

ATM withdrawal.

---

### Example 3

Credit card payment processing.

---

# 24. BASE Model

## Definition

Alternative to ACID for distributed systems.

### B = Basically Available

### A = Soft State

### S = Eventual Consistency

---

## Architecture Thinking

Question:

> Can we trade consistency for scalability?

---

## Examples

### Example 1

Amazon Shopping Cart.

---

### Example 2

Social Media Feed.

---

### Example 3

DynamoDB.

---

# 25. Sharding

## Definition

Splitting data across multiple databases.

---

## Architecture Thinking

Question:

> How do we scale databases beyond a single server?

---

## Examples

### Example 1

Users A-M in DB1.

Users N-Z in DB2.

---

### Example 2

Customer data split by geography.

---

### Example 3

Large SaaS multi-tenant databases.

---

## Benefits

* Improved performance
* Better scalability

---

# 26. Replication

## Definition

Copying data across multiple databases.

---

## Architecture Thinking

Question:

> How do we improve availability and disaster recovery?

---

## Examples

### Example 1

Primary-Replica Database.

---

### Example 2

AWS RDS Read Replica.

---

### Example 3

MongoDB Replica Set.

---

# 27. CQRS (Command Query Responsibility Segregation)

## Definition

Separate systems for:

* Writing data (Commands)
* Reading data (Queries)

---

## Architecture Thinking

Question:

> Can reads and writes be optimized independently?

---

## Examples

### Example 1

E-commerce Product Catalog.

---

### Example 2

Banking Dashboard.

---

### Example 3

Order Management System.

---

# 28. Event Sourcing

## Definition

Store every change as an event rather than storing only current state.

---

## Architecture Thinking

Question:

> Do we need complete audit history?

---

## Examples

### Example 1

Account Created

Money Deposited

Money Withdrawn

---

### Example 2

Insurance Claims System.

---

### Example 3

Trading Platforms.

---

# 29. Distributed Transactions

## Definition

Single transaction spanning multiple systems.

---

## Architecture Thinking

Question:

> How do we maintain consistency across services?

---

## Examples

### Example 1

Order Service + Payment Service.

---

### Example 2

Travel Booking Systems.

---

### Example 3

Inventory + Shipping Workflow.

---

# 30. Saga Pattern

## Definition

Manages distributed transactions using compensating actions.

---

## Architecture Thinking

Question:

> How do we rollback across multiple microservices?

---

## Examples

### Example 1

Order Created

Payment Failed

Order Cancelled

---

### Example 2

Flight Booking.

---

### Example 3

Hotel Reservation.

---

# 31. Circuit Breaker Pattern

## Definition

Stops repeated calls to failing services.

---

## Architecture Thinking

Question:

> How do we prevent cascading failures?

---

## Examples

### Example 1

Netflix Hystrix Pattern.

---

### Example 2

Payment Gateway Failure.

---

### Example 3

Inventory Service Outage.

---

# 32. Bulkhead Pattern

## Definition

Isolates resources so failures don't affect the whole system.

---

## Architecture Thinking

Question:

> Can one failing component take down everything?

---

## Examples

### Example 1

Separate thread pools.

---

### Example 2

Dedicated microservice resources.

---

### Example 3

Independent Kubernetes namespaces.

---

# 33. Retry Pattern

## Definition

Automatically retry failed operations.

---

## Architecture Thinking

Question:

> Is failure temporary or permanent?

---

## Examples

### Example 1

Network Timeout.

---

### Example 2

Database Connection Failure.

---

### Example 3

API Call Failure.

---

# 34. Backpressure

## Definition

Mechanism to slow incoming traffic when consumers cannot keep up.

---

## Architecture Thinking

Question:

> What happens if producers are faster than consumers?

---

## Examples

### Example 1

Kafka Consumer Lag.

---

### Example 2

Streaming Data Pipelines.

---

### Example 3

Reactive Programming.

---

# 35. Rate Limiting

## Definition

Restrict requests within a time window.

---

## Architecture Thinking

Question:

> How do we protect APIs from abuse?

---

## Examples

### Example 1

100 requests/minute.

---

### Example 2

OTP Generation Limits.

---

### Example 3

Login Protection.

---

## Difference from Throttling

| Rate Limiting   | Throttling           |
| --------------- | -------------------- |
| Enforces quotas | Slows traffic        |
| Hard limit      | Controlled reduction |

---

# 36. Auto Scaling

## Definition

Automatically add/remove resources based on demand.

---

## Architecture Thinking

Question:

> Can infrastructure adjust automatically?

---

## Examples

### Example 1

AWS Auto Scaling Group.

---

### Example 2

Kubernetes HPA.

---

### Example 3

Azure VM Scale Set.

---

# 37. Blue-Green Deployment

## Definition

Two identical environments.

### Blue

Current Production

### Green

New Version

Switch traffic instantly.

---

## Architecture Thinking

Question:

> How can we deploy with minimal risk?

---

## Examples

### Example 1

Application upgrade.

---

### Example 2

Database migration.

---

### Example 3

Production release rollback.

---

# 38. Canary Deployment

## Definition

Release software to a small percentage of users first.

---

## Architecture Thinking

Question:

> Can we test production safely?

---

## Examples

### Example 1

5% users receive new version.

---

### Example 2

Regional rollout.

---

### Example 3

Feature validation.

---

# 39. Rolling Deployment

## Definition

Replace instances gradually.

---

## Architecture Thinking

Question:

> Can we update without downtime?

---

## Examples

### Example 1

Kubernetes Rolling Update.

---

### Example 2

EKS Deployments.

---

### Example 3

Application Server Updates.

---

# 40. Zero Downtime Deployment

## Definition

Deploy software without affecting users.

---

## Architecture Thinking

Question:

> Can users continue working during deployment?

---

## Examples

### Example 1

Blue-Green Deployment.

---

### Example 2

Rolling Updates.

---

### Example 3

Canary Release.

---

# 41. Observability

## Definition

Ability to understand system health from outputs.

---

## Three Pillars

* Logs
* Metrics
* Traces

---

## Architecture Thinking

Question:

> Can we quickly diagnose production issues?

---

## Examples

### Example 1

Splunk Logs.

---

### Example 2

Prometheus Metrics.

---

### Example 3

OpenTelemetry Tracing.

---

# 42. Monitoring

## Definition

Tracking known system indicators.

---

## Examples

### Example 1

CPU Usage.

---

### Example 2

Memory Utilization.

---

### Example 3

API Response Time.

---

# 43. SLI (Service Level Indicator)

## Definition

Actual measured performance metric.

---

## Examples

### Example 1

99.95% Availability.

---

### Example 2

Response Time 200 ms.

---

### Example 3

Error Rate 0.01%.

---

# 44. SLO (Service Level Objective)

## Definition

Target performance goal.

---

## Examples

### Example 1

99.9% Uptime.

---

### Example 2

Response Time < 500 ms.

---

### Example 3

Error Rate < 1%.

---

# 45. SLA (Service Level Agreement)

## Definition

Contractual commitment made to customers.

---

## Examples

### Example 1

AWS 99.99% SLA.

---

### Example 2

Banking Application SLA.

---

### Example 3

Managed Service Provider Agreements.

---

# 46. Consensus Algorithm

## Definition

Mechanism by which distributed systems agree on a value.

---

## Architecture Thinking

Question:

> How do distributed nodes agree on truth?

---

## Examples

### Example 1

Raft.

---

### Example 2

Paxos.

---

### Example 3

ZooKeeper.

---

# 47. Leader Election

## Definition

Selecting a primary node among distributed nodes.

---

## Examples

### Example 1

Kubernetes Controller Leader.

---

### Example 2

MongoDB Primary Election.

---

### Example 3

ZooKeeper Coordination.

---

# 48. Service Discovery

## Definition

Automatically locating services in distributed systems.

---

## Architecture Thinking

Question:

> How do services find each other?

---

## Examples

### Example 1

Kubernetes DNS.

---

### Example 2

Consul.

---

### Example 3

Eureka Server.

---

# 49. API Gateway

## Definition

Single entry point for APIs.

---

## Architecture Thinking

Question:

> How do we centralize API management?

---

## Examples

### Example 1

AWS API Gateway.

---

### Example 2

Kong Gateway.

---

### Example 3

Apigee.

---

# 50. Service Mesh

## Definition

Infrastructure layer handling service-to-service communication.

---

## Architecture Thinking

Question:

> How do we manage microservice communication securely?

---

## Examples

### Example 1

Istio.

---

### Example 2

Linkerd.

---

### Example 3

Consul Service Mesh.

---

# Ultimate Architect's Mental Model

Whenever evaluating a system, senior architects typically score the design across these categories:

| Category            | Key Terms                                      |
| ------------------- | ---------------------------------------------- |
| Performance         | Latency, Throughput, IOPS                      |
| Scalability         | Vertical Scaling, Horizontal Scaling, Sharding |
| Availability        | HA, Replication, Failover                      |
| Reliability         | Fault Tolerance, Resilience                    |
| Data Integrity      | ACID, Consistency                              |
| Distributed Systems | CAP, Consensus, Saga                           |
| Security            | Authentication, Authorization, Encryption      |
| Operations          | Monitoring, Observability                      |
| Deployment          | Blue-Green, Canary, Rolling                    |
| Disaster Recovery   | RTO, RPO, Backup                               |
| Cloud Architecture  | Auto Scaling, Load Balancer, API Gateway       |
| Microservices       | Service Discovery, CQRS, Service Mesh          |
| Cost Optimization   | Right Sizing, Spot Instances, Caching          |

# Golden Rule of Architecture

A good architecture is not the one with the most technology.

A good architecture is the one that achieves:

* Business Goals
* Reliability
* Scalability
* Security
* Cost Efficiency
* Simplicity
* Maintainability

while minimizing operational complexity.
