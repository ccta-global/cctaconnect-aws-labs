---
marp: true
theme: default
paginate: true
---

# Amazon VPC (Virtual Private Cloud)
### From Zero to Production Cloud Networking
*Easy-to-Understand Visual Guide, Analogies & Live Labs*

> **Speaker Notes:** Welcome everyone! Today we demystify cloud networking. Even if you have never configured a router before, by the end of this session you will know how to design, secure, and troubleshoot a production AWS network.

---

## Slide 1: What is Amazon VPC? (The Gated Community Analogy)

Think of **AWS Cloud** as a bustling city (e.g., Mumbai `ap-south-1`):

* 🏘️ **VPC** = Your own **private gated apartment complex** inside the city.
* 🏢 **Subnets** = Individual buildings inside your complex:
  * **Public Subnet** = Front lobby open to visitors.
  * **Private Subnet** = Residents-only tower (no direct street entrance).
* 🪧 **Route Tables** = Directional signboards guiding traffic between buildings and gates.
* 🚪 **Internet Gateway (IGW)** = The main front gate connecting your complex to the public highway.

> **Speaker Notes:** Emphasize two golden rules right away: A **VPC spans an entire Region**, while a **Subnet lives inside ONE Availability Zone**.

---

## Slide 2: IP Addresses & CIDR Made Simple

Every device needs an IP address (`10.0.1.25`). **CIDR** defines the size of your network:

* Format: `10.0.0.0 / 16`
* The `/16` means the **first 16 bits are locked**; the remaining `32 - 16 = 16` bits are free for devices ($2^{16} = 65,536$ IPs).

| CIDR | Total IPs | AWS Usable IPs | Best Used For |
|---|---|---|---|
| **`/16`** | 65,536 | 65,531 | Entire VPC (`10.0.0.0/16`) |
| **`/24`** | 256 | 251 | One Subnet (`10.0.1.0/24`) |
| **`/28`** | 16 | 11 | Smallest subnet AWS allows |

> **Speaker Notes:** Ask students: *"Why are usable IPs always 5 less than total IPs?"* Answer on the next slide!

---

## Slide 3: AWS's 5 Reserved IPs & The Golden Rule

In **every subnet** (e.g., `10.0.1.0/24`), AWS reserves **5 IP addresses**:

1. `.0` $\rightarrow$ Network Address
2. `.1` $\rightarrow$ **VPC Router** (the traffic director)
3. `.2` $\rightarrow$ **Amazon DNS Server** (resolves domain names)
4. `.3` $\rightarrow$ Reserved for future AWS use
5. `.255` $\rightarrow$ Broadcast Address

### ⚠️ Golden Rule of IP Planning
**Never overlap CIDR ranges** across VPCs or your office network. If two VPCs both use `10.0.0.0/16`, they can **never** talk to each other!

> **Speaker Notes:** Give a concrete example: VPC-Prod = `10.0.0.0/16`, VPC-Dev = `10.1.0.0/16`, Office = `192.168.0.0/16`.

---

## Slide 4: Public vs. Private Subnets — The Secret

There is **NO "Public" checkbox** in AWS! What makes a subnet public or private is its **Route Table**:

* **Public Subnet**: Route table has `0.0.0.0/0` $\rightarrow$ **Internet Gateway (`igw-xxxx`)**.
* **Private Subnet**: Route table has **NO route to IGW**. Outbound internet goes through a **NAT Gateway (`nat-xxxx`)**.
* **Isolated DB Subnet**: Route table has **only `local`** — zero internet access in or out!

```text
Public Route Table:   10.0.0.0/16 -> local   |   0.0.0.0/0 -> igw-1234
Private Route Table:  10.0.0.0/16 -> local   |   0.0.0.0/0 -> nat-5678
DB Route Table:       10.0.0.0/16 -> local
```

> **Speaker Notes:** Explain *Longest Prefix Match*: Specific routes (`10.0.0.0/16`) always win over the catch-all default route (`0.0.0.0/0`).

---

## Slide 5: Internet Gateway vs. NAT Gateway

How do servers talk to the outside world safely?

| Feature | Internet Gateway (IGW) | NAT Gateway |
|---|---|---|
| **Analogy** | Two-way main highway gate | One-way mail slot (residents can send out, strangers can't walk in) |
| **Direction** | Inbound & Outbound | **Outbound-initiated ONLY** |
| **Where It Lives** | Attached to the VPC | Placed inside a **Public Subnet** with an Elastic IP |
| **Availability** | Automatically Multi-AZ & Free | **Single-AZ** & Paid (deploy 1 per AZ in Prod!) |

> **Speaker Notes:** Common student confusion: *"If NAT Gateway is for private servers, why do we launch it inside a Public Subnet?"* Because the NAT Gateway itself needs a path to the Internet Gateway to forward traffic!

---

## Slide 6: VPC Firewalls — Security Groups vs. NACLs

| Feature | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| **Analogy** | 🕴️ Personal bodyguard at your apartment door | 🚧 Security guard at the building entrance |
| **Scope** | **Instance / ENI level** | **Subnet level** |
| **Memory** | **Stateful** (auto-allows return traffic) | **Stateless** (must allow return ephemeral ports `1024–65535`) |
| **Rules** | **Allow ONLY** | **Allow AND Deny** (numbered order: `100`, `200`...) |

> **Speaker Notes:** Ask: *"How do you block a single hacker IP?"* Answer: **NACL** (with a low rule number like `50`), because Security Groups don't have Deny rules!

---

## Slide 7: Tiered Security Group Chaining (Best Practice)

Instead of allowing IP addresses, link Security Groups together like Lego blocks:

```text
[ Internet ] 
     │  (Port 443 HTTPS)
     ▼
[ alb-sg ]  ──► Allows inbound 443 from 0.0.0.0/0
     │  (Port 8080)
     ▼
[ app-sg ]  ──► Allows inbound 8080 ONLY from [ alb-sg ]
     │  (Port 5432)
     ▼
[ db-sg ]   ──► Allows inbound 5432 ONLY from [ app-sg ]
```

* Even if an attacker compromises the public layer, the database only accepts packets from `app-sg`!

> **Speaker Notes:** Point out how clean this is: when Auto Scaling launches 20 new app servers with `app-sg`, the database automatically trusts them without updating a single IP rule.

---

## Slide 8: VPC Endpoints — Private Access to AWS Services

**The Problem**: Private EC2 servers downloading files from **Amazon S3** normally go through your **NAT Gateway** $\rightarrow$ **Expensive per-GB bill + public internet hop**.

**The Solution**: **VPC Endpoints** keep traffic 100% inside AWS's private fiber network!

1. **Gateway Endpoint (FREE)**:
   * Supports **S3 & DynamoDB only**.
   * Works by adding a route to your **Route Table**.
2. **Interface Endpoint / PrivateLink (Paid ENI)**:
   * Supports **100+ AWS services** (Secrets Manager, ECR, SQS, KMS).
   * Creates a **Private IP (ENI)** inside your subnet protected by a Security Group.

> **Speaker Notes:** Emphasize this FinOps win: adding a Gateway Endpoint for S3 is one of the fastest ways to slash an unexpected NAT Gateway bill.

---

## Slide 9: Connecting Multiple VPCs — Peering vs. Transit Gateway

### 1. VPC Peering (1-to-1 Tunnel)
* Connects two VPCs directly.
* **Non-Transitive**: If `A ↔ B` and `B ↔ C`, **`A` CANNOT reach `C`** through `B`.
* Great for 2–3 VPCs; becomes a tangled spiderweb at 20+ VPCs.

### 2. AWS Transit Gateway (Central Cloud Router)
* **Hub-and-Spoke** architecture connecting 100s of VPCs + On-Premises networks.
* **Transitive**: Everything plugs into one central hub with clean routing tables.

> **Speaker Notes:** Draw a triangle on the whiteboard for Peering ($A, B, C$) vs. a star topology for Transit Gateway.

---

## Slide 10: Hybrid Cloud — Site-to-Site VPN vs. Direct Connect

How do we connect an office data center to AWS?

* **AWS Site-to-Site VPN**:
  * Encrypted IPsec tunnel over the **public internet**.
  * Setup takes **minutes**; bandwidth up to ~1.25 Gbps per tunnel.
* **AWS Direct Connect (DX)**:
  * **Dedicated private physical fiber cable** from your office to AWS.
  * Setup takes **weeks/months**; speeds of **1 Gbps to 100 Gbps** with rock-solid low latency.
  * *Gotcha*: Not encrypted by default—pair with MACsec or IPsec VPN over DX!

> **Speaker Notes:** Many enterprises use **Direct Connect as primary** and **Site-to-Site VPN as automatic failover**.

---

## Slide 11: Production 3-Tier VPC Architecture (Putting It All Together)

```text
                     [ Public Internet ]
                             │
                    [ Internet Gateway ]
                      /              \
      ┌──────────────┴───┐        ┌───┴──────────────┐
      │ Public Subnet A  │        │ Public Subnet B  │  <-- ALB + NAT Gateways
      └────────┬─────────┘        └─────────┬────────┘
               ▼                            ▼
      ┌──────────────────┐        ┌──────────────────┐
      │ Private App A    │        │ Private App B    │  <-- EC2 / ECS / EKS
      └────────┬─────────┘        └─────────┬────────┘
               ▼                            ▼
      ┌──────────────────┐        ┌──────────────────┐
      │ Isolated DB A    │───────►│ Isolated DB B    │  <-- RDS Multi-AZ
      └──────────────────┘        └──────────────────┘
```

> **Speaker Notes:** Walk students through one full request lifecycle from user browser $\rightarrow$ IGW $\rightarrow$ ALB $\rightarrow$ App Server $\rightarrow$ RDS $\rightarrow$ back out!

---

## Slide 12: Troubleshooting Tools (When Traffic Won't Flow)

1. **VPC Reachability Analyzer** *(Your Best Friend)*:
   * Pick Source (e.g., `eni-123`) and Destination (e.g., `rds-456`).
   * AWS checks every Route Table, SG, and NACL and tells you **exactly which rule blocked the packet**!
2. **VPC Flow Logs**:
   * Captures `ACCEPT` and `REJECT` logs for IP traffic (sent to CloudWatch Logs or S3).
   * Great for auditing rejected connections and spotting port scans.

> **Speaker Notes:** Remind students: Flow Logs show **metadata** (IPs, ports, ACCEPT/REJECT), **not** packet payloads.

---

## Slide 13: Top 5 Student & Production Mistakes to Avoid

1. ❌ Creating overlapping CIDRs (`10.0.0.0/16` everywhere).
2. ❌ Putting only **1 NAT Gateway** in a Multi-AZ production VPC.
3. ❌ Forgetting outbound/inbound **ephemeral ports (`1024–65535`)** on custom NACLs.
4. ❌ Paying huge NAT Gateway fees for S3 traffic instead of using a **free S3 Gateway Endpoint**.
5. ❌ Thinking a Public IP makes an EC2 instance reachable without an **Internet Gateway route**.

> **Speaker Notes:** Challenge students to spot which of these 5 mistakes breaks a live demo!

---

## Slide 14: Live Hands-On Lab Roadmap

Today we will build a complete production VPC using **AWS CLI**:

* **Lab 1**: Create Custom VPC (`10.0.0.0/16`) + 4 Subnets across 2 AZs.
* **Lab 2**: Attach Internet Gateway, deploy NAT Gateway & configure Route Tables.
* **Lab 3**: Build chained Security Groups (`web-sg` $\rightarrow$ `app-sg`) & custom NACL.
* **Lab 4**: Attach a **free S3 Gateway VPC Endpoint** & verify the route table update.
* **Lab 5**: Peer two VPCs together and cleanly tear down all resources.

> **Speaker Notes:** Switch to your terminal and run `VPC.md` Section 14 step by step!

---

## Slide 15: Rapid-Fire Interview Recap

* **Q:** How many IPs are usable in a `/24` subnet? $\rightarrow$ **251** ($256 - 5$).
* **Q:** Stateful vs. Stateless firewall? $\rightarrow$ **Security Group (Stateful)** vs. **NACL (Stateless)**.
* **Q:** How do private instances reach S3 for free? $\rightarrow$ **Gateway VPC Endpoint**.
* **Q:** Is VPC Peering transitive? $\rightarrow$ **No** (use **Transit Gateway** for hub-and-spoke).
* **Q:** Tool to diagnose blocked network paths without sending traffic? $\rightarrow$ **VPC Reachability Analyzer**.

> **Speaker Notes:** Thank you! Open the floor for architecture whiteboard questions.