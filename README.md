# Scalable Web Application Infrastructure on AWS

A highly available, auto-scaling web application platform on AWS: EC2 instances in private subnets behind an Application Load Balancer, a Multi-AZ RDS database, and a three-tier VPC spread across two Availability Zones. Built and documented step by step in the AWS console.

**Key services:** VPC, EC2 Auto Scaling, ALB, WAF, RDS Multi-AZ, CloudFront, Route 53, Systems Manager (Session Manager), CloudWatch, SNS.

**Design goal:** Survive the loss of an instance or a whole Availability Zone with no manual action, keep everything except the load balancer off the public internet, and scale with demand.

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Design decisions](#3-design-decisions)
4. [Build](#4-build)
5. [Testing and validation](#5-testing-and-validation)
6. [Cost and cleanup](#6-cost-and-cleanup)
7. [Issues encountered](#7-issues-encountered)
8. [Improvements](#8-improvements)

---

## 1. Overview

This project deploys a production-style web application on AWS and explains every choice behind it.

**What it does**

- Serves a web application from EC2 instances managed by an Auto Scaling group across two Availability Zones.
- Distributes traffic with an internet-facing Application Load Balancer protected by AWS WAF.
- Stores data in an RDS Multi-AZ database with automatic failover.
- Keeps all compute and data in private subnets. Outbound traffic goes through NAT Gateways, and administrators connect with Session Manager.
- Caches static assets at the edge with CloudFront and provides a private hosted zone with an alias to the ALB in addition to health checks.
- Monitors the platform with CloudWatch dashboards and alarms, with notifications through SNS.

**What it demonstrates**

- Designing a VPC with correct subnet, route table and NAT Gateway configuration.
- Building high availability across multiple Availability Zones.
- Configuring ALB listener rules and target group health checks.
- Auto Scaling with target tracking.
- Layered security: WAF, security group chaining, private subnets, no inbound SSH.

---

## 2. Architecture

![Architecture diagram](Diagrams/Project%20architecture.drawio.png)

### Request flow

1. **Name resolution:** users reach the ALB through its generated DNS name and CloudFront through its `*.cloudfront.net`name inside the VPC. A Route 53 private hosted zone resolves an internal name to the ALB with an alias record. Route 53 health checks monitor the ALB.
2. **Dynamic request:** user, Internet Gateway, ALB (inspected by WAF).
3. **Static request:** user, CloudFront edge cache.
4. **Cache miss:** CloudFront fetches the file from the ALB (its origin), then caches it.
5. **ALB to compute:** the ALB forwards to a healthy EC2 instance in either AZ through the target group.
6. **Data:** the instance reads and writes through the single RDS endpoint. The primary replicates synchronously to a standby in the other AZ.
7. **Outbound only:** patches and SSM traffic leave through the NAT Gateway in the instance's own AZ.
8. **Admin access:** Session Manager, with no bastion and no port 22.

### Network plan

| Tier | AZ A | AZ B | Internet route |
|---|---|---|---|
| Public (ALB, NAT) | 10.0.0.0/24 | 10.0.1.0/24 | Internet Gateway |
| Private app (EC2) | 10.0.10.0/24 | 10.0.11.0/24 | NAT Gateway (same AZ) |
| Private data (RDS) | 10.0.20.0/24 | 10.0.21.0/24 | none |

VPC CIDR: `10.0.0.0/16`.

### Security group chain

| Group | Inbound |
|---|---|
| `saa-web-alb-sg` | 80/443 from 0.0.0.0/0 |
| `saa-web-app-sg` | 80 from `saa-web-alb-sg` only |
| `saa-web-db-sg` | 3306 or 5432 from `saa-web-app-sg` only |

---

## 3. Design decisions

| Decision | Why |
|---|---|
| Two AZs | Minimum for real HA, required by RDS Multi-AZ |
| Three subnet tiers | Each tier gets only the access it needs and the data tier has no internet route |
| EC2 in private subnets | Only the ALB is reachable from the internet |
| Application Load Balancer | Layer 7, path routing, HTTP health checks, WAF support |
| WAF on the ALB | Managed rules against the OWASP Top 10 on the dynamic path |
| Auto Scaling, min 2, target tracking | Replaces failed instances and scales on a simple metric target |
| Security group referencing | Rules follow the groups, not IP addresses |
| One NAT Gateway per AZ | Outbound access survives an AZ failure and avoids cross-AZ traffic |
| Session Manager | No exposed port, no keys, sessions are logged |
| RDS Multi-AZ | Synchronous standby and automatic failover, one endpoint for the app |
| CloudFront with the ALB as origin | Caches static files at the edge without adding S3 to the scope |
| Route 53 alias to the ALB on `app.`, CloudFront on `static.` | Matches the requirement that Route 53 points to the ALB while keeping CloudFront's caching role |
| Default NACLs | Stateless NACLs are easy to misconfigure, security groups provide the controls |

---

## 4. Build

Each step follows the same template: **What**, **Why**, **How to verify**, **Screenshots**, **Issues**.

### 4.1 Network foundation

VPC, subnets, Internet Gateway, route tables, NAT Gateways, S3 gateway endpoint.

- **What:** `TODO`
- **Why:** `TODO`
- **How to verify:** `TODO`
- **Screenshots:** `TODO`
- **Issues:** `TODO`

### 4.2 Security groups

- **What:** `TODO`
- **Why:** `TODO`
- **How to verify:** `TODO`
- **Screenshots:** `TODO`
- **Issues:** `TODO`

### 4.3 Data layer

RDS Multi-AZ.

- **What:** `TODO`
- **Why:** `TODO`
- **How to verify:** `TODO`
- **Screenshots:** `TODO`
- **Issues:** `TODO`

### 4.4 Compute layer

Launch template and Auto Scaling group.

- **What:** `TODO`
- **Why:** `TODO`
- **How to verify:** `TODO`
- **Screenshots:** `TODO`
- **Issues:** `TODO`

### 4.5 Load balancing and edge

ALB, WAF, Route 53, CloudFront.

- **What:** `TODO`
- **Why:** `TODO`
- **How to verify:** `TODO`
- **Screenshots:** `TODO`
- **Issues:** `TODO`

### 4.6 Access and observability

Session Manager, CloudWatch, SNS.

- **What:** `TODO`
- **Why:** `TODO`
- **How to verify:** `TODO`
- **Screenshots:** `TODO`
- **Issues:** `TODO`

---

## 5. Testing and validation

| # | Claim | How it was tested | Expected | Result |
|---|---|---|---|---|
| 1 | Traffic reaches both AZs | Reload the page repeatedly through the ALB | Responses alternate between instances in both AZs | `TODO` |
| 2 | Failed instances are replaced | Terminate one instance | The ASG launches a replacement, and the site stays up | `TODO` |
| 3 | The group scales out under load | Generate CPU load | The target tracking policy adds instances | `TODO` |
| 4 | The database fails over | Reboot RDS with failover | Standby is promoted and the app reconnects | `TODO` |
| 5 | WAF blocks bad requests | Send a request matching a managed rule | HTTP 403 | `TODO` |
| 6 | Admin access needs no open port | Open a Session Manager shell | Shell opens, with no port 22 rule | `TODO` |
| 7 | CloudFront caches static files | Request the same static file twice | `X-Cache: Hit from cloudfront` on the second request | `TODO` |

---

## 6. Cost and cleanup

- **Estimated cost if left running:** roughly $3-4 per day (NAT Gateways, RDS Multi-AZ, ALB and public IPv4 addresses are the main items). `TODO: replace with actual spend from the Billing console`.
- **Cost controls used:** billing alarms at $5 and $10, resources deleted at the end of each session, smallest instance types.
- **Cleanup order:** `TODO: confirm` ASG, ALB, RDS, NAT Gateways (then release their Elastic IPs), endpoints, remaining resources by the `saa-web-` prefix.
- **Final check:** `TODO: nothing left running in any region`.

---

## 7. Issues encountered

| Step | Issue | Cause | Fix |
|---|---|---|---|
| `TODO` | `TODO` | `TODO` | `TODO` |

---

## 8. Improvements

- Infrastructure as code with Terraform for repeatable deploys and one-command teardown.
- S3 as the origin for static assets, with the ALB serving only dynamic content.
- HTTPS everywhere with ACM certificates, and an HTTP to HTTPS redirect.
- WAF on CloudFront as well as on the ALB.
- Step scaling or scheduled scaling to complement target tracking.
- RDS read replicas if reads become the bottleneck.
- VPC Flow Logs for network visibility, and custom NACLs for defense in depth.
- CI checks on the repository (diagram export, markdown linting).
