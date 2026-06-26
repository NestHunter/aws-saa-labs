# Connect a Lambda Function to a VPC — Step-by-Step Walkthrough

This document contains the detailed steps for this lab. For a high-level overview, see [README.md](./README.md).

---

## Why This Lab Matters

By default, a Lambda function runs in an AWS-managed environment outside of any customer VPC. This means it can reach public AWS services (like S3) directly, but it cannot communicate with resources in a private subnet — EC2 instances, RDS databases, ElastiCache clusters, or anything else that doesn't have a public endpoint.

Attaching Lambda to a VPC solves this, but introduces a tradeoff: the function now routes all traffic through your VPC, including calls to public AWS APIs. A NAT Gateway in a public subnet is required to restore that outbound connectivity.

This lab builds the full chain: VPC → private Lambda → private EC2 web server → S3.

---

## Part 1 — Build the Network

### Step 1: Create a VPC using the wizard

1. Open the **VPC Console** and choose **Create VPC**.
2. Select **VPC and more** to use the guided wizard.
3. Configure the following:
   - **Name:** choose something descriptive (e.g., `lambda-vpc-lab`)
   - **IPv4 CIDR block:** accept the default or choose your own
   - **Availability Zones:** 2
   - **Public subnets:** 2
   - **Private subnets:** 2
   - **NAT Gateways:** 1 (in 1 AZ)
   - **S3 Gateway endpoint:** not required for this lab
4. Choose **Create VPC** and wait for all resources to be provisioned.

> The NAT Gateway is essential. Without it, a Lambda function attached to private subnets has no path to reach AWS APIs — including S3 — or the public internet.

### Step 2: Create a security group

1. In the VPC Console, navigate to **Security Groups** and choose **Create security group**.
2. Associate it with the VPC created above.
3. Add an **inbound rule** allowing HTTP (port 80) from any source (`0.0.0.0/0`).
4. Leave the default outbound rule (allow all) in place.
5. Name it something identifiable (e.g., `lambda-lab-sg`).

---

## Part 2 — Configure IAM Permissions

Lambda functions attached to a VPC need permission to create and manage **Elastic Network Interfaces (ENIs)** — this is how Lambda places itself inside the VPC. Without this, the function will fail to attach or invoke.

### Step 3: Add the ENI management policy

Navigate to the Lambda execution role in **IAM → Roles** and attach the following inline policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeNetworkInterfaces",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface"
            ],
            "Resource": "*"
        }
    ]
}
```

### Step 4: Add S3 write permissions

Attach an additional policy (inline or managed) granting the Lambda role permission to write to S3:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
        }
    ]
}
```

Replace `YOUR-BUCKET-NAME` with the name of your S3 bucket. Scoping the resource to a specific bucket is better practice than using `*`.

---

## Part 3 — Create the Lambda Function and Attach It to the VPC

### Step 5: Create the Lambda function

1. Open the **Lambda Console** and choose **Create function**.
2. Select **Author from scratch**.
3. Configure:
   - **Function name:** `simpleLambda`
   - **Runtime:** Python (latest available version)
   - **Execution role:** use an existing role or create a new one — ensure the ENI and S3 policies from Part 2 are attached
4. Choose **Create function**.

### Step 6: Attach the function to the VPC

1. In the function's configuration, open the **VPC** tab.
2. Select the VPC created in Step 1.
3. Select the **private subnets** (not the public ones).
4. Select the security group created in Step 2.
5. Choose **Save**.

> VPC attachment can take a minute or two to apply. Lambda provisions ENIs during this process — this is why the IAM permissions in Step 3 are required before saving.

### Step 7: Run an initial test

Invoke the function using the **Test** tab in the console with a default empty event. Confirm the invocation succeeds and that CloudWatch Logs show no VPC connectivity errors. At this stage the function just runs the default placeholder code — the goal is to confirm the VPC wiring is correct before adding logic.

---

## Part 4 — Launch the EC2 Web Server

### Step 8: Launch an EC2 instance in a private subnet

1. Open the **EC2 Console** and choose **Launch instance**.
2. Select **Amazon Linux 2** (or Amazon Linux 2023).
3. Choose an instance type — `t2.micro` or `t3.micro` is sufficient.
4. Under **Network settings**, select:
   - The VPC created in Step 1
   - One of the **private subnets**
   - **Auto-assign Public IP:** disabled (this instance will not be publicly accessible)
5. Attach the security group from Step 2.
6. Under **Advanced details → User data**, paste the following script:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AVAILABILITY_ZONE=$(curl -s \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>EC2 Metadata</title>
</head>
<body>
    <h1>EC2 Instance Metadata</h1>
    <p>Instance ID: ${INSTANCE_ID}</p>
    <p>Availability Zone: ${AVAILABILITY_ZONE}</p>
</body>
</html>
EOF

chown apache:apache /var/www/html/index.html
```

This script installs Apache, retrieves the instance's own metadata using IMDSv2 (the token-based method), and writes a simple HTML page displaying the Instance ID and Availability Zone.

7. Launch the instance and wait for it to reach the **running** state.
8. Note the instance's **private IP address** from the EC2 Console — you will need it in the next step.

---

## Part 5 — Update Lambda and Validate End-to-End

### Step 9: Update the Lambda function code

Replace the default function code with the following Python script. Update the three variables at the top with your own values before deploying.

```python
import boto3
import http.client

s3 = boto3.client('s3')

# Replace these values with your own
EC2_PRIVATE_IP = '10.0.x.x'        # Private IP of the EC2 instance from Step 8
S3_BUCKET_NAME = 'your-bucket-name' # Name of your S3 bucket
S3_OBJECT_KEY  = 'web-server-message.txt'

def get_web_server_message():
    conn = http.client.HTTPConnection(EC2_PRIVATE_IP)
    conn.request("GET", "/")
    response = conn.getresponse()
    data = response.read().decode("utf-8")
    conn.close()
    return data

def upload_to_s3(message):
    try:
        response = s3.put_object(
            Bucket=S3_BUCKET_NAME,
            Key=S3_OBJECT_KEY,
            Body=message
        )
        print("Uploaded to S3:", response)
    except Exception as error:
        print("Upload error:", error)
        raise error

def lambda_handler(event, context):
    print("Lambda function invoked")
    try:
        message = get_web_server_message()
        print("Response from EC2:", message)
        upload_to_s3(message)
        return "Document uploaded to S3 successfully!"
    except Exception as error:
        print("Error:", error)
        raise error
```

**What this code does:**
- `get_web_server_message()` opens an HTTP connection to the EC2 instance's private IP on port 80 and reads the HTML response
- `upload_to_s3()` writes that response as a text object to the specified S3 bucket
- `lambda_handler()` orchestrates the two steps and returns a success string on completion

Choose **Deploy** to save the updated code.

### Step 10: Test the updated function

1. In the Lambda Console, open the **Test** tab.
2. Use a default empty event (`{}`) and choose **Test**.
3. Confirm the execution result shows `"Document uploaded to S3 successfully!"`.
4. Check the **CloudWatch Logs** for the function to see the printed output, including the raw HTML response from the EC2 instance.

### Step 11: Verify the S3 upload

1. Open the **S3 Console** and navigate to your bucket.
2. Confirm that `web-server-message.txt` (or whatever key name you set in `S3_OBJECT_KEY`) is present.
3. Download or preview the object — it should contain the HTML page with the EC2 Instance ID and Availability Zone served by the Apache web server on the private EC2 instance.

---

## How the Full Flow Works

```
Lambda (private subnet)
    │
    │  HTTP GET to EC2 private IP
    ▼
EC2 web server (private subnet, Apache)
    │
    │  Returns HTML with Instance ID + AZ
    ▼
Lambda
    │
    │  s3.put_object()
    │  (routed via NAT Gateway → Internet Gateway → S3 public endpoint)
    ▼
S3 Bucket
    └── web-server-message.txt
```

> Lambda reaches S3 via the NAT Gateway because S3 is a public AWS service. If you want to avoid NAT Gateway data transfer costs on the S3 path in production, you can add a **VPC Gateway Endpoint for S3**, which routes S3 traffic over the AWS private network instead.

---

## Cleanup

To avoid ongoing charges after completing the lab:

1. **Delete the Lambda function** from the Lambda Console.
2. **Terminate the EC2 instance** from the EC2 Console.
3. **Delete the S3 object** and optionally the bucket if it was created for this lab.
4. **Delete the NAT Gateway** (NAT Gateways have an hourly cost — delete this first before removing the VPC).
5. **Release the Elastic IP** associated with the NAT Gateway.
6. **Delete the VPC** and its associated subnets, route tables, and internet gateway.

---

## Relevance to AWS SAA Exam

| SAA Topic | How It Appeared in This Lab |
|---|---|
| Lambda VPC configuration | Attaching Lambda to private subnets and a security group |
| Elastic Network Interfaces | Understanding why IAM ENI permissions are required for VPC-attached Lambda |
| NAT Gateway | Restoring outbound internet and AWS API access from a private Lambda function |
| IAM permissions for Lambda | Scoped policies for ENI management and S3 writes |
| EC2 user data | Bootstrapping a web server with Apache and IMDSv2 metadata at launch |
| IMDSv2 | Token-based metadata retrieval — the current AWS recommended approach |
| S3 object uploads via SDK | Using `boto3` to call `put_object` from Lambda |
| VPC Gateway Endpoints | Optional optimization to route S3 traffic privately without NAT |
| Private resource communication | Lambda reaching an EC2 instance by private IP within the same VPC |
