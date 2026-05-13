## Table of Contents

1. The Mental Model — What a VPC Really Is
2. Why VPC Exists (The Problem It Solves)
3. The Address Space — CIDR, IP Math, and Subnet Planning
4. Availability Zones and the Multi-AZ Strategy
5. Subnets — Public vs Private, and Why That Distinction Matters
6. The Internet Gateway — Doorway to the Public Internet
7. The NAT Gateway — One-Way Outbound for Private Workloads
8. Route Tables — The Brain of the Network
9. Security Groups vs Network ACLs — Two Layers of Defense
10. How a Packet Actually Travels (End-to-End Trace)
11. Anatomy of the Terraform VPC Module
12. Wiring Diagram — How Every Resource Connects
13. Production Considerations — Cost, Resilience, Security, Scale
14. Common Failure Modes and How to Debug Them
15. Glossary
16. Quick-Reference Cheat Sheet

---

## 1. The Mental Model — What a VPC Really Is

A VPC, or Virtual Private Cloud, is your own logically isolated slice of the AWS network. Think of AWS as a massive apartment building. Every customer in that building needs network space. A VPC is your private apartment — you decide the floor plan, where the doors are, which rooms are visible from the street, which rooms are locked behind interior walls, and which corridors connect to which.

More precisely, a VPC is a software-defined network. AWS gives you the illusion of having physical routers, switches, firewalls, and gateways, but underneath, it is all virtualized on top of AWS's underlying physical network fabric. Two important consequences fall out of this:

- **You never touch hardware.** Everything is API calls and configuration. Terraform sits on top of those APIs.
- **The boundaries are logical, not physical.** Two VPCs on the same physical AWS hardware are completely isolated from each other because the routing rules and packet encapsulation enforce that separation.

A VPC lives inside exactly one AWS region (for example, `us-east-1` or `ap-south-1`). It spans every Availability Zone in that region, but it does not cross regions. If you need a VPC in another region, you create another VPC there and connect them with peering or Transit Gateway.

The VPC itself is just a container. The interesting work happens with the resources you place inside it.

---

## 2. Why VPC Exists (The Problem It Solves)

Before VPC existed, AWS had a flat network called EC2-Classic. Every instance had a public IP, every instance could see every other instance in the same region (subject to security groups), and there was no way to isolate workloads at the network layer. This was operationally terrifying. A misconfigured security group could expose a database. There was no concept of "this server cannot, by design, be reached from the internet."

VPC solves five distinct problems:

1. **Isolation.** Your network is genuinely separate from every other AWS customer's network. Packets cannot leak between VPCs unless you explicitly create a connection.
2. **Address control.** You pick your IP ranges. This matters when you need to connect to on-premises networks that already use specific ranges.
3. **Segmentation.** You can carve your VPC into subnets, putting databases in one subnet, web servers in another, and enforcing that web servers can talk to databases but not the reverse direction of unsolicited traffic.
4. **Selective exposure.** Some resources must face the public internet (load balancers, bastion hosts). Most should not (databases, application servers, internal queues). VPC lets you express this distinction at the network layer rather than relying solely on application-level controls.
5. **Hybrid connectivity.** VPCs can connect to corporate data centers via VPN or Direct Connect, making AWS resources feel like an extension of an existing network.

The general principle is *defense in depth*. Application bugs happen. Misconfigurations happen. Credentials leak. The network layer is your last line of structural defense — if a private database has no path to the internet at all, then no amount of application compromise gets data exfiltrated through that path.

---

## 3. The Address Space — CIDR, IP Math, and Subnet Planning

Every VPC is defined by a CIDR block. CIDR stands for Classless Inter-Domain Routing, and it is a notation for expressing a range of IP addresses. The notation looks like `10.0.0.0/16`.

The number after the slash is the prefix length. It tells you how many bits at the front of the address are fixed. An IPv4 address is 32 bits, so `/16` means the first 16 bits are fixed and the remaining 16 bits are free to vary. Two-to-the-sixteenth is 65,536, so a `/16` VPC has roughly 65,536 IP addresses available.

A common CIDR cheat sheet looks like this:

| Prefix | Total IPs | Usable in a subnet | Typical use |
|---|---|---|---|
| /16 | 65,536 | 65,531 | A whole VPC |
| /20 | 4,096 | 4,091 | A large subnet |
| /24 | 256 | 251 | A standard subnet |
| /28 | 16 | 11 | A tiny subnet (Lambda, ENIs) |

AWS reserves five IPs in every subnet: the network address, the VPC router, two reserved for AWS DNS and future use, and the broadcast address. So a `/24` with 256 total IPs gives you 251 usable.

**Why this matters for the module.** The Easyshop-Infrastructure VPC module almost certainly takes a CIDR block as a variable, then subdivides it into smaller CIDR blocks for each subnet using Terraform's `cidrsubnet()` function. A typical layout for a `10.0.0.0/16` VPC across two Availability Zones might be:

- Public subnet AZ-a: `10.0.0.0/24`
- Public subnet AZ-b: `10.0.1.0/24`
- Private subnet AZ-a: `10.0.10.0/24`
- Private subnet AZ-b: `10.0.11.0/24`

The non-contiguous numbering (10, 11 instead of 2, 3) is deliberate. It leaves room between the public and private subnets for future subnet types like database subnets or intra-VPC subnets without renumbering.

**Use the RFC 1918 private ranges.** These three ranges are reserved for private networks and will never collide with anything on the public internet:

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

Pick a range that does not overlap with your on-premises network, your VPN partners, or your other VPCs. Overlapping CIDR is the single most common cause of "I cannot peer these VPCs" frustration months later.

---

## 4. Availability Zones and the Multi-AZ Strategy

An Availability Zone, or AZ, is a physically isolated data center campus within an AWS region. Each AZ has independent power, cooling, and network connectivity. If a tornado, a power outage, or a fiber cut takes out one AZ, the others keep running.

Every production VPC should span at least two AZs. Three is better. The reason is brutally simple: AWS does not guarantee that a single AZ stays up. The Service Level Agreement is regional, not zonal. If you deploy everything in `us-east-1a` and that zone has a bad day, your application is down.

**Subnets are AZ-scoped.** A subnet lives in exactly one AZ. You cannot have a subnet that spans multiple AZs. This is the fundamental reason VPC modules create N copies of each subnet type — one per AZ.

In the Easyshop pattern, the module receives a list of AZs as input (something like `["ap-south-1a", "ap-south-1b"]`) and uses Terraform's `count` or `for_each` to create a public subnet and a private subnet in each AZ.

**The multi-AZ payoff.** When you put an Application Load Balancer in front of EC2 instances, the load balancer is configured with subnets in multiple AZs. The ALB then has presence in each AZ and can route traffic to healthy targets even when one AZ fails. Same for RDS in multi-AZ mode — the standby replica lives in a different AZ than the primary, and failover is automatic.

You pay for this resilience in NAT Gateway costs (one per AZ is the standard production pattern) and in slightly more complex routing. Skipping multi-AZ to save money is a false economy if uptime matters.

---

## 5. Subnets — Public vs Private, and Why That Distinction Matters

A subnet is a slice of the VPC's address space, bound to a single AZ. The classification of "public" or "private" is not a property of the subnet itself — it is a property of the subnet's route table.

**A public subnet has a route to an Internet Gateway.** Resources in it can have public IP addresses, and traffic destined to `0.0.0.0/0` (anywhere on the internet) is sent to the Internet Gateway.

**A private subnet has no direct route to an Internet Gateway.** Resources in it have no public IP. Outbound internet traffic, if any, goes through a NAT Gateway. Inbound traffic from the internet cannot reach private subnets directly; it must go through a public-facing resource like a load balancer that then forwards to the private targets.

This distinction is *the* central concept of VPC design.

**What goes where in a typical production layout:**

| Subnet type | Typical residents |
|---|---|
| Public | Application Load Balancer, NAT Gateway, Bastion host, anything that must terminate public traffic |
| Private | EC2 application servers, ECS tasks, EKS worker nodes, Lambda functions in VPC, RDS databases, ElastiCache, internal services |

The Easyshop VPC module almost certainly creates both types. The application running on EasyShop (a web shop) follows the standard three-layer pattern: an ALB in the public subnet receives HTTPS, forwards to ECS/EC2 in the private subnet, which talks to RDS in a private subnet.

**Subnet sizing strategy.** Public subnets can be small — usually you only have a NAT Gateway and a load balancer there, each using one ENI. A `/24` is generous. Private subnets need to be larger because each EC2 instance, each ECS task, each Lambda execution can consume an IP. Underestimating this is a classic mistake. If you run a busy EKS cluster, you can exhaust a `/24` quickly because each pod gets its own IP under the AWS VPC CNI.

---

## 6. The Internet Gateway — Doorway to the Public Internet

An Internet Gateway, or IGW, is a horizontally scaled, redundant, highly available VPC component that allows communication between resources in your VPC and the public internet.

It does two jobs:

1. It is the **target in your route table** for internet-bound traffic from public subnets.
2. It performs **one-to-one NAT** for instances that have a public IPv4 address, mapping the instance's private IP to its public IP transparently.

Key properties to remember:

- **You attach exactly one IGW per VPC.** It is a VPC-level resource, not subnet-level.
- **It does not cost anything to exist.** You pay only for the data flowing through.
- **It does not limit bandwidth.** AWS scales it for you.
- **A subnet becomes public the moment its route table sends `0.0.0.0/0` to the IGW.** Nothing else changes about the subnet.

In Terraform, this is two resources: `aws_internet_gateway` (the gateway itself, with a reference to the VPC) and a route inside the public route table with `gateway_id` pointing at it. The Easyshop module ties these together.

A subtle but important point: an instance in a public subnet still needs a public IP (either via `map_public_ip_on_launch = true` on the subnet, or an Elastic IP) to actually be reachable from the internet. Being in a subnet with a route to an IGW is necessary but not sufficient — you also need a public address.

---

## 7. The NAT Gateway — One-Way Outbound for Private Workloads

Resources in private subnets often need outbound internet access. Application servers need to download package updates, call external APIs, fetch container images from public registries, send webhooks. They just must not be reachable *from* the internet on unsolicited connections.

This asymmetry is exactly what a NAT Gateway provides. NAT stands for Network Address Translation. The NAT Gateway sits in a public subnet, has an Elastic IP attached to it, and works like this:

1. An instance in a private subnet wants to reach `api.stripe.com`.
2. Its route table sends `0.0.0.0/0` to the NAT Gateway.
3. The NAT Gateway rewrites the packet's source IP from the instance's private IP to its own Elastic IP.
4. The packet exits through the Internet Gateway.
5. Stripe responds. The response comes back to the NAT Gateway's Elastic IP.
6. The NAT Gateway remembers the original mapping (this is the "stateful" part of NAT) and forwards the response to the original instance.

Stripe never learns the instance's private IP. Stripe could not initiate a connection to the instance even if it wanted to, because there is no route from the public internet to a private subnet.

**The Elastic IP.** A NAT Gateway needs a static public IP. That is an `aws_eip` resource in Terraform, and the NAT Gateway is configured to use it. The Elastic IP is the IP that the rest of the world sees as your traffic's origin.

**High availability.** The NAT Gateway itself is highly available *within its AZ*. It is not multi-AZ. If you put a single NAT Gateway in `ap-south-1a` and that AZ fails, every private subnet pointing at it loses internet access, even private subnets in healthy AZs. The production pattern is one NAT Gateway per AZ, with each private subnet's route table pointing at the NAT Gateway in its own AZ.

A typical Easyshop module either creates one NAT Gateway per AZ (production) or one shared NAT Gateway (dev, to save money — they cost roughly $32/month per gateway plus data processing). The module probably exposes a variable like `single_nat_gateway` to control this.

**Cost warning.** NAT Gateways are one of the most surprising line items on AWS bills. They charge per gigabyte of data processed, on top of the hourly rate. If your private workloads pull a lot of data from public sources (large container images, big package mirrors, S3 over the public path), this adds up. VPC endpoints, covered later, are the standard mitigation.

---

## 8. Route Tables — The Brain of the Network

A route table is a set of rules — each rule is called a route — that tells the VPC's virtual router where to send traffic with a given destination CIDR.

Every subnet must be associated with exactly one route table. If you do not associate one explicitly, the subnet uses the VPC's main route table. The Easyshop module almost certainly creates explicit route tables for public and private subnets rather than relying on the main route table.

**Routes are evaluated by longest-prefix match.** If a packet's destination matches multiple routes, the most specific one wins. A route to `10.0.5.0/24` beats a route to `10.0.0.0/16` beats a route to `0.0.0.0/0`.

Every route table starts with one route you cannot delete: the local route. It says "anything destined for the VPC CIDR stays inside the VPC." This is what makes intra-VPC communication just work without any configuration.

**Public route table:**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | Internet Gateway |

**Private route table:**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | NAT Gateway |

That is the entire difference. Same VPC, same hardware, same everything — but one subnet faces the world because its route table says so, and another is private because its route table says so.

**One route table per AZ for private subnets.** In a multi-AZ production setup with one NAT Gateway per AZ, you also need one private route table per AZ, because each one points at a different NAT Gateway. The module handles this by creating route tables in a loop and associating each private subnet with the route table for its AZ.

---

## 9. Security Groups vs Network ACLs — Two Layers of Defense

VPC gives you two firewall mechanisms. They look similar but behave differently, and understanding the difference is essential.

**Security Group.** A stateful, instance-level (technically ENI-level) firewall. You attach it to EC2 instances, RDS databases, load balancers, Lambda functions in VPC, ECS tasks. Properties:

- **Stateful.** If you allow inbound on port 443, the response traffic is automatically allowed outbound. You do not write the reverse rule.
- **Allow-only.** You can only write *allow* rules. The default is deny. If no rule matches, the traffic is dropped.
- **Default outbound is allow-all.** A new security group lets everything out unless you tighten it.
- **Composable.** A security group rule can reference another security group as its source. This is the idiomatic way: "the database SG allows port 5432 from the application SG."

**Network ACL.** A stateless, subnet-level firewall. Properties:

- **Stateless.** Inbound rules and outbound rules are independent. If you allow inbound on 443, you must also allow outbound on the ephemeral port range (1024–65535) for the responses.
- **Allow and deny.** Rules can explicitly deny. Rules are processed in numeric order until one matches.
- **Subnet-wide.** Applies to all traffic entering or leaving the subnet, regardless of which instance.

**Practical recommendation.** Use Security Groups as your primary tool. They are stateful, composable, and harder to misconfigure. Use Network ACLs sparingly — for example, to block a known-bad IP at the subnet boundary, or to enforce a coarse rule across all instances in a subnet.

The Easyshop VPC module probably keeps things simple here: it creates the VPC and subnets, and lets each consuming module (the ALB module, the EC2 module, the RDS module) define its own security groups. That separation is good practice — networking primitives stay in the VPC module, and access policies live with the workloads they protect.

---

## 10. How a Packet Actually Travels (End-to-End Trace)

To make all of this concrete, let us trace a real request: a user in Chennai opens `easyshop.example.com` in a browser, which loads an HTML page that calls an API, which writes to a database. Five hops happen inside the VPC.

**Step 1 — DNS resolution.** The browser resolves `easyshop.example.com` to the public IP of the Application Load Balancer. Route 53 (or whatever DNS) returns the ALB's public IP. The ALB's public IP lives on an ENI in a public subnet.

**Step 2 — TCP handshake to the ALB.** The user's packet arrives at the AWS edge, gets routed through the Internet Gateway, and lands at the ALB. The IGW does NAT — it sees the ALB's public IP, looks up which ENI in which subnet that maps to, and delivers the packet. The ALB's security group must allow inbound 443 from `0.0.0.0/0`.

**Step 3 — ALB forwards to a target.** The ALB picks a healthy target — say, an ECS task in the private subnet in AZ-a. The ALB opens a new TCP connection to the task's private IP. This traffic never leaves the VPC. The local route in the route table handles it: source is the ALB's ENI in the public subnet, destination is `10.0.10.45` in the private subnet, both are inside `10.0.0.0/16`, so the VPC router moves the packet directly. The task's security group must allow inbound on the application port from the ALB's security group.

**Step 4 — Application calls the database.** The task needs to write an order to RDS. RDS lives at a private DNS name that resolves to a private IP, say `10.0.20.10`. The task opens a TCP connection. Again, all intra-VPC. The RDS security group must allow inbound 5432 from the task's security group.

**Step 5 — Application calls an external API.** The task also needs to charge the user's card via Stripe. It makes an HTTPS request to `api.stripe.com`. DNS resolves to a public IP. The task's route table says `0.0.0.0/0` goes to the NAT Gateway in AZ-a. The packet hits the NAT Gateway, which rewrites the source to its Elastic IP, and forwards it through the Internet Gateway to Stripe. Stripe's response comes back the same path in reverse.

If you understand this trace, you understand VPC.

---

## 11. Anatomy of the Terraform VPC Module

A standard `modules/vpc/` directory contains a small set of files. Each has a specific job in the Terraform module conventions.

**`main.tf`** — declares the resources. For a VPC module, this typically includes:

- `aws_vpc` — the VPC itself
- `aws_subnet` (public, one per AZ) — using `count` or `for_each`
- `aws_subnet` (private, one per AZ)
- `aws_internet_gateway` — one, attached to the VPC
- `aws_eip` — Elastic IPs for NAT Gateways
- `aws_nat_gateway` — typically one per AZ
- `aws_route_table` (public) — one
- `aws_route_table` (private) — one per AZ
- `aws_route` or inline `route` blocks — `0.0.0.0/0` to IGW (public) and to NAT (private)
- `aws_route_table_association` — binds each subnet to its route table

**`variables.tf`** — declares inputs. Typical variables:

- `vpc_cidr` — the CIDR block for the VPC
- `public_subnet_cidrs` and `private_subnet_cidrs` — lists, one entry per AZ
- `availability_zones` — list of AZs to deploy into
- `project` or `environment` — for tagging
- `enable_nat_gateway` — boolean to allow disabling for dev
- `single_nat_gateway` — boolean to save cost in non-prod

**`outputs.tf`** — declares what the module exposes. Other modules consume these to place resources inside the VPC:

- `vpc_id` — referenced by security groups, endpoints, peering
- `public_subnet_ids` — list, for load balancers and bastions
- `private_subnet_ids` — list, for application instances and databases
- `vpc_cidr_block` — for security group rules that whitelist intra-VPC traffic
- `nat_gateway_ips` — sometimes useful for whitelisting outbound IPs at external services

**`terraform.tfvars`** or environment-specific tfvars — actual values. This usually lives outside the module, in the calling code, because the module is meant to be reusable.

The flow is: an environment (`environments/prod/main.tf`, for example) instantiates the VPC module with specific values, the module creates all the resources, and downstream modules (`modules/alb/`, `modules/ecs/`, `modules/rds/`) consume the outputs.

---

## 12. Wiring Diagram — How Every Resource Connects

To pull this all together, here is how the resources reference one another in code. The arrows mean "this resource has an attribute that points at that resource."

```
aws_vpc.main
   ↑
   ├── aws_internet_gateway.igw  (vpc_id = aws_vpc.main.id)
   │
   ├── aws_subnet.public[*]      (vpc_id, availability_zone, cidr_block)
   │      ↑
   │      ├── aws_eip.nat[*]                     (one per AZ)
   │      │      ↑
   │      │      └── aws_nat_gateway.nat[*]      (subnet_id = aws_subnet.public[az].id,
   │      │                                       allocation_id = aws_eip.nat[az].id)
   │      │
   │      └── aws_route_table_association.public[*]
   │             ↑
   │             └── aws_route_table.public
   │                    └── route 0.0.0.0/0 → aws_internet_gateway.igw
   │
   └── aws_subnet.private[*]     (vpc_id, availability_zone, cidr_block)
          ↑
          └── aws_route_table_association.private[*]
                 ↑
                 └── aws_route_table.private[*]       (one per AZ)
                        └── route 0.0.0.0/0 → aws_nat_gateway.nat[az]
```

**Reading this diagram:**

The VPC is the root. Everything else holds a `vpc_id` reference to it. The Internet Gateway attaches to the VPC. Public subnets exist in the VPC, each in a specific AZ. Inside each public subnet, a NAT Gateway is placed, each with its own Elastic IP. The public route table sends `0.0.0.0/0` to the Internet Gateway, and every public subnet is associated with this single route table.

On the private side, private subnets exist in the VPC, one per AZ. Unlike the public side, there is *one private route table per AZ*, because each one needs to send `0.0.0.0/0` to a different NAT Gateway — the NAT in the same AZ. Each private subnet is associated with its own AZ's route table.

Terraform figures out the order to create these by tracing the references. The IGW must exist before the public route's `0.0.0.0/0` entry can resolve. The Elastic IPs must exist before the NAT Gateways can claim them. The NAT Gateways must exist before the private routes can target them. You do not write this ordering; Terraform builds it from the resource graph.

---

## 13. Production Considerations — Cost, Resilience, Security, Scale

**Cost.** The two non-trivial line items in a VPC are NAT Gateway and VPC Endpoints. The IGW itself is free (you pay for data transfer). Subnets, route tables, security groups, and NACLs are all free. The bulk of VPC-related cost in production is usually:

- NAT Gateway hourly charge (one per AZ, ~$32/month each)
- NAT Gateway data processing (per GB processed)
- Inter-AZ data transfer (every byte that crosses an AZ boundary inside the VPC has a small per-GB cost)
- VPC endpoint hourly charges (for Interface endpoints)
- Public IPv4 addresses (AWS now charges per hour for every public IPv4, even attached ones)

**VPC endpoints** are the most important cost-and-security optimization to know about. Without them, when an EC2 instance in a private subnet reads from S3, the traffic goes out via the NAT Gateway to the public S3 endpoint and back. With a Gateway VPC Endpoint for S3 (free), the traffic stays on the AWS backbone and bypasses the NAT entirely. Interface endpoints (for services like Secrets Manager, ECR, SSM) are not free but eliminate NAT data processing charges for those services and keep the traffic off the public internet. A serious production VPC has S3 and DynamoDB Gateway endpoints at minimum.

**Resilience.** The recipe is multi-AZ everywhere. ALB in multiple AZs, ASG spanning multiple AZs, RDS Multi-AZ deployment, NAT Gateway per AZ, private route table per AZ. The cost of one extra NAT Gateway is far less than the cost of an outage.

**Security.** Apply the principle of least privilege at every layer:

- Public subnets contain only what *must* face the internet. NAT Gateway and load balancer, that is it.
- Application servers go in private subnets. Always.
- Databases go in private subnets. Always. Many teams put them in a dedicated "data" subnet with even tighter security groups.
- Security groups reference other security groups rather than CIDR blocks where possible. This way, scaling the application up does not require updating the database's allow-list.
- Enable VPC Flow Logs. They are free to generate (you pay for storage) and invaluable when investigating incidents.
- Use VPC endpoints for AWS service traffic so it never leaves the AWS network.

**Scale.** Plan the CIDR block for growth. A `/16` gives you headroom; a `/24` will run out fast in any non-trivial deployment. Avoid the temptation to use overlapping ranges across environments — production at `10.0.0.0/16`, staging at `10.1.0.0/16`, dev at `10.2.0.0/16` makes future peering trivial. Pick AZs intentionally; some regions have AZs that lack certain instance types, and discovering this in the middle of a deployment is painful.

---

## 14. Common Failure Modes and How to Debug Them

Most VPC problems boil down to one of these:

**"My instance cannot reach the internet."** Walk the checklist in order:

1. Is the instance in a public subnet (then it needs a public IP) or a private subnet (then it needs a working NAT path)?
2. Does the route table associated with the subnet have a `0.0.0.0/0` route?
3. Does that route point at the right target — IGW for public, NAT Gateway for private?
4. Is the NAT Gateway in the `Available` state, with an Elastic IP attached?
5. Does the instance's security group allow outbound on the port in question? (Default SGs allow all outbound, but customized ones may not.)
6. Does the network ACL allow both outbound (the request) *and* inbound (the response on ephemeral ports 1024–65535)?

**"I cannot reach my instance from the internet."** Reverse direction checklist:

1. Is the instance in a public subnet with a public IP?
2. Is the route table sending `0.0.0.0/0` to the IGW?
3. Does the security group allow inbound on the port from your source IP?
4. Does the NACL allow inbound on that port *and* outbound on the ephemeral port range?
5. Is the OS-level firewall (iptables, Windows Firewall) allowing the port?

**"Two instances in the same VPC cannot talk to each other."** Almost always a security group issue. The VPC's local route handles routing automatically; the problem is the firewall.

**"My peered VPCs cannot communicate."** Usually overlapping CIDR. Peering requires distinct ranges. Even with distinct ranges, you must add explicit routes in both directions in both VPCs' route tables, and security groups in both VPCs must allow the traffic.

**Diagnostic tools to know:**

- **VPC Reachability Analyzer.** Tell it "source X, destination Y, port Z," and it tells you whether the path is reachable, and if not, exactly which hop is blocking. This is the single most useful tool when you are stuck.
- **VPC Flow Logs.** Show every accepted and rejected packet at the ENI level. Search for REJECT entries to find what is being blocked.
- **CloudWatch Logs for NAT Gateway.** Shows port allocation errors if you exhaust the NAT's ephemeral port pool (a real problem at high concurrency).

---

## 15. Glossary

**VPC** — Virtual Private Cloud. Your logically isolated network in AWS.

**CIDR** — Classless Inter-Domain Routing. The `10.0.0.0/16` notation for IP ranges.

**Subnet** — A range of IPs within a VPC, bound to a single Availability Zone.

**Availability Zone (AZ)** — A physically isolated data center campus within an AWS region.

**Internet Gateway (IGW)** — VPC component allowing bidirectional internet connectivity. One per VPC.

**NAT Gateway** — Managed component allowing private-subnet outbound internet access. Lives in a public subnet, uses an Elastic IP.

**Elastic IP (EIP)** — A static public IPv4 address you can attach to resources. Persists across stop/start.

**Route Table** — A set of rules directing traffic by destination CIDR. Associated with one or more subnets.

**Security Group** — Stateful, instance-level firewall. Allow-only rules. Composable by SG reference.

**Network ACL (NACL)** — Stateless, subnet-level firewall. Allow and deny rules. Numbered.

**ENI** — Elastic Network Interface. A virtual NIC. Every EC2 instance has at least one. Security groups and IPs attach to ENIs.

**VPC Endpoint** — Private connection from your VPC to AWS services without going through the internet. Gateway type (S3, DynamoDB, free) and Interface type (everything else, paid).

**Flow Log** — Per-packet metadata record of accepted/rejected traffic at an ENI, subnet, or VPC.

**Peering** — A direct, private connection between two VPCs. Non-transitive.

**Transit Gateway** — A hub-and-spoke routing service connecting many VPCs and on-prem networks. Transitive.

---

## 16. Quick-Reference Cheat Sheet

**The five-second VPC explanation.** A VPC is your private network inside AWS. You pick the IP range. You divide it into subnets, one per Availability Zone. Public subnets have a route to the Internet Gateway and can host things that face the internet. Private subnets route outbound traffic through a NAT Gateway and host everything that should not be reachable from the internet. Security groups are the firewall on each resource, and they are stateful and reference other security groups.

**The one-page mental checklist for building a VPC:**

- Pick a CIDR block from RFC 1918. Use `/16` unless you have a reason not to.
- Pick at least two Availability Zones. Three for serious production.
- Create one public subnet per AZ and one private subnet per AZ.
- Attach one Internet Gateway to the VPC.
- Create one Elastic IP and one NAT Gateway per AZ. (Or one shared for dev.)
- Create one public route table with `0.0.0.0/0 → IGW`. Associate all public subnets.
- Create one private route table *per AZ* with `0.0.0.0/0 → NAT in same AZ`. Associate each private subnet with its AZ's route table.
- Add VPC endpoints for S3 and DynamoDB (free, save NAT cost).
- Add VPC endpoints for any AWS services your private workloads call heavily (SSM, ECR, Secrets Manager).
- Enable Flow Logs.
- Tag everything.

**The first questions to ask when something does not work:**

1. Is this resource in a public or a private subnet? Should it be?
2. What does the route table for that subnet say?
3. What does the security group allow?
4. Is the NACL doing something unexpected?
5. Have I checked Reachability Analyzer?

That is VPC. Every other VPC topic — peering, Transit Gateway, PrivateLink, IPAM, VPC Lattice, dual-stack IPv6 — builds on these primitives. Master the primitives and the rest is configuration.
