---
title : "Test IAM Permission"
date : 2026-06-10
weight : 6
chapter : false
pre : " <b> 5.11.6. </b> "
---

#### Overview

In this step, we will verify all the IAM permissions configured for the **DocuMind AI** backend and worker.

After creating the IAM Role, defining the policies, and attaching the role to the backend/worker, it is critical to verify that the system can perform all required actions with **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **CloudWatch Logs**, and **AWS Secrets Manager** hay không.

This verification is a crucial gate before running the document processing pipeline end-to-end.

---

#### Objectives of This Step

Upon completing this step, you will:

* Verify that the backend/worker resolves the correct IAM Role.
* Verify Amazon S3 storage read/write permissions.
* Verify SQS message queue send/receive permissions.
* Verify Amazon Textract OCR invocation permissions.
* Verify CloudWatch Logs writing permissions.
* Verify AWS Secrets Manager secret reading permissions.
* Diagnose and resolve `AccessDenied` errors.

---

#### IAM Permission Verification Flow

```text id="qr6w2c"
Backend / Worker
  |
  v
IAM Role
  |
  |-- Test S3
  |-- Test SQS
  |-- Test Textract
  |-- Test CloudWatch
  |-- Test Secrets Manager
  |
  v
Permission Verified
```

![test-iam-permission](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.6-test-iam-permission/test-iam-permission.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the terminal window output for `aws sts get-caller-identity`, S3 file list, SQS messages, or CloudWatch log group screens indicating active role access.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.6-test-iam-permission/test-iam-permission.png`

---

#### Pre-requisites Before Testing

Before running tests, ensure:

* The IAM Role has been created.
* The S3 Policy has been defined.
* The SQS + Textract Policy has been defined.
* The CloudWatch + Secrets Manager Policy has been defined.
* The policies are successfully attached to the IAM Role.
* The IAM Role is attached to the EC2 backend instance (if running on EC2).
* The backend/worker `.env` contains the correct `AWS_REGION`.
* The bucket, queue, and secret resources actually exist on AWS.

---

#### Step 1: Verify Current Identity Credentials

Execute the identity check command:

```bash id="c2dgle"
aws sts get-caller-identity
```

Expected output when running on an EC2 instance with the IAM Role attached:

```json id="1qk9sf"
{
  "UserId": "AROAxxxxxxxx:i-xxxxxxxx",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/DocuMindBackendWorkerRole/i-xxxxxxxx"
}
```

*Note: If the `assumed-role` ARN is missing, modify the EC2 instance settings to attach the role, or check if the AWS CLI is override by local access key settings.*

---

#### Step 2: Test S3 ListBucket Permissions

Run:

```bash id="0pwo2p"
aws s3 ls s3://documind-document-storage
```

If successful, the role possesses `s3:ListBucket` permissions.

If a failure occurs:

```text id="yrw09e"
AccessDenied
```

Check the S3 policy resource blocks to ensure it includes the root bucket ARN:

```text id="g1ylnz"
arn:aws:s3:::documind-document-storage
```

---

#### Step 3: Test S3 PutObject Permissions

Create a temporary test file:

```bash id="lq7u1z"
echo "DocuMind IAM test file" > iam-test.txt
```

Upload it to S3:

```bash id="h5c7qe"
aws s3 cp iam-test.txt s3://documind-document-storage/test/iam-test.txt
```

If successful, the role possesses `s3:PutObject` permissions.

---

#### Step 4: Test S3 GetObject Permissions

Download the uploaded test file:

```bash id="qir9vr"
aws s3 cp s3://documind-document-storage/test/iam-test.txt downloaded-iam-test.txt
```

If successful, the role possesses `s3:GetObject` permissions.

---

#### Step 5: Test SQS SendMessage Permissions

Post a test message to SQS:

```bash id="w4kv2p"
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_iam_test","action":"PROCESS_DOCUMENT","source":"IAM_TEST"}'
```

If successful, the API returns a JSON block containing the `MessageId`.

---

#### Step 6: Test SQS ReceiveMessage Permissions

Retrieve the test message:

```bash id="qpicsz"
aws sqs receive-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

If successful, the client returns the message payload, verifying `sqs:ReceiveMessage` permissions.

---

#### Step 7: Test SQS DeleteMessage Permissions

After retrieving the message, copy the `ReceiptHandle` string and run:

```bash id="8jnvbd"
aws sqs delete-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --receipt-handle "your-receipt-handle"
```

If no errors are thrown, the role possesses `sqs:DeleteMessage` permissions.

---

#### Step 8: Test Amazon Textract Permissions

Use a PDF or image file that is already stored in the S3 bucket.

Example object path:

```text id="jje6ny"
s3://documind-document-storage/test/invoice-demo.pdf
```

The worker or backend script invokes Textract against this object. If using the AWS SDK, run the OCR functionality built in step 5.5.

Expected console logs:

```text id="3pjy9b"
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
```

Potential errors:

* `AccessDeniedException`: The policy lacks required Textract actions.
* `InvalidS3ObjectException`: The S3 bucket name or object key is incorrect.

---

#### Step 9: Test CloudWatch Logs Permissions

Run the backend or worker application and check if logs are published to CloudWatch.

Navigate to the CloudWatch Console:

```text id="jdbtzn"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Expected log events:

```text id="hx1mzb"
[APP_STARTED]
[SQS_WORKER_STARTED]
[DOCUMENT_UPLOAD_STARTED]
```

If the log stream does not appear, check if `logs:CreateLogStream` and `logs:PutLogEvents` are granted.

---

#### Step 10: Test Secrets Manager Permissions

Run:

```bash id="lce44g"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

If successful, the role possesses `secretsmanager:GetSecretValue` permissions.

If a `AccessDeniedException` occurs, verify that the resource ARN in the policy matches the secret.

---

#### Step 11: End-to-End Pipeline Verification

After verifying individual services, trigger a complete processing workflow:

1. Launch the backend application.
2. Launch the worker process.
3. Upload a sample PDF via the frontend UI or Postman.
4. The backend uploads the document to S3.
5. The backend registers the job in SQS.
6. The worker retrieves the message from the queue.
7. The worker invokes Amazon Textract for OCR.
8. The worker calls Gemini/OpenAI for analysis.
9. The worker persists the analysis to the PostgreSQL database.
10. The frontend Dashboard updates the status to `COMPLETED`.

If this flow completes, all IAM permissions are verified.

---

#### Permission Reference Matrix

| Service         | Verification Action           | Expected Outcome        |
| --------------- | ----------------------------- | ----------------------- |
| STS             | `aws sts get-caller-identity` | Returns assumed-role ARN|
| S3              | `aws s3 ls`                   | Lists bucket contents   |
| S3              | `aws s3 cp` upload            | Uploads file payload    |
| S3              | `aws s3 cp` download          | Downloads file payload  |
| SQS             | `send-message`                | Returns MessageId       |
| SQS             | `receive-message`             | Retrieves message       |
| SQS             | `delete-message`              | Deletes message         |
| Textract        | OCR test file                 | Finishes OCR extraction |
| CloudWatch      | Check Log Group               | Displays app logs       |
| Secrets Manager | `get-secret-value`            | Returns secret value    |

---

#### Troubleshoot Scenarios

| Error                            | Cause                                  | Solution                                      |
| -------------------------------- | -------------------------------------- | --------------------------------------------- |
| `Unable to locate credentials`   | IAM Role or access keys missing        | Attach the IAM Role to the EC2 instance       |
| `AccessDenied` on S3             | Missing S3 policy actions              | Check bucket and object ARNs in S3 policy     |
| `QueueDoesNotExist`              | Incorrect queue URL or region          | Verify settings in `.env`                     |
| `AccessDenied` on SQS            | SQS policy actions missing             | Verify queue ARNs in SQS policy               |
| `AccessDeniedException` Textract | Missing Textract permissions           | Add Textract actions in policy                |
| `InvalidS3ObjectException`       | Missing S3 permissions or wrong key    | Check object path and S3 GetObject permissions|
| CloudWatch log stream missing    | Missing logs policy or logger config   | Add log group/stream actions in policy        |
| Secret not found                 | Incorrect secret name identifier       | Verify `AWS_SECRET_NAME` value                |
| Region mismatch                  | AWS services in different regions      | Align region settings in `.env`               |

---

#### Completion Checklist

You have completed this step when:

* `aws sts get-caller-identity` confirms the assumed role is active.
* S3 List, Put, and Get commands execute without errors.
* SQS Send, Receive, and Delete operations succeed.
* The worker invokes Amazon Textract OCR successfully.
* Backend/worker logs are visible in CloudWatch logs.
* Backend can retrieve keys from Secrets Manager (if enabled).
* No `AccessDenied` errors are present in the app.
* The upload → S3 → SQS → Textract → AI Analysis pipeline functions.

---

#### Expected Outcome

Following this step, the DocuMind AI system has verified IAM Role configurations. The backend and worker have all required privileges to run the AWS document processing pipeline without hardcoding AWS Access Keys in production.
