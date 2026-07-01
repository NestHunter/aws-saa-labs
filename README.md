# AWS Labs

Hands-on AWS lab work — built in public as I go.

This repository is a collection of labs, architecture builds, troubleshooting write-ups, and hands-on exercises spanning the full AWS learning path. It started with SAA-C03 prep and will grow into cloud security, infrastructure, and real-world architecture patterns over time.

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

- [Enabling Outbound Internet Access from a Private Subnet with a NAT Gateway](./labs/private-subnet-nat-gateway/README.md)  
  Hands-on lab configuring a NAT Gateway, route tables, and security groups to give a private EC2 instance outbound internet access while keeping it unreachable from the public internet.

- [Network Evaluation Challenge — VPC Troubleshooting Skills Assessment](./labs/network-evaluation-challenge/README.md)  
  Skills assessment diagnosing a broken VPC environment: identifying an inaccessible EC2 instance, correcting an overly restrictive security group, and fixing a missing private subnet route to the NAT Gateway. Covers why assigning a public IP is a workaround rather than a proper design, and what the correct production architecture looks like.

### Serverless & Compute

- [Connect a Lambda Function to a VPC](./labs/connect-lambda-to-vpc/README.md)  
  Hands-on lab deploying a Python Lambda function inside a VPC to call a private EC2 web server and upload the response to S3. Covers VPC attachment, NAT Gateway requirements, ENI-based IAM permissions, IMDSv2 user data, and the full Lambda → EC2 → S3 workflow.  
  → [Step-by-step walkthrough](./labs/connect-lambda-to-vpc/connect-lambda-to-vpc.md)

- [Fix an API Gateway GET Method Using Lambda Proxy Integration](./labs/api-gateway-lambda-proxy-fix/README.md)  
  Troubleshooting lab resolving a broken API Gateway integration after a Swagger import. Covers identifying a misconfigured GET method, recreating it with Lambda proxy integration mapped to `helloWorldFunction`, redeploying to the `prod` stage, and validating the fix through API testing.

- [Build a Serverless Web Application (S3, API Gateway, Lambda, SQS, DynamoDB)](./labs/serverless-web-application/README.md)  
  End-to-end lab building a serverless e-commerce style application. A static S3-hosted frontend submits orders through API Gateway to a producer Lambda, which queues messages in SQS. A consumer Lambda processes the queue and persists records to DynamoDB. Covers event-driven architecture, Lambda proxy integration, SQS decoupling, IAM scoping, and incremental validation. *See also: `serverless-message-board` for a simpler direct build without the SQS async layer.*

- [Serverless Message Board (S3, API Gateway, Lambda, DynamoDB)](./labs/serverless-message-board/README.md)  
  Working end-to-end serverless message board. Browser submits and retrieves messages via a static S3 frontend calling API Gateway, which invokes Lambda (Python/boto3) to read and write DynamoDB. Covers CORS configuration, IAM least-privilege, and real troubleshooting: double-path routing bug, preflight failures, missing PutItem/Scan permissions, and DynamoDB key case mismatch. *See also: `serverless-web-application` for a more complex build that adds SQS async decoupling.*

### Storage

- [Amazon EBS Volumes](./labs/ebs-volume-lab/README.md)  
  Hands-on lab covering the full EBS lifecycle: creating and attaching a 10 GB gp2 volume, formatting and mounting it on Linux, persisting the mount via `/etc/fstab`, taking a snapshot, and restoring it to a second instance in a different Availability Zone. Demonstrates that EBS volumes are AZ-scoped and that snapshots are the standard way to move data between AZs.  
  → [Step-by-step walkthrough](./labs/ebs-volume-lab/amazon-ebs-volumes.md)

### Troubleshooting

- [Troubleshooting an Application Load Balancer and CloudFormation Stack](./troubleshooting/alb-cloudformation-troubleshooting/README.md)  
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
| Lambda — VPC attachment and ENI networking | Documented |
| Lambda — S3 integration via boto3 | Documented |
| EC2 user data and IMDSv2 | Documented |
| API Gateway — REST API, Lambda proxy integration, Swagger import | Documented |
| S3 — static website hosting and bucket policies | Documented |
| SQS — event-driven decoupling and Lambda triggers | Documented |
| Lambda — SQS consumer and DynamoDB integration | Documented |
| End-to-end serverless application architecture | Documented |
| Serverless direct integration (API GW + Lambda + DynamoDB) | Documented |
| CORS configuration for REST APIs | Documented |
| Amazon S3 — object storage concepts | Coming soon |
| IAM — roles, policies, and permissions | Coming soon |
| Route 53 | Coming soon |
| RDS | Coming soon |
| Elastic Load Balancing — configuration | Partially covered (ALB troubleshooting) |
| Auto Scaling | Partially covered (ALB/ASG troubleshooting) |
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
