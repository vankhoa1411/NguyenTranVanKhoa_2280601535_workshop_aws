---
title : "IAM Role and Policy"
date : 2026-06-10
weight : 11
chapter : false
pre : " <b> 5.11. </b> "
---

#### Configure IAM Role and Policy

In this section, we will configure the **IAM Role** and **IAM Policy** for the **DocuMind AI** system.

IAM stands for **Identity and Access Management**. It is an AWS service used to manage access permissions for AWS resources. In the DocuMind AI project, the backend and worker need to access multiple AWS services including **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Amazon CloudWatch**, and **AWS Secrets Manager**.

Instead of hardcoding `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in source code files or production `.env` files, the system should leverage an **IAM Role**. When the backend/worker runs on EC2 or another AWS environment with a role attached, the AWS SDK automatically retrieves temporary security credentials from the IAM Role, making service calls significantly safer.

---

#### The Role of IAM in DocuMind AI

In the **DocuMind AI** system, IAM Roles and Policies grant the backend/worker permissions to perform the following:

* Upload document payloads to Amazon S3.
* Retrieve documents from Amazon S3 for OCR extraction.
* Post document processing messages to Amazon SQS.
* Retrieve and delete messages from Amazon SQS.
* Invoke Amazon Textract to extract text from documents.
* Log backend/worker outputs directly to CloudWatch.
* Fetch production secrets from AWS Secrets Manager.
* Eliminate the risk of credential leakage in production.

---

#### IAM Role and Policy Architecture

Permission routing in the system:

```text
Backend / Worker / EC2
  |
  v
IAM Role
  |
  |-- S3 Policy
  |-- SQS + Textract Policy
  |-- CloudWatch + Secrets Manager Policy
  |
  v
AWS Services
```

![iam-role-policy-flow](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/iam-role-policy-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the Backend/Worker utilizing an IAM Role to access Amazon S3, Amazon SQS, Amazon Textract, CloudWatch Logs, and AWS Secrets Manager.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/iam-role-policy-flow.png`

---

#### Why Use IAM Roles Instead of Access Keys?

In a local environment, you can use the AWS CLI or AWS credentials files for quick testing. However, when deploying to production, AWS access keys should never be stored in `.env` files.

What you should avoid in production:

```env
AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-key"
```

Reasons:

* High risk of exposure if accidentally committed to GitHub.
* Difficult to manage when multiple environments share the same key.
* Hard to rotate keys periodically.
* Violates the security principle of least privilege.
* If a key is leaked, attackers gain access to your AWS resource pool.

Instead, implement:

```text
IAM Role attached to EC2 / Backend Server
```

In this setup, the backend only needs to configure the region, bucket name, and queue URL. The AWS SDK handles the rest by pulling credentials dynamically from the IAM Role.

---

#### Principle of Least Privilege

When creating IAM Policies, apply the **least privilege** principle, meaning you only grant the minimum permissions required for execution.

Examples:

* For S3 file uploads, grant:
  `s3:PutObject`
* For S3 file reads, grant:
  `s3:GetObject`
* For SQS message retrieval, grant:
  `sqs:ReceiveMessage`
  `sqs:DeleteMessage`

Never define overly broad policies like:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

Or:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

Broad wildcards introduce massive security vulnerabilities if your backend or server is compromised.

---

#### Detailed Content

In section **5.11 - IAM Role and Policy**, we will cover the following steps:

* [Create IAM Role](5.11.1-create-iam-role/)
* [Create S3 Policy](5.11.2-create-s3-policy/)
* [Create SQS and Textract Policy](5.11.3-create-sqs-textract-policy/)
* [Create CloudWatch and Secrets Manager Policy](5.11.4-create-cloudwatch-secrets-policy/)
* [Attach IAM Role to Backend](5.11.5-attach-role-to-backend/)
* [Test IAM Permission](5.11.6-test-iam-permission/)

---

#### Policies to Create

In this workshop, permissions are separated into distinct policies for easier management:

| Policy                            | Purpose                                                      |
| --------------------------------- | ------------------------------------------------------------ |
| `DocuMindS3AccessPolicy`          | Allows uploading and reading documents inside S3             |
| `DocuMindSQSTextractPolicy`       | Allows sending/receiving SQS messages and invoking Textract  |
| `DocuMindCloudWatchSecretsPolicy` | Allows writing logs to CloudWatch and reading Secrets Manager|

These policies will be attached to the target IAM Role:

```text
DocuMindBackendWorkerRole
```

---

#### S3 Permissions Required

Backend and worker S3 actions:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

Details:

| Action            | Purpose                               |
| ----------------- | ------------------------------------- |
| `s3:PutObject`    | Backend uploads documents to S3       |
| `s3:GetObject`    | Worker reads documents for OCR        |
| `s3:DeleteObject` | Deletes objects when document is removed|
| `s3:ListBucket`   | Validates objects inside the bucket   |

Resources must be restricted to the specific project bucket:

```text
arn:aws:s3:::documind-document-storage
arn:aws:s3:::documind-document-storage/*
```

---

#### SQS Permissions Required

Backend queue action:

```text
sqs:SendMessage
```

Worker queue actions:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
sqs:ChangeMessageVisibility
```

If utilizing a Dead Letter Queue (DLQ), you can also grant permissions to check DLQ status so administrators can inspect failures.

---

#### Textract Permissions Required

The worker needs to invoke Amazon Textract for OCR.

Core actions:

```text
textract:DetectDocumentText
textract:AnalyzeDocument
```

If using Textract asynchronous jobs, add:

```text
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

---

#### CloudWatch Permissions Required

Backend and worker require permissions to publish application logs.

Core actions:

```text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

Proposed log group name:

```text
/docmind/application
```

Logged events should include:

```text
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_WORKER_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_PROCESSING_FAILED]
```

Never output API keys, JWT tokens, database passwords, or raw sensitive file contents in logs.

---

#### Secrets Manager Permissions Required

If using AWS Secrets Manager in production, backend needs access:

```text
secretsmanager:GetSecretValue
```

Stored parameters:

```text
DATABASE_URL
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
AWS_SQS_QUEUE_URL
```

Proposed secret key name:

```text
docmind/backend
docmind/production
docmind/ai-keys
```

---

#### Environment Variables After Using IAM Role

In production using IAM Roles, access keys are omitted.

Retained variables:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
```

If Secrets Manager is configured:

```env
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
```

Omit these completely from production configurations:

```env
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

---

#### Test IAM Permission

To verify permissions after creating roles and policies:

```bash
aws sts get-caller-identity
```

Expected result when running on an EC2 instance configured with the IAM Role:

```text
arn:aws:sts::123456789012:assumed-role/DocuMindBackendWorkerRole/i-xxxxxxxx
```

Test individual services:

```bash
aws s3 ls s3://documind-document-storage
```

```bash
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_test","action":"PROCESS_DOCUMENT"}'
```

```bash
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

---

#### Pipeline Integration

IAM Roles are used throughout the pipeline:

```text
Document Upload API
  |
  |-- uses S3 Policy
  v
Amazon S3
 
Backend sends job
  |
  |-- uses SQS Policy
  v
Amazon SQS
 
Worker processing
  |
  |-- uses SQS Policy
  |-- uses S3 Policy
  |-- uses Textract Policy
  v
OCR Result
 
Backend / Worker logs
  |
  |-- uses CloudWatch Policy
  v
CloudWatch Logs
 
Production secrets
  |
  |-- uses Secrets Manager Policy
  v
Secrets Manager
```

---

#### Troubleshoot Scenarios

| Error                            | Cause                                   | Solution                           |
| -------------------------------- | --------------------------------------- | ---------------------------------- |
| `Unable to locate credentials`   | EC2 or backend lacks the IAM Role       | Attach the IAM Role to the instance|
| `AccessDenied` on S3 upload      | Missing `s3:PutObject`                  | Verify S3 Policy configurations    |
| `AccessDenied` on S3 read        | Missing `s3:GetObject`                  | Verify S3 Policy configurations    |
| `QueueDoesNotExist`              | Incorrect SQS URL or Region             | Verify `.env` properties           |
| `AccessDenied` on SQS send       | Missing `sqs:SendMessage`               | Verify SQS Policy configurations   |
| `AccessDeniedException` Textract | Missing Textract permissions            | Verify Textract Policy            |
| CloudWatch logs missing          | Missing logging permissions             | Verify CloudWatch Policy           |
| Secrets Manager fetch fails      | Missing `secretsmanager:GetSecretValue` | Verify Secrets Policy              |
| Region mismatch                  | AWS services reside in different regions| Align the `AWS_REGION` settings    |

---

#### Expected Outcomes

Upon completing this section, the system should achieve:

* IAM Role created for the backend/worker.
* S3 Policy created.
* SQS and Textract Policy created.
* CloudWatch and Secrets Manager Policy created.
* Policies successfully attached to the IAM Role.
* IAM Role attached to the EC2 instance (if deployed on EC2).
* Backend/worker does not use hardcoded AWS Access Keys in production.
* Backend can upload files to S3.
* Worker can fetch messages from SQS.
* Worker can call Amazon Textract.
* Backend/worker can output logs to CloudWatch.
* Backend can retrieve secrets from Secrets Manager if configured.

---

#### Expected Outcome

Following this section, DocuMind AI has a more secure AWS authorization layer. The backend and worker can securely access needed AWS services via IAM Roles and Policies, eliminating credential exposure risks and prepping the platform for production.
