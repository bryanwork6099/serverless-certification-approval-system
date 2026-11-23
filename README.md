# Serverless Certification Approval System

A serverless application built on AWS that automates certification funding requests through a human-in-the-loop approval workflow.

The solution uses AWS Lambda, API Gateway, Step Functions, and DynamoDB to manage request submissions, approval decisions, and real-time status tracking without managing servers.

## Project Overview

This project demonstrates how to build an event-driven, serverless approval system using AWS services.

Users can submit certification requests containing their name, course, and associated cost. Once submitted, the application automatically starts an approval workflow, stores the request, and waits for a manager's decision.

Managers can approve or reject requests through an HTTP API. AWS Step Functions then resumes the paused workflow and updates the request status in DynamoDB.

### Key Features

- **Request Submission:** Submit certification funding requests through an HTTP API.
- **Workflow Orchestration:** Automate the approval process using AWS Step Functions.
- **Human-in-the-Loop Approval:** Pause workflow execution until a manager provides a decision.
- **Callback Integration:** Resume paused workflows using the Step Functions task-token mechanism.
- **Status Tracking:** Retrieve request details and approval status from DynamoDB.
- **Serverless Architecture:** Execute application logic without provisioning or managing servers.

## Architecture

### AWS Architecture Diagram

```mermaid
flowchart TD
    Client[Client / PowerShell / cURL]
    APIGW[Amazon API Gateway]

    Submit[SubmitRequestFunction]
    Approval[HandleApprovalFunction]
    Status[CheckStatusFunction]
    Notify[NotifyManagerFunction]

    SFN[AWS Step Functions]

    DDB[(Amazon DynamoDB)]
    CW[Amazon CloudWatch Logs]

    Client -->|POST /request| APIGW
    Client -->|POST /approval| APIGW
    Client -->|GET /request/{requestId}| APIGW

    APIGW --> Submit
    APIGW --> Approval
    APIGW --> Status

    Submit -->|Start Execution| SFN

    SFN -->|Save Request| DDB
    SFN -->|Invoke with Task Token| Notify

    Notify -.->|Log Token - Demo Only| CW

    Approval -->|SendTaskSuccess| SFN

    SFN -->|Update Final Status| DDB

    Status -->|Retrieve Request| DDB
```

### AWS Services Used

| Service | Purpose |
|---|---|
| AWS Lambda | Executes application logic for request submission, notification, approval handling, and status retrieval. |
| Amazon API Gateway | Exposes HTTP endpoints for client interactions. |
| AWS Step Functions | Orchestrates the approval workflow and manages callback-based waiting. |
| Amazon DynamoDB | Stores certification request information and approval statuses. |
| Amazon CloudWatch | Collects Lambda execution logs and supports troubleshooting. |
| AWS IAM | Manages permissions between AWS services. |

## Project Structure

```text
Serverless-Certification-Approval-System/
│
├── src/
│   ├── submit_request.py
│   ├── notify_manager.py
│   ├── handle_approval.py
│   └── check_status.py
│
├── step-functions-definition.json
│
├── README.md
│
└── EXPLANATION.md
```

## Prerequisites

Before deploying the project, ensure you have:

- An active AWS account.
- Access to the AWS Management Console.
- Permission to create Lambda functions, IAM roles, API Gateway APIs, Step Functions state machines, and DynamoDB tables.
- Basic knowledge of Python and AWS services.

**Recommended:** AWS CLI and PowerShell or cURL for testing the API endpoints.

## Deployment Guide

This guide walks through deploying the infrastructure manually through the AWS Management Console.

### Step 1: Create DynamoDB Table

1. Open the **Amazon DynamoDB** console.
2. Select **Tables** and click **Create table**.
3. Configure the following:

| Setting | Value |
|---|---|
| Table name | `CertificationRequests` |
| Partition key | `requestId` |
| Partition key type | String |

4. Leave the remaining settings at their defaults.
5. Click **Create table**.
6. Wait until the table status becomes **Active**.

### Step 2: Create IAM Roles

The application requires permissions for Lambda functions and Step Functions to communicate with other AWS services.

For better security, use separate execution roles.

#### 2.1 Lambda Execution Role

1. Open the **IAM** console.
2. Navigate to **Roles** → **Create role**.
3. Select **AWS service**.
4. Choose **Lambda** as the use case.
5. Attach the following policies:

   - `AWSLambdaBasicExecutionRole`

6. Name the role:

   `CertificationLambdaRole`

7. Create the role.

Additional permissions must be granted according to each function's responsibilities:

| Function | Permission | Resource |
|---|---|---|
| SubmitRequestFunction | `states:StartExecution` | Approval state machine ARN |
| HandleApprovalFunction | `states:SendTaskSuccess` | Appropriate state machine resources |
| CheckStatusFunction | `dynamodb:GetItem` | CertificationRequests table |
| NotifyManagerFunction | CloudWatch Logs | Its log group |

For a quick demonstration, these permissions can be added through separate inline policies or appropriately scoped managed policies. Avoid granting broad full-access permissions in production.

#### 2.2 Step Functions Execution Role

Step Functions also requires an independent IAM execution role.

Create a role with the AWS Step Functions service as its trusted entity.

The role should permit:

- `lambda:InvokeFunction` on `NotifyManagerFunction`.
- `dynamodb:PutItem` on `CertificationRequests`.
- `dynamodb:UpdateItem` on `CertificationRequests`.

Name the role:

`CertificationStepFunctionsRole`

Ensure the state machine uses this role when it is created.

### Step 3: Create Lambda Functions

Navigate to the **AWS Lambda** console.

Create four functions using the following configuration.

**General Configuration**

| Setting | Value |
|---|---|
| Function names | Listed below |
| Runtime | A currently supported Python runtime |
| Architecture | x86_64 |
| Execution role | `CertificationLambdaRole` |

Use the appropriate source code from the `src/` directory.

#### 3.1 SubmitRequestFunction

1. Create a Lambda function named `SubmitRequestFunction`.
2. Select the appropriate Python runtime.
3. Assign `CertificationLambdaRole`.
4. Copy the contents of `src/submit_request.py` into the code editor.
5. Navigate to **Configuration** → **Environment variables**.
6. Add:

| Key | Value |
|---|---|
| `STATE_MACHINE_ARN` | Your Step Functions state machine ARN |

The state machine ARN will be available after Step 4.

You can temporarily create the function without the environment variable and configure it after creating the state machine.

#### 3.2 NotifyManagerFunction

1. Create a function named `NotifyManagerFunction`.
2. Assign `CertificationLambdaRole`.
3. Copy the contents of `src/notify_manager.py`.
4. Deploy the function.

This function receives the callback task token from Step Functions and logs it for the demonstration.

#### 3.3 HandleApprovalFunction

1. Create a function named `HandleApprovalFunction`.
2. Assign `CertificationLambdaRole`.
3. Copy the contents of `src/handle_approval.py`.
4. Deploy the function.

This function processes approval decisions and resumes the waiting Step Functions execution.

#### 3.4 CheckStatusFunction

1. Create a function named `CheckStatusFunction`.
2. Assign `CertificationLambdaRole`.
3. Copy the contents of `src/check_status.py`.
4. Navigate to **Configuration** → **Environment variables**.
5. Add:

| Key | Value |
|---|---|
| `TABLE_NAME` | `CertificationRequests` |

6. Deploy the function.

### Step 4: Create Step Functions State Machine

1. Open the **AWS Step Functions** console.
2. Select **State machines**.
3. Click **Create state machine**.
4. Select **Write your workflow in code** or the equivalent code-based creation option.
5. Select **Standard** as the workflow type.
6. Open `step-functions-definition.json` from the project repository.
7. Copy the workflow definition into the editor.

#### Update Resource Placeholders

Before creating the state machine, replace the placeholders in the JSON definition with your actual AWS resource names or ARNs.

| Placeholder | Replacement |
|---|---|
| `${DynamoDBTableName}` | `CertificationRequests` |
| `${NotifyManagerFunctionName}` | `NotifyManagerFunction` or its full Lambda ARN |

Check that the table name and Lambda integration reference are correctly configured in every relevant state.

8. Click **Next**.
9. Set the state machine name to:

   `ApprovalStateMachine`

10. Configure the Step Functions execution role using `CertificationStepFunctionsRole`.
11. Review the permissions and create the state machine.
12. Copy the state machine ARN.

**Important:** Use a Standard Workflow because this application requires callback-based waiting for human approval.

### Step 5: Update Lambda Configuration

Now that the state machine has been created, configure the request submission Lambda.

1. Open the **AWS Lambda** console.
2. Select `SubmitRequestFunction`.
3. Navigate to **Configuration** → **Environment variables**.
4. Edit `STATE_MACHINE_ARN`.
5. Enter the ARN copied from Step 4.
6. Save the configuration.

The Lambda function can now start the approval workflow.

### Step 6: Create API Gateway HTTP API

1. Open the **Amazon API Gateway** console.
2. Select **Create API**.
3. Choose **HTTP API**.
4. Click **Build**.
5. Set the API name:

   `CertificationAPI`

#### Configure Integrations

Create Lambda integrations for:

- `SubmitRequestFunction`
- `HandleApprovalFunction`
- `CheckStatusFunction`

#### Configure Routes

Create the following routes and associate each with its corresponding Lambda integration.

| Method | Route | Integration |
|---|---|---|
| POST | `/request` | SubmitRequestFunction |
| POST | `/approval` | HandleApprovalFunction |
| GET | `/request/{requestId}` | CheckStatusFunction |

#### Configure Stage

1. Select the `$default` stage.
2. Enable automatic deployment if required.
3. Review the API configuration.
4. Click **Create**.

Copy the API Invoke URL.

Example:

```text
https://xxxxxxxx.execute-api.ap-southeast-1.amazonaws.com
```

Your application infrastructure is now deployed.

---

## Verification and Testing

The following tests demonstrate the complete certification approval lifecycle.

### Step 1: Submit a Certification Request

Send a POST request to the `/request` endpoint.

**Example Request:**

```json
{
  "name": "Alice",
  "course": "AWS Certified Developer",
  "cost": 150
}
```

**PowerShell Example:**

```powershell
$ApiUrl = "https://YOUR-API-URL"

$body = @{
    name   = "Alice"
    course = "AWS Certified Developer"
    cost   = 150
} | ConvertTo-Json -Compress

Invoke-RestMethod `
    -Uri "$ApiUrl/request" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body
```

**Expected Response:**

```json
{
  "requestId": "generated-request-id",
  "executionArn": "step-functions-execution-arn"
}
```

Save the returned `requestId` for the subsequent tests.

### Step 2: Retrieve the Approval Task Token

1. Open the **Amazon CloudWatch** console.
2. Navigate to **Logs** → **Log groups**.
3. Open the log group for:

   `/aws/lambda/NotifyManagerFunction`

4. Open the latest log stream.
5. Locate the log entry containing:

```text
APPROVAL TOKEN: ...
```

6. Copy the task token for the submitted request.

**Note:** The token is required to resume the corresponding waiting workflow. Keep it confidential and do not share it publicly.

### Step 3: Check the Initial Request Status

Send a GET request using the previously generated request ID.

**PowerShell Example:**

```powershell
$RequestId = "YOUR-REQUEST-ID"

Invoke-RestMethod `
    -Uri "$ApiUrl/request/$RequestId" `
    -Method GET
```

**Expected Response:**

```json
{
  "requestId": "generated-request-id",
  "name": "Alice",
  "course": "AWS Certified Developer",
  "cost": 150,
  "status": "PENDING"
}
```

The request should remain in `PENDING` while Step Functions waits for the manager's decision.

### Step 4: Approve the Certification Request

Submit the manager's approval decision through the `/approval` endpoint.

**Example Request:**

```json
{
  "requestId": "YOUR-REQUEST-ID",
  "decision": "APPROVED",
  "taskToken": "YOUR-TASK-TOKEN"
}
```

**PowerShell Example:**

```powershell
$approvalBody = @{
    requestId = $RequestId
    decision  = "APPROVED"
    taskToken = "YOUR-TASK-TOKEN"
} | ConvertTo-Json -Compress

Invoke-RestMethod `
    -Uri "$ApiUrl/approval" `
    -Method POST `
    -ContentType "application/json" `
    -Body $approvalBody
```

The approval Lambda sends the callback response to Step Functions using `SendTaskSuccess`.

The workflow resumes and updates the DynamoDB record.

### Step 5: Verify the Final Approval Status

Send another GET request to retrieve the updated request.

```powershell
Invoke-RestMethod `
    -Uri "$ApiUrl/request/$RequestId" `
    -Method GET
```

**Expected Result:**

```json
{
  "requestId": "generated-request-id",
  "name": "Alice",
  "course": "AWS Certified Developer",
  "cost": 150,
  "status": "APPROVED"
}
```

<Note>The exact response fields depend on the implementation of the Lambda functions. The examples illustrate the expected application behavior.</Note>

### Step 6: Test the Rejection Workflow

To test the rejection path, submit another certification request and repeat the approval process using:

```json
{
  "requestId": "YOUR-REQUEST-ID",
  "decision": "REJECTED",
  "taskToken": "YOUR-TASK-TOKEN"
}
```

Verify that the final DynamoDB status becomes:

```text
REJECTED
```

This confirms that both decision branches in the Step Functions state machine work correctly.

---

## Troubleshooting

### 1. Task Timed Out Error

**Possible Cause:**

The Step Functions task exceeded its configured timeout, or the execution expired before the callback was received.

**Solution:**

1. Open the Step Functions console.
2. Check the state machine type and confirm it is a Standard Workflow.
3. Inspect the execution history.
4. Check whether the `WaitForApproval` state timed out.
5. Submit a new certification request and retrieve its corresponding task token.

Remember that a task token belongs to a specific waiting task. A token from another request cannot resume the new execution.

### 2. Invalid or Expired Task Token

**Possible Causes:**

- The token was copied incorrectly.
- The token belongs to another request.
- The waiting task has already completed.
- The waiting task or workflow has timed out.

**Solution:**

Retrieve the correct token associated with the current execution and verify that the workflow is still waiting for approval.

### 3. Lambda AccessDeniedException

**Possible Cause:**

The Lambda execution role does not have the required IAM permissions.

**Solution:**

Check the function's execution role and verify that the necessary permissions are attached.

Examples include:

- `states:StartExecution`
- `states:SendTaskSuccess`
- `dynamodb:GetItem`

Also verify the Step Functions execution role has the necessary Lambda invocation and DynamoDB permissions.

### 4. DynamoDB Item Not Found

**Possible Causes:**

- The request ID is incorrect.
- The initial DynamoDB write failed.
- The Lambda function is configured with the wrong table name.

**Solution:**

1. Open DynamoDB.
2. Select `CertificationRequests`.
3. Navigate to **Explore table items**.
4. Search using the request ID.
5. Check the Lambda execution logs and Step Functions execution history if the item is missing.

### 5. API Gateway Internal Server Error

**Possible Causes:**

- Lambda execution errors.
- Incorrect API Gateway integration.
- Missing environment variables.
- Invalid JSON payload.
- Missing IAM permissions.

**Solution:**

Check the relevant Lambda function's CloudWatch Logs and verify that the API route is connected to the correct function.

---

## Security Considerations

This project is designed as a learning and portfolio demonstration. Before using a similar architecture in production, consider the following improvements:

- Apply least-privilege IAM policies.
- Protect the approval endpoint with authentication and authorization.
- Never expose callback task tokens in public logs or repositories.
- Validate incoming request data and manager decisions.
- Configure appropriate API throttling and monitoring.
- Enable encryption for sensitive stored information.
- Introduce an audit trail for approval decisions.

---

## Learning Outcomes

Through this project, I gained practical experience with:

- Designing a serverless architecture using AWS services.
- Developing Python-based AWS Lambda functions using Boto3.
- Creating HTTP APIs and integrating them with Lambda.
- Building automated workflows using AWS Step Functions.
- Implementing human-in-the-loop approval using callback task tokens.
- Managing application data using Amazon DynamoDB.
- Configuring IAM roles and permissions for service-to-service communication.
- Monitoring and troubleshooting serverless applications using CloudWatch.

## Documentation

For a more detailed explanation of the application architecture, Lambda functions, callback integration, and Step Functions workflow, refer to:

[EXPLANATION.md](EXPLANATION.md)

---

## Future Enhancements

Potential improvements to extend the project include:

- Integrate Amazon SES to send automated approval notifications.
- Build a web-based certification request submission interface.
- Introduce manager authentication using Amazon Cognito.
- Implement approval deadlines and automated timeout handling.
- Add SNS notifications for completed approval decisions.
- Implement infrastructure deployment using AWS CloudFormation or AWS CDK.

---

## Disclaimer

This project is developed for educational purposes to demonstrate serverless application development and workflow orchestration using AWS services.
