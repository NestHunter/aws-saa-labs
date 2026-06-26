# Lab: Connect a Lambda Function to a VPC

This lab demonstrates how to deploy an AWS Lambda function inside a VPC so it can communicate with private resources — in this case, an EC2 instance running a web server — and write the result to Amazon S3.

The detailed step-by-step walkthrough is in [connect-lambda-to-vpc.md](./connect-lambda-to-vpc.md).

---

## Prerequisites

- An AWS account with permissions to create VPCs, EC2 instances, Lambda functions, IAM roles, and S3 buckets
- Basic familiarity with the AWS Management Console
- An existing S3 bucket (or permissions to create one) to receive the uploaded output
- Basic understanding of Python (the Lambda function is written in Python)

---

## Lab Steps (High Level)

### Part 1 — Build the network

1. Use the **VPC and more** wizard to create a VPC with:
   - 2 public subnets and 2 private subnets
   - 1 NAT Gateway (required so Lambda can reach the internet and AWS APIs from within the VPC)
2. Create a **security group** that allows inbound HTTP traffic.

### Part 2 — Configure IAM permissions

3. Attach an inline IAM policy to the Lambda execution role that grants permission to create and manage **Elastic Network Interfaces (ENIs)** — required for VPC-attached Lambda functions.
4. Attach an additional policy granting **S3 write access** (`s3:PutObject`) so the function can upload files to a bucket.

### Part 3 — Create and attach the Lambda function to the VPC

5. Create a Python-based Lambda function named `simpleLambda`.
6. Under the VPC configuration, select the VPC created in Part 1, the **private subnets**, and the security group.
7. Deploy a basic test version of the function and invoke it to confirm there are no VPC connectivity errors.

### Part 4 — Launch the EC2 web server

8. Launch an EC2 instance in one of the **private subnets** with user data that installs Apache and serves EC2 instance metadata (Instance ID and Availability Zone) as an HTML page.
9. Note the instance's **private IP address** — this is what Lambda will use to call it.

### Part 5 — Update Lambda and validate end-to-end

10. Update the Lambda function code to:
    - Make an HTTP GET request to the EC2 instance's private IP
    - Upload the HTML response as a text file to the S3 bucket
11. Invoke the updated function and confirm it returns a success message.
12. Navigate to the S3 bucket and verify the uploaded object contains the EC2 metadata response.

---

## What This Lab Demonstrates

| Concept | Why It Matters |
|---|---|
| Lambda VPC attachment | By default, Lambda runs outside any VPC and cannot reach private resources; attaching it to a VPC gives it access to private subnets |
| NAT Gateway requirement | A VPC-attached Lambda loses its default internet access; a NAT Gateway in a public subnet restores outbound connectivity to AWS APIs and the internet |
| ENI-based networking | Lambda creates Elastic Network Interfaces to communicate within the VPC; the execution role must have permission to manage them |
| Private EC2 communication | Lambda can call resources by private IP when both are in the same VPC, without exposing the EC2 instance publicly |
| Lambda writing to S3 | Lambda needs an explicit IAM permission (`s3:PutObject`) to write objects — S3 is a public AWS service reached via the NAT Gateway from within the VPC |
| End-to-end serverless workflow | This pattern — Lambda → private resource → S3 — is a common real-world design for data collection, processing pipelines, and internal API calls |

---

## Related Lab Notes

- [Enabling Outbound Internet Access from a Private Subnet with a NAT Gateway](../private-subnet-nat-gateway/README.md)
- [Network Evaluation Challenge — VPC Troubleshooting Skills Assessment](../network-evaluation-challenge/README.md)
- [Amazon EBS Volumes](../ebs-volume-lab/README.md)
