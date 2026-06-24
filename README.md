# AWS SAA Labs

Welcome to my AWS Solutions Architect Associate lab repository.

This repository is a collection of labs, notes, small builds, and hands-on exercises created while studying for the AWS Certified Solutions Architect – Associate certification and building my broader cloud security skill set.

## Purpose

The goal of this repo is to turn study time into visible proof of progress.

Instead of only consuming AWS course material, I'm using this space to document what I'm learning, capture key takeaways, and build hands-on familiarity with AWS services in a practical way.

## What You'll Find Here

This repository may include:

- Lab walkthroughs
- Study notes
- Architecture summaries
- Small AWS builds
- Service-specific examples
- Security considerations tied to each topic
- Lessons learned from implementation

---

## Lab Notes

### Networking & VPC

- [Enabling Outbound Internet Access from a Private Subnet with a NAT Gateway](./labs/private-subnet-nat-gateway.md)  
  Hands-on lab configuring a NAT Gateway, route tables, and security groups to give a private EC2 instance outbound internet access while keeping it unreachable from the public internet.

- [Network Evaluation Challenge — VPC Troubleshooting Skills Assessment](./labs/network-evaluation-challenge.md)  
  Skills assessment diagnosing a broken VPC environment: identifying an inaccessible EC2 instance, correcting an overly restrictive security group, and fixing a missing private subnet route to the NAT Gateway. Covers why assigning a public IP is a workaround rather than a proper design, and what the correct production architecture looks like.

### Storage

- [Amazon EBS Volumes](./ebs-volume-lab/README.md)  
  Hands-on lab covering the full EBS lifecycle: creating and attaching a 10 GB gp2 volume, formatting and mounting it on Linux, persisting the mount via `/etc/fstab`, taking a snapshot, and restoring it to a second instance in a different Availability Zone. Demonstrates that EBS volumes are AZ-scoped and that snapshots are the standard way to move data between AZs.  
  → [Step-by-step walkthrough](./ebs-volume-lab/amazon-ebs-volumes.md)

### Troubleshooting

- [Troubleshooting an Application Load Balancer and CloudFormation Stack](./troubleshooting/alb-cloudformation-troubleshooting.md)  
  Diagnosing why a web application wasn't loading through an ALB and fixing issues with Auto Scaling Group attachment, subnet placement, and security group rules.

---

## Topics Covered

| Topic | Status |
|---|---|
| VPC — subnets, route tables, Internet Gateway | Documented |
| NAT Gateway — private subnet outbound access | Documented |
| Security groups — stateful rules, SG-to-SG referencing | Documented |
| EC2 accessibility — key pairs, Instance Connect, management paths | Documented |
| VPC troubleshooting — multi-layer diagnosis | Documented |
| Application Load Balancer — troubleshooting | Documented |
| EBS volumes — create, attach, format, mount, fstab | Documented |
| EBS snapshots — cross-AZ data movement | Documented |
| Amazon S3 | Coming soon |
| IAM | Coming soon |
| EC2 | Coming soon |
| Route 53 | Coming soon |
| RDS | Coming soon |
| Elastic Load Balancing — configuration | Coming soon |
| Auto Scaling | Coming soon |
| CloudWatch | Coming soon |

---

## Why I'm Building This Publicly

I want my GitHub to reflect more than just completed projects.

This repo shows how I think, how I learn, and how I break down AWS concepts into practical understanding. It also gives me a place to document growth over time as I move from foundational architecture knowledge into cloud security and real-world implementations.

## Learning Approach

My approach is simple:

- Learn the concept  
- Build or document something from it  
- Capture what matters  
- Revisit and improve over time  

The focus here is not perfection — it's consistency, clarity, and progress.

## Long-Term Goal

This repository supports my path toward becoming a Cloud Security Engineer by helping me build stronger AWS fundamentals, document my hands-on learning, and create a public body of work I can continue building on.

## Related Repositories

- [cloud-security-portfolio](https://github.com/NestHunter/cloud-security-portfolio) — central portfolio hub  
- Future project repos tied to AWS architecture, Terraform, and cloud security labs
