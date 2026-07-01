# Troubleshooting an Application Load Balancer in AWS

## Overview

In this skills assessment, I deployed an AWS environment using CloudFormation and had to diagnose why the application was not loading successfully through the Application Load Balancer (ALB).

The goal was not just to launch the stack, but to identify the infrastructure issues preventing the application from being accessible.

## Problem

The CloudFormation stack deployed, but the website was not loading successfully through the ALB.

## What I Found

After reviewing the environment, I identified three main issues:

### 1. Auto Scaling Group was not attached to the load balancer

The EC2 instances created by the Auto Scaling Group were not being dynamically assigned to the Application Load Balancer.

**Fix:**  
I updated the Auto Scaling Group to attach to the correct Application Load Balancer / target group so traffic could be routed to the instances properly.

### 2. Load balancer was associated with the wrong subnet placement

The load balancer was pointing to a private VPC/subnet placement instead of the public-facing one.

**Fix:**  
I updated the load balancer configuration so it was associated with the public-facing network path required to accept external traffic.

### 3. Security group rules did not allow HTTP traffic

The security group configuration was too restrictive for web traffic.

**Fix:**  
I updated the security group to allow inbound traffic on port 80 so the application could be reached through HTTP.

## Outcome

After correcting these issues, the application loaded successfully through the Application Load Balancer and the site displayed the success message from the lab.

## Key Lessons Learned

- Load balancers must be correctly associated with target groups and Auto Scaling Groups.
- Public-facing applications require correct subnet and networking placement.
- Security groups can silently block application access even when the infrastructure appears to be deployed correctly.
- Troubleshooting in AWS often requires checking multiple layers: compute, networking, load balancing, and security.

## Skills Practiced

- AWS CloudFormation
- Application Load Balancer troubleshooting
- Auto Scaling Group configuration
- VPC and subnet awareness
- Security group troubleshooting
- Root cause analysis in AWS

## Reflection

This lab helped reinforce that successful cloud deployments are not just about provisioning resources — they also depend on how those resources are connected, exposed, and secured.
