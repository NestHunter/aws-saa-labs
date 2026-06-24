# AWS Lab: Enabling Outbound Internet Access from a Private Subnet with a NAT Gateway

## Overview

In this lab, I configured an EC2 instance in a private subnet so it could reach the internet for outbound traffic while still remaining inaccessible from the public internet.

This was one of the more clarifying VPC labs so far. It forced me to think carefully about the difference between what security groups *allow* and what route tables actually *enable* — two layers that work together but serve completely different purposes.

Core concepts at play:

- Route tables
- NAT Gateways
- Security groups
- Public vs. private subnet behavior

The key idea: a private instance should be able to initiate outbound connections without ever being directly reachable from the internet.

---

## Objective

Allow an EC2 instance in a private subnet to:

- Access the internet for outbound traffic (updates, package downloads, API calls, patching)
- Remain unreachable directly from the internet
- Accept controlled access only from a trusted source — in this case, the public EC2 instance via security group reference

This reflects a common real-world AWS design pattern where application servers live in private subnets but still need a path outward.

---

## The Initial Problem

When the lab started, the private instance had no internet connectivity.

My first instinct was to check the security group and ensure outbound HTTP/HTTPS was allowed. That was correct — but it didn't fix the problem.

This is where the lab got interesting.

> Security groups control what traffic is *allowed*.  
> Route tables control where traffic can *go*.

Even with the most permissive outbound security group rules, if the private subnet has no route pointing to a NAT Gateway, internet-bound traffic has nowhere to go. The packet hits a dead end at the subnet boundary.

---

## Architecture

**Environment:**

| Component | Details |
|---|---|
| VPC | Custom VPC |
| Public Subnet | Hosts the public EC2 instance and the NAT Gateway |
| Private Subnet | Hosts the private EC2 instance |
| Internet Gateway | Attached to the VPC; used by the public subnet |
| NAT Gateway | Deployed in the public subnet with an Elastic IP |
| Public EC2 | Entry point; used to reach the private instance |
| Private EC2 | Target instance; no public IP |

**Traffic flow design:**

```
Private EC2
    │
    ▼
Private Subnet Route Table
    │  0.0.0.0/0 → NAT Gateway
    ▼
NAT Gateway (in Public Subnet)
    │
    ▼
Public Subnet Route Table
    │  0.0.0.0/0 → Internet Gateway
    ▼
Internet Gateway
    │
    ▼
Internet
```

Return traffic flows back through the same path. Because security groups are stateful, the response traffic is automatically permitted without needing a separate inbound rule.

---

## What I Configured

### 1. NAT Gateway

Created a NAT Gateway in the **public subnet** and assigned it an Elastic IP.

Placement matters here: the NAT Gateway needs to live in a public subnet because it must have its own route to the Internet Gateway. A NAT Gateway placed in a private subnet cannot reach the internet — it has the same routing problem as the private instance itself.

### 2. Public Subnet Route Table

Confirmed the public subnet route table already had:

| Destination | Target |
|---|---|
| VPC CIDR (local) | local |
| 0.0.0.0/0 | Internet Gateway |

This is what makes the NAT Gateway reachable from the outside and allows it to forward traffic outward.

### 3. Private Subnet Route Table

This was the critical fix.

Added a new route to the private subnet's route table:

| Destination | Target |
|---|---|
| VPC CIDR (local) | local |
| 0.0.0.0/0 | **NAT Gateway** |

Once this route existed, the private subnet had a valid forwarding path for all internet-bound traffic.

### 4. Security Group — Private Instance

**Inbound rules (restricted):**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Public instance security group |
| All ICMP | ICMP | — | Public instance security group |

No inbound rules allow traffic originating from the internet. Access is only permitted from the trusted public instance, referenced by security group ID rather than a static IP.

**Outbound rules:**

| Type | Protocol | Port | Destination |
|---|---|---|---|
| HTTP | TCP | 80 | 0.0.0.0/0 |
| HTTPS | TCP | 443 | 0.0.0.0/0 |

The instance can initiate outbound web traffic. Return traffic is automatically allowed because security groups track connection state.

---

## Why Both Routing and Security Groups Matter

This lab made the layered nature of AWS networking much more concrete.

### Route table responsibility

The route table determines the traffic *path*. Without a `0.0.0.0/0 → NAT Gateway` entry in the private subnet's route table, there is no forwarding mechanism — the traffic has nowhere to go regardless of what the security group says.

### Security group responsibility

The security group determines whether a protocol and port are *permitted*. Even with the correct NAT route in place, outbound HTTP/HTTPS would still fail if those ports weren't explicitly allowed in the outbound rules.

### The mental model that clicked

For outbound internet access from a private instance, AWS needs all four of these:

1. A route in the private subnet: `0.0.0.0/0 → NAT Gateway`
2. A NAT Gateway deployed in a public subnet with an Elastic IP
3. A route in the public subnet: `0.0.0.0/0 → Internet Gateway`
4. Security group rules permitting the desired outbound traffic

Remove any one of these and the connectivity breaks. All four layers have to be present.

---

## Validation

After updating the route table and confirming the security group rules, the private instance was able to initiate outbound connectivity successfully.

**Before the fix:**
- Security group allowed outbound HTTP/HTTPS
- Private subnet had no route to the NAT Gateway
- Result: traffic failed at the routing layer

**After the fix:**
- Private subnet route table pointed `0.0.0.0/0` to the NAT Gateway
- Security group already had the correct outbound rules
- Result: outbound internet access worked; no direct inbound internet access to the private instance

The architecture behaved exactly as intended.

---

## Key Lessons Learned

**1. Private subnets do not automatically have internet access.**  
A private subnet can only reach the internet if a valid path exists — typically through a NAT Gateway. No path = no connectivity, regardless of security group configuration.

**2. Security groups are not routing tools.**  
Allowing outbound traffic in a security group does not route that traffic anywhere. Routing is handled entirely by route tables at the subnet level.

**3. NAT Gateway enables outbound-only internet access for private resources.**  
This is one of the most common AWS networking patterns. Private instances can initiate connections outward; inbound connections from the internet cannot reach the private instance directly.

**4. "Private" does not mean "isolated."**  
A private EC2 instance can still communicate outward to the internet and inward to trusted resources within the VPC. It is simply not directly exposed to inbound internet traffic.

**5. Security group referencing is more flexible than IP-based rules.**  
Allowing inbound access from another security group — rather than a specific IP address — is cleaner and more maintainable in dynamic environments.

---

## Relevance to AWS SAA Exam

This lab maps directly to several AWS Solutions Architect Associate domains:

| SAA Topic | How It Appeared in This Lab |
|---|---|
| VPC design | Custom VPC with public and private subnets |
| Route tables | Configuring subnet-level routing for public and private paths |
| NAT Gateway | Placement, Elastic IP, and outbound-only behavior |
| Security groups | Stateful rules, port-level control, SG-to-SG referencing |
| Internet Gateway | Enabling internet access for the public subnet |
| Secure network architecture | Isolating private instances while preserving outbound access |

Understanding how these services interact — rather than treating each one in isolation — is exactly the kind of systems thinking the SAA exam tests.

---

## Personal Takeaway

This was the lab where subnet-level routing really clicked for me.

Before this, I understood NAT Gateways conceptually. After this lab, I understand *why* they're placed in public subnets, what breaks without the route table entry, and how security groups and routing work as separate but complementary controls.

The distinction I'll carry forward:

- **Security groups answer:** "Is this traffic allowed?"
- **Route tables answer:** "Where does this traffic go?"

Both questions have to have the right answers for connectivity to work.
