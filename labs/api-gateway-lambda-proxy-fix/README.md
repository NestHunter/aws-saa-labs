# Fix an API Gateway GET Method Using Lambda Proxy Integration

## Overview

This lab documents a troubleshooting scenario where an API Gateway REST API was imported from a Swagger definition and the `GET /hello` method stopped working. The fix required identifying the misconfiguration introduced by the import and recreating the GET method with the correct integration settings.

The API was originally built manually with a working Lambda proxy integration to the `helloWorldFunction` Lambda function. After deleting and reimporting the API via a Swagger definition, the endpoint failed to return a response. The issue was that the Swagger import did not properly restore the integration in a working state — despite the definition appearing to reference the correct Lambda function.

---

## Problem

After importing the API from a Swagger definition and redeploying to the `prod` stage, a test against the `GET /hello` method failed.

The Swagger definition referenced `helloWorldFunction` via ARN and specified `aws_proxy` as the integration type, but the integration was not functioning correctly once imported. Running a test event in the API Gateway console returned an error rather than the expected JSON response from the Lambda function.

---

## Root Cause

The Swagger import created the GET method and integration configuration, but the integration was not wired correctly enough for API Gateway to invoke the Lambda function. This is a known limitation of importing API Gateway configurations via Swagger: the import may not fully configure all integration properties — particularly the Lambda permission grant that allows API Gateway to invoke the function.

Rather than attempting to patch the imported configuration, the most reliable resolution was to delete the broken GET method entirely and recreate it manually through the console with Lambda proxy integration enabled.

---

## Resolution Steps

1. Open **API Gateway** in the AWS console and navigate to the `myAPI` REST API.
2. Select the `/hello` resource.
3. Select the `GET` method and choose **Delete method**. Confirm the deletion.
4. With `/hello` still selected, choose **Create method**.
5. Set the method type to `GET`.
6. For integration type, select **Lambda Function**.
7. Enable **Lambda proxy integration**.
8. In the Lambda function field, enter or select `helloWorldFunction`.
9. Save the method configuration. Grant API Gateway permission to invoke the Lambda function when prompted.
10. Navigate to **Deploy API**, select the `prod` stage, and redeploy.
11. Run a test against the `GET` method using the console test tool or the invoke URL:

```
https://<your-api-id>.execute-api.<region>.amazonaws.com/prod/hello
```

---

## Validation

Once the GET method was deleted and recreated with Lambda proxy integration correctly enabled and mapped to `helloWorldFunction`, the API test passed immediately. The response returned a `200` status code and a JSON body containing a randomized greeting message from the Lambda function — confirming end-to-end connectivity between API Gateway and Lambda.

---

## What This Lab Demonstrates

| Concept | Notes |
|---|---|
| API Gateway method troubleshooting | Identifying a broken integration and determining the fastest path to resolution |
| Lambda proxy integration | Understanding how `aws_proxy` passes the full HTTP request to Lambda and expects a formatted response back |
| Swagger/OpenAPI import limitations | Imported configurations may not fully reproduce all integration properties, particularly Lambda invoke permissions |
| Correct GET method configuration | The integration type, proxy setting, and function reference all need to be explicitly correct |
| Service integration debugging | Distinguishing between a routing problem (method/resource) and an integration problem (Lambda binding) |
| Validation through API testing | Using the built-in API Gateway test tool and the live invoke URL to confirm the fix |
