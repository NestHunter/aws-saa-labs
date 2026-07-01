# Build a Serverless Web Application

## Overview

This lab documents building a fully serverless e-commerce style web application on AWS. The application accepts product purchase submissions through a static frontend, routes them through an API and message queue, and persists them to a database — all without managing any servers.

The architecture follows a classic event-driven serverless pattern: an HTTP request triggers a Lambda function, which decouples the work by placing a message onto an SQS queue, and a second Lambda function processes that message asynchronously and writes the result to DynamoDB. This separation of concerns between the producer and consumer is a foundational pattern in scalable cloud application design.

---

## Architecture

**Request and data flow:**

```
Browser → S3 (static frontend) → API Gateway → Lambda (producer) → SQS → Lambda (consumer) → DynamoDB
```

- A static HTML form hosted in S3 collects product purchase data from the user
- The form submits the data to a REST API endpoint in API Gateway
- API Gateway invokes `productPurchasesSendDataToQueue` via Lambda proxy integration
- That function places the order payload onto the `ProductPurchasesDataQueue` SQS queue
- SQS triggers `productPurchasesDataHandler` as an event source
- The consumer Lambda writes each order as a record in the `ProductPurchases` DynamoDB table

---

## Services Used

| Service | Role in This Lab |
|---|---|
| Amazon S3 | Hosts the static frontend (HTML form) with public website access enabled |
| Amazon API Gateway | Exposes a REST API with a `PUT /productpurchase` method that accepts order submissions |
| AWS Lambda (`productPurchasesSendDataToQueue`) | Receives the API request and places the order data onto the SQS queue |
| Amazon SQS | Decouples the API-facing producer from the database-writing consumer; buffers messages asynchronously |
| AWS Lambda (`productPurchasesDataHandler`) | Consumes messages from the SQS queue and writes each order into DynamoDB |
| Amazon DynamoDB | Persists order records; partition key is `ProductPurchaseKey` |
| AWS IAM | Scoped execution roles and policies for both Lambda functions covering SQS and DynamoDB access |
| AWS CloudShell | Used for CLI-based testing of the SQS queue before the frontend was wired up |

---

## Build Sequence

### 1. Create the DynamoDB table

Created a DynamoDB table with the following configuration:

- **Table name:** `ProductPurchases`
- **Partition key:** `ProductPurchaseKey` (String)
- All other settings left at defaults

This table is the final destination for every order submitted through the application.

### 2. Create the SQS queue

Created a standard SQS queue named `ProductPurchasesDataQueue`. This queue sits between the two Lambda functions, buffering messages from the producer until the consumer is ready to process them. Using SQS here decouples the API response time from the DynamoDB write operation — the producer can return immediately after queuing the message without waiting for persistence to complete.

### 3. Create the consumer Lambda function

Created the first Lambda function, `productPurchasesDataHandler`, which is responsible for reading messages off the queue and writing them to DynamoDB.

- Attached an IAM execution role and policy (incorporating `pushPurchasesToQueue` in its naming) scoped to SQS receive/delete permissions and DynamoDB `PutItem`
- Configured the SQS queue as an event source trigger so Lambda is invoked automatically when messages arrive

### 4. Test the queue and consumer with CloudShell

Before building the frontend, validated the queue-to-DynamoDB path using the AWS CLI in CloudShell:

- Uploaded and extracted the lab resource files in CloudShell
- Sent test messages directly to the queue:

```bash
aws sqs send-message \
  --queue-url <QUEUE-URL> \
  --message-body file://message-body-1.json
```

- Opened the DynamoDB console and confirmed that the test records appeared in the `ProductPurchases` table

This step isolated and verified the consumer path independently before introducing the API and frontend layers.

### 5. Create the producer Lambda function

Created the second Lambda function, `productPurchasesSendDataToQueue`, which sits between API Gateway and the SQS queue.

- Attached an IAM execution role and policy (incorporating `productPurchasesSendMessage` in its naming) scoped to `sqs:SendMessage` on the `ProductPurchasesDataQueue`
- The function receives the incoming order payload from API Gateway and forwards it to the queue

### 6. Create the API Gateway REST API

Built a REST API named `productPurchase` with the following configuration:

- **Resource path:** `/productpurchase`
- **CORS:** enabled on the resource
- **Method:** `PUT`
- **Integration:** Lambda proxy integration targeting `productPurchasesSendDataToQueue`
- **Deployed to stage:** `dev`

The full invoke URL for the frontend takes the form:

```
https://<api-id>.execute-api.<region>.amazonaws.com/dev/productpurchase
```

### 7. Create the S3 static website and test end-to-end

Configured an S3 bucket to serve the frontend:

- **Bucket name:** `product-purchases-webform-XXXX` (unique suffix)
- Disabled Block Public Access to allow static website hosting
- Enabled static website hosting with `index.html` as the index document
- Applied a bucket policy granting public read access
- Synced the frontend files to the bucket using CloudShell:

```bash
aws s3 sync . s3://product-purchases-webform-XXXX
```

Frontend files included:
- `index.html` — the purchase submission form
- `favicon.ico`
- `logo.png`

Opened the S3 website endpoint, submitted a product purchase through the form, then confirmed the new record appeared in the `ProductPurchases` DynamoDB table.

---

## Validation

**Phase 1 — CLI queue test (Step 4)**

Before the frontend existed, the SQS-to-DynamoDB path was validated independently using the AWS CLI in CloudShell. Sending a test message directly to `ProductPurchasesDataQueue` and confirming the record appeared in DynamoDB verified that the consumer Lambda and its IAM permissions were working correctly in isolation.

**Phase 2 — End-to-end application test (Step 7)**

After the API and frontend were in place, the full application path was tested by submitting a purchase through the S3-hosted HTML form. A successful submission resulted in a new item in the `ProductPurchases` DynamoDB table, confirming that every layer — S3 → API Gateway → producer Lambda → SQS → consumer Lambda → DynamoDB — was functioning correctly together.

---

## What This Lab Demonstrates

| Concept | Notes |
|---|---|
| S3 static website hosting | Serving a public frontend directly from S3 with bucket policy-based access control |
| API Gateway REST API design | Creating a resource path, enabling CORS, and configuring a PUT method with Lambda proxy integration |
| Lambda proxy integration | Passing the full HTTP request context from API Gateway to Lambda without transformation |
| SQS event-driven decoupling | Using a queue between producer and consumer to decouple the API response from the database write |
| Lambda as SQS consumer | Configuring SQS as a Lambda event source trigger for automatic message processing |
| DynamoDB persistence | Writing structured records from a serverless function into a NoSQL table |
| IAM scoping for serverless | Creating separate, least-privilege execution roles for each Lambda function based on what it needs to access |
| Incremental validation | Testing the queue-to-database path in isolation before introducing the API and frontend layers |
| End-to-end serverless architecture | Combining five AWS services into a cohesive, fully managed application with no server infrastructure to maintain |

---

## Related Labs

- [Serverless Message Board](../serverless-message-board/README.md) — a simpler version of this pattern that connects API Gateway directly to Lambda and DynamoDB without an SQS queue. A good starting point if you want to understand the core API GW → Lambda → DynamoDB integration before adding async decoupling.
