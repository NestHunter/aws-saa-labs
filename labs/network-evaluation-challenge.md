# AWS Lab: Network Evaluation Challenge — VPC Troubleshooting Skills Assessment

## Overview

This was a different kind of lab.

Instead of configuring a fresh environment from scratch, the objective was to evaluate an existing VPC setup, determine why an EC2 instance couldn't reach the internet, identify what had been built incorrectly from the start, and rebuild it properly.

It tested both hands-on troubleshooting and architectural judgment — the goal wasn't just to "make it work," but to recognize foundational design mistakes and explain how the environment should have been built in a production-ready way.

Core concepts at play:

- EC2 instance accessibility and launch configuration
- Security group egress rules
- Private subnet routing
- NAT Gateway design
- The difference between a quick workaround and a secure architecture

---

## Objective

The assessment required me to:

- Determine whether the original EC2 instance had been provisioned correctly
- Ensure an EC2 instance in the same private subnet could reach the internet
- Verify that outbound traffic from the private subnet was routed correctly out of the VPC
- Validate the solution by running:

```bash
curl https://www.google.com
```

Success was confirmed when the instance returned a large HTML response — proof that outbound internet connectivity was working from the private subnet.

---

## Initial Findings

When I began reviewing the environment, several issues stood out immediately.

### 1. The original EC2 instance had no usable management path

The instance had:
- No public IP address
- No key pair associated with it
- No available EC2 Instance Connect path

That meant there was no practical way to access the instance directly, either for troubleshooting or validation. It was not just misconfigured — it was fundamentally unusable as deployed.

### 2. The security group was too restrictive

The security group associated with the environment only allowed SSH inbound and SSH outbound.

This was a significant problem independent of the routing issue. Even if the route table had been correct, the instance could never have made outbound HTTPS requests like:

```bash
curl https://www.google.com
```

because HTTPS was not permitted in the outbound rules.

### 3. The private subnet route table was missing its default route

The route table for the private subnet had no entry for internet-bound traffic. Without:

```
0.0.0.0/0 → NAT Gateway
```

there was no egress path. Traffic destined for the internet had nowhere to go once it left the instance.

This was the same networking layer issue from earlier NAT Gateway labs — but here it appeared alongside other problems, which is closer to what real-world troubleshooting looks like.

---

## Troubleshooting Process

I worked through the environment layer by layer rather than jumping straight to a single fix.

### Step 1 — Evaluate the original instance

The first task was to assess whether the existing EC2 instance could be rescued or whether it made more sense to replace it.

Given that the instance had no key pair, no public IP, and no EC2 Instance Connect path, there was no practical administrative route into it. Attempting to recover a fundamentally inaccessible instance would have been slower than launching a new one.

Replacing the instance with a properly configured one was the faster and more correct answer — and this was consistent with the assessment's intent.

> This was an important mindset shift: in AWS, it is often faster and cleaner to replace a misconfigured resource than to patch around its original mistakes.

### Step 2 — Correct the security group

I identified that the security group only permitted SSH traffic on both inbound and outbound.

To support the validation test, outbound HTTPS (TCP 443) needed to be allowed. Depending on the broader use case, outbound HTTP (TCP 80) could also be added — but for the purposes of the `curl https://www.google.com` test, HTTPS alone was sufficient.

| Direction | Type | Protocol | Port | Target |
|---|---|---|---|---|
| Outbound | HTTPS | TCP | 443 | 0.0.0.0/0 |

### Step 3 — Fix the private subnet route table

This was the critical networking fix.

The private subnet's route table needed a default route pointing to the NAT Gateway:

| Destination | Target |
|---|---|
| VPC CIDR (local) | local |
| 0.0.0.0/0 | NAT Gateway |

I also verified that the public subnet hosting the NAT Gateway already had a proper default route to the Internet Gateway:

| Destination | Target |
|---|---|
| VPC CIDR (local) | local |
| 0.0.0.0/0 | Internet Gateway |

Both route tables needed to be correct for the full egress path to work.

### Step 4 — Validate connectivity

After correcting the route table and security group, I tested outbound connectivity from the new EC2 instance in the private subnet:

```bash
curl https://www.google.com
```

The initial response was an HTTP redirect. Following the redirect produced a large HTML response from Google — confirming that the private subnet instance could successfully reach the internet outbound.

---

## Root Cause

The environment failed because of a combination of misconfigurations, not a single issue.

| Layer | Problem |
|---|---|
| EC2 launch configuration | No key pair, no public IP, no Instance Connect — instance was not accessible |
| Security group | Only SSH permitted outbound — HTTPS blocked |
| Route table | Private subnet missing `0.0.0.0/0 → NAT Gateway` |

This is one of the more valuable lessons from this type of assessment: AWS troubleshooting often requires validating every layer of the stack together, not just checking one setting at a time.

> Don't assume there is only one problem. Validate the entire path from instance configuration to routing to security controls.

---

## Why Assigning a Public IP Is Not the Right Fix

One of the most important architectural discussions this lab raised was whether simply giving the application instance a public IP was an acceptable solution.

It is not — at least not for a production design.

Assigning a public IP to the application instance makes it directly addressable from the internet. That breaks the security model of a private application subnet. The instance is supposed to be unreachable from inbound internet traffic. A public IP removes that protection entirely.

The correct design preserves the private subnet's isolation:

- The application EC2 instance lives in a private subnet with no public IP
- Outbound internet traffic flows through a NAT Gateway in the public subnet
- The NAT Gateway translates the instance's private IP for outbound connections
- Return traffic is routed back through the NAT Gateway — the internet never initiates a direct connection to the private instance
- Administrative access uses a controlled path: a bastion host, AWS Systems Manager Session Manager, or an EC2 Instance Connect Endpoint

This is the standard AWS pattern for application servers that need outbound access but should never be directly exposed.

---

## Production-Ready Architecture

The correct architecture for this environment:

```
Internet
    │
    ▼
Internet Gateway (attached to VPC)
    │
    ▼
Public Subnet
    ├── NAT Gateway (with Elastic IP)
    └── [Optional] Bastion Host / EC2 Instance Connect Endpoint
    │
    ▼ (via 0.0.0.0/0 → NAT Gateway route in private subnet)
Private Subnet
    └── Application EC2 Instance (no public IP)
```

**Public subnet route table:**

| Destination | Target |
|---|---|
| VPC CIDR (local) | local |
| 0.0.0.0/0 | Internet Gateway |

**Private subnet route table:**

| Destination | Target |
|---|---|
| VPC CIDR (local) | local |
| 0.0.0.0/0 | NAT Gateway |

**Instance design:**
- No public IP
- Outbound HTTPS/HTTP allowed in security group
- Management access via Session Manager, EC2 Instance Connect Endpoint, or a bastion — never direct public exposure

---

## Key Lessons Learned

**1. A broken launch configuration is a separate problem from networking.**  
An instance with no key pair, no public IP, and no Instance Connect path cannot be managed. Fixing the route table won't help if you can't access the instance to validate anything. Instance accessibility needs to be evaluated as part of the troubleshooting chain.

**2. In AWS, replacing a flawed resource is often faster than patching around it.**  
If an EC2 instance was deployed without a key pair or public IP and no other access path exists, launching a correctly configured replacement is a valid and efficient approach. This is different from on-premises thinking, where you might fight to recover a running machine.

**3. Security groups and route tables both have to be correct.**  
This lab made it clear that problems at different layers can compound. A restrictive security group and a missing route are two separate issues — both need to be fixed independently, and fixing one doesn't automatically fix the other.

**4. NAT Gateway placement and route design go together.**  
The NAT Gateway must be in the public subnet. The private subnet must have a default route pointing to it. Both conditions are required. Either one alone is not sufficient.

**5. A public IP is a workaround, not a design.**  
Adding a public IP to get around a broken private subnet design changes the security model of the instance. In any production-like environment, this would be the wrong answer.

---

## Relevance to AWS SAA Exam

This assessment covered several overlapping SAA exam domains at once:

| SAA Topic | How It Appeared in This Assessment |
|---|---|
| VPC design | Identifying and correcting a broken VPC network environment |
| EC2 launch configuration | Recognizing an instance with no viable management path |
| Route tables | Diagnosing a missing default route in the private subnet |
| NAT Gateway | Verifying placement, Elastic IP, and route table linkage |
| Security groups | Identifying overly restrictive outbound rules |
| Internet Gateway | Confirming the public subnet route for NAT Gateway egress |
| Private vs. public subnet behavior | Understanding why public IPs break private subnet security models |
| Troubleshooting methodology | Validating the entire stack rather than assuming a single root cause |

This type of multi-layer scenario — where several things are wrong simultaneously — is closer to real SAA exam scenario questions than single-issue labs. It tests whether you understand how the pieces interact, not just what each piece does individually.

---

## Personal Takeaway

What made this assessment different from straightforward configuration labs was the troubleshooting component.

The environment had been built wrong from the start. The exercise wasn't to follow a checklist and configure things correctly — it was to walk into something broken, figure out all the ways it was broken, and explain both the fix and the reasoning behind why the original design was flawed.

The most valuable part wasn't the `curl` command succeeding. It was being able to answer the underlying question:

**Why is assigning a public IP the wrong answer here — and what does the correct design look like?**

That distinction between a quick workaround and a proper architectural decision is exactly the kind of judgment the AWS Solutions Architect path is trying to build.
