# scalable-web-app-infrastructure-on-aws

Global Layout
Single region, two AZs: Minimum required for true high availability and RDS Multi-AZ. Adding a third AZ introduces extra NAT gateways, subnets, and cost with little learning benefit here.

Rejected: Multi-region (out of scope for the brief).

VPC 10.0.0.0/16 with /24 subnets: Leaves room to grow with a readable allocation scheme (public 0–1, app 10–11, data 20–21).

Rejected: A tight /24 VPC (leaves no room to expand).

Three tiers (public / app / data): Enforces least privilege per tier. The data tier has no internet route, preventing a compromised application instance from using the database host to reach external networks.

Rejected: Two-tier design (database shares a subnet with the application).

Entry Path
One Internet Gateway (IGW): Already redundant across AZs at the VPC level, so a second IGW adds no resilience.

Public subnets restricted to ALB and NAT Gateways: Nothing else is directly reachable from the internet.

Rejected: EC2 instances in public subnets with public IPs (exposes every instance).

Application Load Balancer (Layer 7): Provides path-based routing, HTTP health checks, multi-AZ coverage, and WAF integration.

Rejected: Network Load Balancer (Layer 4 only, no WAF or path-based routing) and Classic Load Balancer (legacy).

WAF attached to the ALB: Applies managed OWASP Top 10 rules to the dynamic traffic path per the brief.

Rejected: WAF on CloudFront (covers static content too, but requires global setup in us-east-1 and adds complexity; worth noting as a future improvement).

Route 53 alias record to the ALB (with health checks): Free for AWS targets, works at the zone apex (unlike a CNAME), and uses health checks to react to unhealthy endpoints.

CloudFront with ALB origin on a static. hostname: Caches static assets at the edge to reduce latency while keeping an S3 origin out of scope.

Rejected: S3 origin (best practice, but expands scope) and placing CloudFront in front of all traffic (would prevent Route 53 from pointing directly to the ALB as specified in the brief).

Compute
EC2 in private subnets: Instances have no public IPs and are reachable only through the ALB.

Rejected: Public instances locked down with strict security groups (one misconfiguration away from exposure).

Auto Scaling Group (ASG) across both AZs (min = 2, target tracking): Replaces unhealthy instances, tolerates an AZ loss, and scales on a straightforward metric target.

Rejected: Step scaling as primary policy (requires manual threshold tuning; can be noted as an alternative tied to the brief's learning outcomes).

Security group chain (ALB → App → DB): Each security group references the upstream group rather than static IP ranges, keeping rules accurate as instances scale or change.

Rejected: CIDR-based rules (brittle).

Outbound Access
One NAT Gateway per AZ: Preserves outbound connectivity if one AZ fails and avoids cross-AZ data transfer fees, at the cost of a second NAT Gateway (roughly ~$1/day each).

Rejected: Single NAT Gateway (cheaper, but creates an AZ-level single point of failure).

S3 Gateway Endpoint: Free; routes S3 traffic (including OS package repositories) over the AWS network instead of paying NAT Gateway data processing charges.

Rejected: Routing all traffic through NAT Gateways.

AWS Systems Manager Session Manager + IAM instance profile: Eliminates bastion hosts, inbound port 22, and SSH key management while logging sessions. Requires the IAM role and an outbound path via NAT.

Rejected: Bastion host or direct SSH (adds an exposed host and key management overhead).

Data
RDS Multi-AZ (single endpoint): Uses synchronous replication to a standby in another AZ and automatic failover while keeping a single DNS endpoint for the app. The standby does not serve traffic during normal operation.

Rejected: Read replica (asynchronous, built for read scaling rather than HA) and self-managed database on EC2 (manual backups, patching, and failover).

Data subnets with no default route (0.0.0.0/0): Blocks outbound internet access from the database tier and prevents external reachability.

Observability
CloudWatch alarms + SNS: Alerts via email on ASG, ALB, and RDS metrics, and pairs with Route 53 health checks.

Rejected: Manual monitoring.

Intentional Gap to Document (NACLs)
The brief mentions Network ACLs (NACLs), which are omitted from the diagram.

Decision: Keep default NACLs (allow all) and enforce access control through stateful security groups, avoiding stateless NACL misconfiguration risk. Document this explicitly in the README.