---
title : "Attach IAM Role to Backend"
date : 2026-06-10
weight : 5
chapter : false
pre : " <b> 5.11.5. </b> "
---

#### Overview

In this step, we will attach the created IAM Policies to the **IAM Role**, and then attach that IAM Role to the **DocuMind AI** backend/worker server.

If the backend runs on **Amazon EC2**, we will attach the IAM Role to the EC2 instance. Consequently, the AWS SDK in the backend can automatically retrieve temporary credentials from the IAM Role to call AWS services like S3, SQS, Textract, CloudWatch, and Secrets Manager.

This configuration eliminates the need to store AWS Access Keys in the production `.env` file.

---

#### Objectives of This Step

Upon completing this step, you will:

* Attach the S3 Policy to the IAM Role.
* Attach the SQS + Textract Policy to the IAM Role.
* Attach the CloudWatch + Secrets Manager Policy to the IAM Role.
* Attach the IAM Role to the EC2 backend server (if applicable).
* Verify the backend/worker utilizes the IAM Role.
* Omit AWS Access Keys from the production `.env` configuration.

---

#### IAM Role Attachment Architecture

Workflow:

```text id="b8kom4"
EC2 Backend Server
  |
  v
IAM Role: DocuMindBackendWorkerRole
  |
  |-- DocuMindS3AccessPolicy
  |-- DocuMindSQSTextractPolicy
  |-- DocuMindCloudWatchSecretsPolicy
  |
  v
AWS Services
```

![attach-role-backend](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.5-attach-role-to-backend/attach-role-backend.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the IAM Role detailing the three attached policies (S3, SQS/Textract, and CloudWatch/Secrets), or show the EC2 instance configuration displaying the attached IAM Role.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.5-attach-role-to-backend/attach-role-backend.png`

---

#### Step 1: Open the IAM Role

Log in to the AWS Console:

```text id="0niwqt"
IAM
```

Select:

```text id="kvr2oz"
Roles
```

Open the role created earlier:

```text id="2fiiux"
DocuMindBackendWorkerRole
```

---

#### Step 2: Attach the S3 Policy

In the Permissions tab, click:

```text id="2tj42f"
Add permissions
```

Select:

```text id="m4cd1u"
Attach policies
```

Search for the policy:

```text id="82ja7f"
DocuMindS3AccessPolicy
```

Check the policy and click:

```text id="0dl3wz"
Add permissions
```

---

#### Step 3: Attach the SQS + Textract Policy

Repeat the attachment steps for:

```text id="fxsk4r"
DocuMindSQSTextractPolicy
```

This policy authorizes the backend/worker to:

* Post messages to SQS.
* Retrieve messages from SQS.
* Delete messages after processing.
* Invoke Amazon Textract OCR.

---

#### Step 4: Attach the CloudWatch + Secrets Manager Policy

Repeat the attachment steps for:

```text id="rhl6h3"
DocuMindCloudWatchSecretsPolicy
```

This policy authorizes the backend/worker to:

* Write application logs to CloudWatch.
* Fetch secrets from AWS Secrets Manager.

---

#### Step 5: Verify Policies are Attached

After completing the attachments, verify that the IAM Role contains:

```text id="w22gvq"
DocuMindS3AccessPolicy
DocuMindSQSTextractPolicy
DocuMindCloudWatchSecretsPolicy
```

Check the Permissions tab to ensure all three policies are visible.

---

#### Step 6: Attach IAM Role to EC2 Instance

If the backend is deployed on an EC2 instance, navigate to:

```text id="2zzlal"
EC2
```

Select the backend instance.

Then select:

```text id="lke7ve"
Actions
→ Security
→ Modify IAM role
```

Choose the role:

```text id="8pvbk6"
DocuMindBackendWorkerRole
```

Click:

```text id="rbsxqt"
Update IAM role
```

---

#### Step 7: Verify IAM Role on EC2

SSH into the EC2 instance and run:

```bash id="99p9t5"
aws sts get-caller-identity
```

The expected assumed-role response output looks like:

```text id="m1jwmg"
arn:aws:sts::123456789012:assumed-role/DocuMindBackendWorkerRole/i-xxxxxxxxxxxx
```

If the ARN matches the role, the EC2 instance is utilizing the IAM Role.

---

#### Step 8: Update Production `.env`

Remove these parameters from the production `.env` configuration:

```env id="6xy9mq"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Retain only:

```env id="6o7qy4"
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

The AWS SDK will automatically resolve credentials via the instance metadata.

---

#### Step 9: Configure AWS SDK client

In the backend code, instantiate the SDK clients using only the region parameter:

```ts id="a7q7n3"
import { S3Client } from "@aws-sdk/client-s3";
 
export const s3Client = new S3Client({
  region: process.env.AWS_REGION
});
```

Do not pass static credentials in your code:

```ts id="zirrqk"
credentials: {
  accessKeyId: "...",
  secretAccessKey: "..."
}
```

---

#### Step 10: Test AWS Services Access

Test S3:

```bash id="fyz2jn"
aws s3 ls s3://documind-document-storage
```

Test SQS:

```bash id="3dmrct"
aws sqs get-queue-attributes \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --attribute-names All
```

Test Secrets Manager:

```bash id="7ioru4"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

---

#### Troubleshoot Scenarios

| Error                            | Cause                            | Solution                           |
| -------------------------------- | -------------------------------- | ---------------------------------- |
| `Unable to locate credentials`   | EC2 instance lacks the IAM Role  | Modify the instance's IAM Role     |
| `AccessDenied` on S3             | S3 policy is not attached        | Check role permissions             |
| `AccessDenied` on SQS            | SQS policy is not attached       | Verify target queue ARN in policy  |
| `AccessDeniedException` Textract | Missing Textract permissions     | Verify Textract Policy             |
| Failed to retrieve secret        | Secrets Manager policy is missing| Check target secret ARN in policy  |
| `aws sts` lacks assumed-role     | CLI is resolving local profile   | Check the credential lookup order  |

---

#### Completion Checklist

You have completed this step when:

* The IAM Role contains the S3 Policy.
* The IAM Role contains the SQS + Textract Policy.
* The IAM Role contains the CloudWatch + Secrets Manager Policy.
* The EC2 backend instance is configured with the IAM Role.
* Running `aws sts get-caller-identity` returns the assumed role ARN.
* AWS credentials are not hardcoded in the backend.
* The AWS SDK can invoke S3, SQS, Textract, CloudWatch, and Secrets Manager.

---

#### Expected Outcome

Following this step, the DocuMind AI backend and worker can securely access AWS services via the assigned IAM Role rather than static access keys, establishing a production-ready security posture.
