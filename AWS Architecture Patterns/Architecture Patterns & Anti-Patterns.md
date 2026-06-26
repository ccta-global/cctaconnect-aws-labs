# Architecture Patterns & Anti-Patterns

## How Architects Think Beyond Technology

---

# Part 1: Architecture Patterns

## 1. Layered Architecture (N-Tier Architecture)

### Structure

```text
Presentation Layer
        ↓
Business Layer
        ↓
Data Access Layer
        ↓
Database
```

### Used In

* Enterprise Applications
* ERP Systems
* Banking Applications

### Examples

#### Example 1

Online Banking Portal

UI → Services → Database

#### Example 2

Hospital Management System

#### Example 3

Insurance Claims Application

### Benefits

* Easy to understand
* Easy to maintain

### Challenges

* Can become tightly coupled
* Scaling entire application may be required

---

## 2. Microservices Architecture

### Structure

```text
User
 ↓
API Gateway
 ↓

--------------------
Order Service
Payment Service
Inventory Service
Notification Service
--------------------
```

### Used In

* Amazon
* Netflix
* Uber
* Spotify

### Examples

#### Example 1

Order Service deployed independently

#### Example 2

Payment Service scaled separately

#### Example 3

Notification Service updated without impacting others

### Benefits

* Independent deployment
* Independent scaling

### Challenges

* Distributed complexity
* Network failures
* Data consistency

---

## 3. Event-Driven Architecture

### Structure

```text
Producer
    ↓
Event Bus
    ↓
Consumers
```

### Used In

* E-commerce
* Banking
* IoT Platforms

### Examples

#### Example 1

Order Created Event triggers:

* Inventory Update
* Email Notification
* Billing

#### Example 2

Payment Success Event

#### Example 3

Sensor Data Processing

### Benefits

* Highly scalable
* Loosely coupled

### Challenges

* Debugging complexity
* Eventual consistency

---

## 4. CQRS Pattern

### Structure

```text
Commands (Write)
        ↓
Database
        ↑
Queries (Read)
```

### Examples

#### Example 1

Banking Dashboard

#### Example 2

Large Product Catalog

#### Example 3

Analytics Systems

### Benefits

* Optimized reads
* Optimized writes

### Challenges

* Increased complexity

---

## 5. Event Sourcing Pattern

Instead of storing:

```text
Balance = ₹5000
```

Store:

```text
Account Created
Money Deposited
Money Withdrawn
Money Deposited
```

### Examples

#### Example 1

Banking

#### Example 2

Insurance

#### Example 3

Trading Platforms

### Benefits

* Complete audit trail

### Challenges

* Complex implementation

---

## 6. API Gateway Pattern

### Purpose

Single entry point for all APIs.

### Examples

* AWS API Gateway
* Kong
* Apigee

### Responsibilities

* Authentication
* Authorization
* Rate Limiting
* Routing

---

## 7. Strangler Fig Pattern

### Purpose

Gradually replace monoliths.

### Approach

```text
Old Monolith
      ↓
New Service Added
      ↓
More Services Added
      ↓
Monolith Removed
```

### Examples

* Legacy Banking Systems
* ERP Modernization
* Mainframe Migration

---

## 8. Saga Pattern

### Purpose

Manage distributed transactions.

### Example Flow

```text
Order Created
      ↓
Payment Initiated
      ↓
Payment Failed
      ↓
Compensating Action
      ↓
Order Cancelled
```

### Examples

* Order Management
* Travel Booking
* Hotel Reservation

---

## 9. Sidecar Pattern

### Common in Kubernetes

```text
Application Container
       +
Logging Container
```

### Examples

* Log Collection
* Monitoring Agent
* Security Agent

---

## 10. Backend For Frontend (BFF)

### Purpose

Different APIs for different clients.

### Examples

* Mobile API
* Web API
* Smart TV API

---

# Part 2: Cloud Architecture Patterns

## Auto Scaling Pattern

### Examples

* AWS Auto Scaling Group
* Azure VM Scale Sets
* Kubernetes HPA

---

## Multi-AZ Pattern

### Examples

* AWS RDS Multi-AZ
* Load Balanced Applications
* Kubernetes Across Availability Zones

---

## Multi-Region Pattern

### Examples

* Global Banking
* Global E-Commerce
* SaaS Platforms

---

## Cache Aside Pattern

### Flow

```text
Application
     ↓
Check Cache
     ↓
Cache Hit? → Return Data
     ↓
Cache Miss
     ↓
Database
     ↓
Update Cache
     ↓
Return Data
```

### Examples

* Redis
* Memcached
* ElastiCache

---

# Part 3: Architecture Anti-Patterns

## 1. Distributed Monolith

### Looks Like

Microservices

### Behaves Like

Monolith

### Symptoms

Every service depends on every other service.

### Example

Order Service requires:

* Inventory Service
* Payment Service
* Customer Service
* Notification Service

### Problem

Microservices complexity without microservices benefits.

---

## 2. God Service

### Definition

One service handles too many responsibilities.

### Example

Customer Service handles:

* Orders
* Payments
* Inventory
* Notifications

### Problem

Hard to scale and maintain.

---

## 3. Shared Database Anti-Pattern

### Example

Multiple microservices use the same database.

### Problems

* Tight coupling
* Deployment issues
* Scaling bottlenecks

### Better Approach

Database per service.

---

## 4. Chatty Services

### Example

```text
User Request
     ↓
Service A
     ↓
Service B
     ↓
Service C
     ↓
Service D
     ↓
Response
```

### Problems

* High latency
* Excessive network calls

### Better Approach

API Aggregation

---

## 5. Premature Microservices

### Example

Startup with:

* Kubernetes
* Kafka
* Service Mesh
* 20 Microservices

for 100 users.

### Problem

Operational complexity exceeds business value.

### Better Approach

Start with Modular Monolith.

---

## 6. Database as Integration Layer

### Example

```text
Application A
      ↓
Shared Database
      ↑
Application B
```

### Problems

* Hidden dependencies
* Schema conflicts
* Tight coupling

### Better Approach

APIs or Events

---

## 7. Single Point of Failure (SPOF)

### Examples

* Single Server
* Single Database
* Single Load Balancer
* Single Network Device

### Rule

Every critical component should have redundancy.

---

## 8. Over-Caching

### Symptoms

Users receive stale information.

### Example

Stock Trading Application

Prices cached for one minute.

### Problem

Business-critical data becomes inaccurate.

---

## 9. No Observability

### Symptoms

Production issue occurs.

Nobody knows:

* What happened?
* Where?
* Why?

### Solution

Implement:

* Logs
* Metrics
* Traces

---

## 10. Technology-Driven Architecture

### Wrong Question

> Can we use Kubernetes?

### Right Question

> What business problem are we solving?

---

# Architecture Decision Record (ADR)

Every major architecture decision should document:

## Context

Why the decision is needed.

## Options

Alternatives considered.

## Decision

Chosen solution.

## Consequences

Benefits and risks.

---

## Example ADR

### Context

Need a scalable messaging platform.

### Options

* RabbitMQ
* Kafka
* AWS SQS

### Decision

Kafka

### Reason

High throughput requirements.

### Consequences

Improved scalability but increased operational complexity.

---

# Senior Architect Review Checklist

Before approving a design:

## Scalability

* Can it handle 10x growth?

## Availability

* Can it survive failures?

## Security

* Is data protected?

## Cost

* Is it financially sustainable?

## Operability

* Can support teams manage it?

## Maintainability

* Can engineers understand it?

## Simplicity

* Is there a simpler solution?

## Future Readiness

* Will it remain effective in 3 years?

---

# Key Takeaway

**Junior Engineers focus on code.**

**Senior Engineers focus on design.**

**Architects focus on trade-offs, risk management, business outcomes, and long-term sustainability.**
