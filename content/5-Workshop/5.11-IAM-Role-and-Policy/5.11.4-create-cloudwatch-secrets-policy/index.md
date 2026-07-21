---
title : "Create CloudWatch and Secrets Manager Policy"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.11.4. </b> "
---

#### Overview

In this step, we will create the **IAM Policy for Amazon CloudWatch and AWS Secrets Manager** for the **DocuMind AI** system.

CloudWatch is utilized to store backend and worker logs, assisting in tracking upload issues, OCR, AI Analysis, SQS processing, and security events. AWS Secrets Manager is used to store sensitive parameters such as `DATABASE_URL`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, or other production configs.

This policy grants the backend/worker permissions to write logs and read secrets in production environments.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an IAM Policy for CloudWatch Logs.
* Authorize the backend/worker to write logs to CloudWatch.
* Authorize the backend/worker to read secrets from AWS Secrets Manager.
* Restrict secret permissions to the project's specific secret ARNs.
* Prepare the policy to be attached to the IAM Role.

---

#### The Role of CloudWatch in DocuMind AI

CloudWatch helps the system:

* Record backend logs.
* Record worker logs.
* Track S3 upload errors.
* Monitor SQS worker errors.
* Monitor Textract OCR exceptions.
* Monitor Gemini/OpenAI API failures.
* Monitor security events and forbidden access attempts.
* Support administrators in operational debugging.

Proposed log group name:

```text id="qga2aq"
/docmind/application
```

---

#### The Role of Secrets Manager

Secrets Manager securely stores parameters such as:

```text id="9bbrzb"
DATABASE_URL
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
AWS_SQS_QUEUE_URL
```

Avoid hardcoding secrets in source code files or pushing active `.env` configurations to GitHub in production.

---

#### CloudWatch and Secrets Manager Flow

```text id="bksf96"
Backend / Worker
  |
  |-- Write logs --> CloudWatch Logs
  |
  |-- Read secrets --> AWS Secrets Manager
  |
  v
Run application securely
```

![cloudwatch-secrets-policy](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.4-create-cloudwatch-secrets-policy/cloudwatch-secrets-policy.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the CloudWatch Logs and Secrets Manager IAM Policy details page, or show the CloudWatch Log Group `/docmind/application` in the AWS Console.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.4-create-cloudwatch-secrets-policy/cloudwatch-secrets-policy.png`

---

#### CloudWatch Logs Permissions Required

Typical logging permissions:

```text id="ovw715"
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

Action descriptions:

| Action                    | Purpose                                         |
| ------------------------- | ----------------------------------------------- |
| `logs:CreateLogGroup`     | Creates the log group if it does not exist      |
| `logs:CreateLogStream`    | Creates log streams for backend/worker services |
| `logs:PutLogEvents`       | Publishes log events to CloudWatch              |
| `logs:DescribeLogStreams` | Inspects existing log stream structures         |

---

#### Secrets Manager Permissions Required

Secret retrieval permission:

```text id="p3grf8"
secretsmanager:GetSecretValue
```

Since the backend only needs to read parameters, do not grant write or delete permissions. Avoid broad wildcard rules like:

```text id="zc1lal"
secretsmanager:*
```

---

#### Step 1: Open the IAM Policies Console

Log in to the AWS Console, search for:

```text id="bzy1nn"
IAM
```

Select:

```text id="l5sl6a"
Policies
```

Then click:

```text id="8qi01z"
Create policy
```

Select the tab:

```text id="y70n3z"
JSON
```

---

#### Step 2: CloudWatch and Secrets Manager Policy JSON Document

Example policy document:

```json id="xtn1n1"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindCloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": [
        "arn:aws:logs:ap-southeast-1:123456789012:log-group:/docmind/application",
        "arn:aws:logs:ap-southeast-1:123456789012:log-group:/docmind/application:*"
      ]
    },
    {
      "Sid": "DocuMindSecretsReadAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:docmind/*"
      ]
    }
  ]
}
```

*Note: Replace `ap-southeast-1`, `123456789012` and secret ARN paths with your actual AWS account details.*

---

#### Step 3: Name the Policy

Specify the policy name:

```text id="6f9byy"
DocuMindCloudWatchSecretsPolicy
```

Proposed description:

```text id="j82slj"
Allows DocuMind AI backend and worker to write CloudWatch logs and read secrets from AWS Secrets Manager.
```

Then click:

```text id="xu2eh1"
Create policy
```

---

#### Step 4: Prepare Secret Key Names

Organize secrets under a distinct prefix namespace:

```text id="l6swpk"
docmind/backend
docmind/production
docmind/ai-keys
```

Example JSON configuration layout:

```json id="sgajkz"
{
  "DATABASE_URL": "postgresql://username:password@host:5432/docmind",
  "OPENAI_API_KEY": "your-openai-api-key",
  "GEMINI_API_KEY": "your-gemini-api-key",
  "JWT_SECRET": "your-jwt-secret"
}
```

---

#### Step 5: Configure Backend to Retrieve Secrets

The backend can toggle between local file and AWS secret retrieval using environment variables:

```env id="jh61ny"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
```

Use local `.env` variables during development, and retrieve secrets from AWS Secrets Manager in production.

---

#### Step 6: Log Filtering Guidelines

The backend/worker should log events:

```text id="bc8doy"
[APP_STARTED]
[DATABASE_CONNECTED]
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[TEXTRACT_OCR_STARTED]
[AI_ANALYSIS_STARTED]
[DOCUMENT_PROCESSING_FAILED]
```

Never print these parameters to the log files:

```text id="1zjxrq"
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
DATABASE_URL full password
AWS credentials
```

---

#### Test CloudWatch Logs

After attaching the policy to the IAM Role and launching the services, check the console:

```text id="o0fprq"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Verify that log streams are created and updating.

---

#### Test Secrets Manager Access

Test secret retrieval using the AWS CLI:

```bash id="jnr6cp"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

If the role has correct permissions, the terminal prints the secret values.

---

#### Troubleshoot Scenarios

| Error                                | Cause                                   | Solution                                      |
| ------------------------------------ | --------------------------------------- | --------------------------------------------- |
| Failed to write CloudWatch logs      | Missing `logs:PutLogEvents` action      | Verify policy configurations                  |
| Failed to create log streams         | Missing `logs:CreateLogStream` action   | Add CreateLogStream action in policy          |
| `AccessDeniedException` on secret read| Missing `secretsmanager:GetSecretValue` | Verify policy configurations                  |
| Secret not found                     | Incorrect secret name                   | Verify the `AWS_SECRET_NAME` configuration    |
| Secrets exposed in logs              | Logging raw config payloads             | Mask secret credentials before logging        |
| Region mismatch                      | Secret or log group is in other region  | Verify the target `AWS_REGION`                |

---

#### Completion Checklist

You have completed this step when:

* The CloudWatch + Secrets Manager IAM Policy is created.
* The policy grants CloudWatch Logs writing rights.
* The policy grants Secrets Manager reading rights.
* Resource ARNs target correct region and account IDs.
* Secret names use the `docmind/` prefix structure.
* The backend/worker does not output secrets in log messages.
* The policy is ready to be attached to the IAM Role.

---

#### Expected Outcome

Following this step, DocuMind AI has a configured policy for CloudWatch Logs and AWS Secrets Manager. The backend/worker can write operational logs and fetch database and AI credentials securely in production.
