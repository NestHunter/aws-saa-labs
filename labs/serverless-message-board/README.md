# Serverless Message Board Application

> A working end-to-end serverless web application built on AWS. Users submit and retrieve messages through a static frontend backed by API Gateway, Lambda, and DynamoDB.

> **Documentation note:** This lab was built and tested manually in the AWS Console. No project code files were retained in this repository — the configuration steps, request flow, and troubleshooting below reflect exactly what was implemented. No credentials, account IDs, or endpoint URLs are included.

---

## What I Built

This lab is a fully functional serverless message board. A user visits a static HTML page hosted on Amazon S3, types a message, and submits it. The browser makes a `fetch()` call to an Amazon API Gateway endpoint, which triggers an AWS Lambda function. Lambda writes the message to Amazon DynamoDB on POST and scans the table to return all stored messages on GET.

The project works end-to-end from browser to database with no servers to manage. Every component is a managed AWS service.

---

## Architecture

```
Browser (S3 Static Site)
        |
        | HTTPS fetch() to /messages
        v
Amazon API Gateway  (/messages — GET, POST, OPTIONS)
        |
        | Invokes
        v
AWS Lambda  (Python / boto3)
        |
        | PutItem / Scan
        v
Amazon DynamoDB  (Messages table)
```

> See `architecture.png` for a visual diagram.

**Data flow:**
1. Browser loads the static frontend from S3
2. JavaScript calls the API Gateway endpoint
3. API Gateway proxies the request to Lambda
4. Lambda reads or writes to DynamoDB
5. Response flows back to the browser and updates the page

---

## Services Used

| Service | Role |
|---|---|
| **Amazon S3** | Hosts the static frontend with static website hosting enabled |
| **Amazon API Gateway** | Exposes the `/messages` REST endpoint; handles routing and CORS |
| **AWS Lambda** | Backend logic — processes GET and POST requests using Python and `boto3` |
| **Amazon DynamoDB** | NoSQL table that stores submitted messages; partition key: `messageId` |
| **AWS IAM** | Lambda execution role with `dynamodb:PutItem` and `dynamodb:Scan` permissions |

---

## How to Deploy

> Manual deployment via the AWS Console. No IaC tooling required.

### 1. DynamoDB
- Create a table named `Messages`
- Set the partition key to `messageId` (type: String — case must match exactly)

### 2. Lambda
- Create a function with the Python 3.x runtime
- Write the handler directly in the Lambda console's inline code editor: on `POST`, write the incoming message to the table with `put_item`; on `GET`, return all stored messages with `scan`
- Attach an execution role with the following permissions:
  - `dynamodb:PutItem`
  - `dynamodb:Scan`
- Set the `TABLE_NAME` environment variable to `Messages`

### 3. API Gateway
- Create a REST API
- Add a `/messages` resource
- Enable **CORS** on the resource (allow `GET`, `POST`, `OPTIONS`)
- Create `GET` and `POST` methods, both integrated with the Lambda function (Lambda Proxy integration)
- Deploy the API to a stage (e.g., `prod`)
- Copy the **Invoke URL** — it ends at the stage name (e.g., `https://abc123.execute-api.us-east-1.amazonaws.com/prod`)

### 4. Frontend
- In the static HTML page, set `API_BASE_URL` to your stage URL (do **not** append `/messages` here — the JavaScript `fetch()` call already appends it)
- Create an S3 bucket with static website hosting enabled
- Upload the HTML page
- Set the bucket policy to allow public read
- Open the S3 website endpoint in your browser

---

## Challenges and Fixes

These are real issues encountered during this lab — not edge cases, but things that will catch you if you're not paying attention.

### Double path: `/messages/messages`
**Problem:** The API Gateway Invoke URL already contained `/messages` in the base URL, and the JavaScript `fetch()` call appended `/messages` again, resulting in a 404.  
**Fix:** The correct base URL stops at the stage (e.g., `.../prod`). The path `/messages` is appended only once, in the `fetch()` call.

---

### CORS errors on preflight requests
**Problem:** The browser blocked requests with a CORS error. API Gateway was not returning the required `Access-Control-Allow-Origin` headers, and the `OPTIONS` preflight method was not configured.  
**Fix:** CORS was enabled directly on the `/messages` resource in API Gateway, with `GET`, `POST`, and `OPTIONS` all explicitly allowed. The API was redeployed after making the change.

---

### Lambda missing DynamoDB permissions
**Problem:** Lambda returned an `AccessDeniedException` when attempting to write to and read from DynamoDB.  
**Fix:** The Lambda execution role was updated with an inline policy granting `dynamodb:PutItem` and `dynamodb:Scan` on the `Messages` table ARN.

---

### DynamoDB partition key case mismatch
**Problem:** Lambda wrote items using `messageID` (uppercase D) but the DynamoDB table schema defined the partition key as `messageId` (lowercase d). Items were written without error but could not be retrieved correctly.  
**Fix:** The Lambda code and the DynamoDB table definition were aligned to use `messageId` consistently throughout.

---

## Lessons Learned

- **Serverless wiring requires attention to detail at every seam.** API Gateway, Lambda, and DynamoDB each have their own configuration, and a mismatch at any boundary — a path, a key name, a missing header — will silently fail or produce a confusing error.

- **CORS is a browser-level check, not a Lambda issue.** Debugging CORS means looking at API Gateway's method response headers and the `OPTIONS` method, not the Lambda function itself.

- **IAM least-privilege is real.** Lambda has no DynamoDB access by default. You have to explicitly grant it. This is by design and a good habit to understand early.

- **Partition key case sensitivity matters in DynamoDB.** The schema definition and every reference in application code must match exactly — `messageId` and `messageID` are different keys.

- **Build and test incrementally.** Testing the Lambda function directly in the console before wiring it to API Gateway isolated issues faster than debugging the full stack at once.

- **Static site + serverless backend is a legitimate production pattern.** S3 + API Gateway + Lambda + DynamoDB covers a real-world architecture used at scale. This lab is a small but accurate model of that pattern.

---

## Related Labs

- [Build a Serverless Web Application](../serverless-web-application/README.md) — a more complex version of this pattern that adds SQS between Lambda and DynamoDB for asynchronous decoupling. If this lab is where the core API GW → Lambda → DynamoDB integration clicked, that one is the next step.

---

## Notes

- This project was built as a hands-on AWS lab as part of ongoing AWS Solutions Architect Associate preparation.
- No IaC (Terraform, CloudFormation, CDK) was used — all resources were provisioned manually through the AWS Console.
- The purpose was to understand the end-to-end integration of core serverless services, not to automate deployment.
