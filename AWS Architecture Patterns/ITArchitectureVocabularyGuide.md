# IT Architecture Vocabulary Guide

## Most Common Technical Terms Used During Architecture Discussions

---

# Why Architects Use These Terms

In architecture discussions, engineers are not just building features; they are designing systems that can handle growth, failures, traffic spikes, security risks, and business continuity requirements.

These terms help architects answer questions like:

* Can the system handle 10x growth?
* What happens if a server crashes?
* How quickly can we recover from disaster?
* Can the application serve millions of users?
* How much data can be processed per second?
* What are the trade-offs between cost and performance?

---

# 1. Scalability

## Definition

Scalability is the ability of a system to handle increasing workload without significantly affecting performance.

Simply put:

> If users grow from 1,000 to 1,000,000, can the system still work efficiently?

---

## Architecture Thinking

Architects ask:

* Can this solution support future growth?
* How much traffic can it handle?
* What happens if user volume increases 10 times?

---

## Examples

### Example 1: E-commerce Website

Current users: 10,000

Future users: 1,000,000

System should continue serving pages efficiently.

---

### Example 2: Netflix

Can serve millions of users watching videos simultaneously.

---

### Example 3: Banking Application

Handles normal transactions today and Black Friday level traffic tomorrow.

---

# 2. Vertical Scaling (Scale Up)

## Definition

Increasing resources of an existing server.

Example:

* CPU: 4 Core → 16 Core
* RAM: 8 GB → 64 GB

---

## Architecture Thinking

Question:

> Can we improve performance by making the machine bigger?

---

## Examples

### Example 1

Database Server

8 GB RAM → 64 GB RAM

---

### Example 2

Application Server

4 CPU → 32 CPU

---

### Example 3

SAP Server

Upgrade machine hardware rather than adding more servers.

---

## Pros

* Simple
* Easy to implement

## Cons

* Hardware limit exists
* Single point of failure

---

# 3. Horizontal Scaling (Scale Out)

## Definition

Adding more servers instead of upgrading one server.

---

## Architecture Thinking

Question:

> Can we distribute workload across multiple machines?

---

## Examples

### Example 1

1 Web Server → 10 Web Servers

Behind Load Balancer

---

### Example 2

Kubernetes Pods

5 Pods → 50 Pods

---

### Example 3

Microservices

Multiple instances deployed across nodes.

---

## Pros

* Better fault tolerance
* Unlimited growth potential

## Cons

* More complex architecture

---

# 4. Latency

## Definition

Time taken for a request to travel from source to destination and receive a response.

---

## Architecture Thinking

Question:

> How fast does the user get a response?

---

## Examples

### Example 1

Google Search

Response in 100 ms

---

### Example 2

Payment Gateway

Response in 500 ms

---

### Example 3

Database Query

Response in 20 ms

---

## Typical Targets

| System           | Latency |
| ---------------- | ------- |
| Trading Platform | <10 ms  |
| Search Engine    | <100 ms |
| E-Commerce       | <500 ms |

---

# 5. High Availability (HA)

## Definition

Ability of a system to remain operational even during failures.

---

## Architecture Thinking

Question:

> If one server dies, will the application still work?

---

## Examples

### Example 1

Two Application Servers

One fails → Traffic routed to second.

---

### Example 2

AWS Multi-AZ RDS

Primary DB fails → Standby takes over.

---

### Example 3

Load Balanced APIs

Traffic automatically redirected.

---

## Availability Formula

Availability = Uptime / Total Time

Examples:

| Availability | Downtime per Year |
| ------------ | ----------------- |
| 99%          | 3.65 Days         |
| 99.9%        | 8.76 Hours        |
| 99.99%       | 52 Minutes        |
| 99.999%      | 5 Minutes         |

---

# 6. Durability (AWS Perspective)

## Definition

Durability means data will not be lost.

---

## Architecture Thinking

Question:

> If hardware fails, can data still be recovered?

---

## Examples

### Example 1

AWS S3

11 Nines Durability

99.999999999%

---

### Example 2

Backup stored across multiple data centers.

---

### Example 3

Database snapshots replicated to another region.

---

## HA vs Durability

| HA                | Durability    |
| ----------------- | ------------- |
| Service Available | Data Safe     |
| Focus on uptime   | Focus on data |

---

# 7. RTO (Recovery Time Objective)

## Definition

Maximum acceptable time to restore service after failure.

---

## Architecture Thinking

Question:

> How quickly must we recover?

---

## Examples

### Example 1

Banking Application

RTO = 15 minutes

---

### Example 2

HR Portal

RTO = 4 hours

---

### Example 3

Internal Reporting Tool

RTO = 24 hours

---

# 8. RPO (Recovery Point Objective)

## Definition

Maximum acceptable data loss.

---

## Architecture Thinking

Question:

> How much data can we afford to lose?

---

## Examples

### Example 1

RPO = 0

No data loss allowed.

---

### Example 2

RPO = 5 minutes

Maximum 5 minutes of data loss.

---

### Example 3

Nightly Backup

RPO = 24 hours

---

## RTO vs RPO

| RTO                 | RPO               |
| ------------------- | ----------------- |
| Time to Recover     | Data Loss Allowed |
| Service Restoration | Data Restoration  |

---

# 9. Throughput

## Definition

Amount of work completed in a given time.

---

## Architecture Thinking

Question:

> How many requests can the system process?

---

## Examples

### Example 1

API handles 10,000 requests/sec

---

### Example 2

Kafka processes 1 Million messages/minute

---

### Example 3

Payment System processes 5,000 TPS

---

# 10. Throttling

## Definition

Restricting the number of requests a user/system can make.

---

## Architecture Thinking

Question:

> How do we protect the system from overload?

---

## Examples

### Example 1

API Gateway

100 requests/minute/user

---

### Example 2

Login Endpoint

5 failed attempts allowed

---

### Example 3

AWS Lambda Concurrency Limits

Prevent excessive invocation.

---

# 11. IOPS (Input Output Operations Per Second)

## Definition

Number of disk read/write operations per second.

---

## Architecture Thinking

Question:

> Can storage keep up with application demand?

---

## Examples

### Example 1

Database requires 10,000 IOPS

---

### Example 2

SSD supports higher IOPS than HDD

---

### Example 3

AWS EBS Provisioned IOPS Volume

Guaranteed performance.

---

# 12. Bandwidth

## Definition

Maximum amount of data transferable in a given time.

---

## Architecture Thinking

Question:

> How much data can flow through the network?

---

## Examples

### Example 1

100 Mbps Internet Connection

---

### Example 2

10 Gbps Data Center Network

---

### Example 3

Video Streaming Infrastructure

High bandwidth requirement.

---

# 13. Fault Tolerance

## Definition

Ability to continue operating despite component failures.

---

## Architecture Thinking

Question:

> Can the system survive failures automatically?

---

## Examples

### Example 1

Multiple Application Servers

---

### Example 2

Kubernetes Self-Healing Pods

---

### Example 3

Multi-Region Deployment

---

# 14. Load Balancing

## Definition

Distributing traffic across multiple servers.

---

## Architecture Thinking

Question:

> How do we avoid overloading one server?

---

## Examples

### Example 1

AWS ALB

---

### Example 2

NGINX Load Balancer

---

### Example 3

Azure Load Balancer

---

# 15. Caching

## Definition

Storing frequently accessed data closer to users.

---

## Architecture Thinking

Question:

> How can we reduce database calls?

---

## Examples

### Example 1

Redis Cache

---

### Example 2

CDN Edge Cache

---

### Example 3

Browser Cache

---

# 16. Consistency

## Definition

All users see the same data at the same time.

---

## Architecture Thinking

Question:

> Is latest data immediately visible everywhere?

---

## Examples

### Example 1

Bank Balance Updates

---

### Example 2

Inventory Management

---

### Example 3

Distributed Databases

---

# 17. Eventual Consistency

## Definition

Data becomes consistent after some delay.

---

## Architecture Thinking

Question:

> Can we trade immediate consistency for scalability?

---

## Examples

### Example 1

AWS DynamoDB

---

### Example 2

DNS Propagation

---

### Example 3

Social Media Like Counts

---

# 18. Stateless vs Stateful

## Stateless

Server stores no user session.

### Examples

* REST APIs
* Microservices
* AWS Lambda

---

## Stateful

Server remembers user information.

### Examples

* Database
* Shopping Cart Session
* WebSocket Connections

---

# 19. Resilience

## Definition

Ability to recover quickly from failures.

---

## Architecture Thinking

Question:

> How fast can the system bounce back?

---

## Examples

### Example 1

Circuit Breaker Pattern

---

### Example 2

Auto Scaling

---

### Example 3

Retry Mechanism

---

# 20. Idempotency

## Definition

Executing the same request multiple times produces the same result.

---

## Architecture Thinking

Question:

> What if client retries due to timeout?

---

## Examples

### Example 1

Payment API

---

### Example 2

Order Creation API

---

### Example 3

AWS SQS Message Processing

---

# Quick Architecture Interview Cheat Sheet

| Term                 | Architect Thinks About |
| -------------------- | ---------------------- |
| Scalability          | Growth                 |
| Vertical Scaling     | Bigger Server          |
| Horizontal Scaling   | More Servers           |
| Latency              | Response Time          |
| Throughput           | Capacity               |
| HA                   | Uptime                 |
| Durability           | Data Safety            |
| RTO                  | Recovery Time          |
| RPO                  | Data Loss              |
| Throttling           | Protection             |
| IOPS                 | Storage Performance    |
| Bandwidth            | Network Capacity       |
| Fault Tolerance      | Failure Handling       |
| Load Balancing       | Traffic Distribution   |
| Caching              | Performance            |
| Consistency          | Data Accuracy          |
| Eventual Consistency | Scalability Tradeoff   |
| Stateless            | Easy Scaling           |
| Stateful             | Session/Data Retention |
| Resilience           | Fast Recovery          |
| Idempotency          | Safe Retries           |

---

# Final Rule for Architecture Discussions

Whenever architects discuss a system, they usually evaluate it using these dimensions:

1. Scalability
2. Availability
3. Reliability
4. Durability
5. Performance
6. Security
7. Cost Optimization
8. Disaster Recovery (RTO/RPO)
9. Maintainability
10. Observability

A strong architecture balances all ten dimensions rather than optimizing only one.
