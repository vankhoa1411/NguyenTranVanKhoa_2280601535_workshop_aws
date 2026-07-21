---
title : "Create SQS and Textract Policy"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.11.3. </b> "
---

#### Overview

In this step, we will create the **IAM Policy for Amazon SQS and Amazon Textract** for the **DocuMind AI** system.

The backend requires permissions to post messages to **Amazon SQS** after uploading documents. The worker needs permissions to retrieve messages, delete processed messages from the queue upon success, and invoke **Amazon Textract** to perform OCR on the documents.

This policy enables the asynchronous document processing pipeline to execute smoothly from file upload to OCR processing.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an IAM Policy for Amazon SQS.
* Authorize the backend to send messages to SQS.
* Authorize the worker to retrieve and delete messages from SQS.
* Authorize the worker to invoke Amazon Textract.
* Restrict SQS permissions to the project's specific queue ARNs.
* Prepare the policy to be attached to the IAM Role.

---

#### The Role of the SQS and Textract Policy

In DocuMind AI:

* The backend utilizes SQS to queue document processing jobs.
* The worker retrieves and processes these queued jobs.
* The worker calls Textract to extract text from PDF, PNG, JPG, or JPEG files.
* Upon successful processing, the worker deletes the message from SQS.
* If a failure occurs, the message is retried or routed to the Dead Letter Queue (DLQ).

Permission flow:

```text id="fv6hlj"
IAM Role
  |
  |-- SQS Policy --> Amazon SQS
  |
  |-- Textract Policy --> Amazon Textract
```

![sqs-textract-policy](/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.3-create-sqs-textract-policy/sqs-textract-policy.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the SQS and Textract IAM Policy details or JSON definition page inside the AWS Console.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.3-create-sqs-textract-policy/sqs-textract-policy.png`

---

#### SQS Permissions Required

Backend requires:

```text id="l2sp73"
sqs:SendMessage
```

Worker requires:

```text id="lk222a"
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
sqs:ChangeMessageVisibility
```

If utilizing a Dead Letter Queue (DLQ), you can also grant permissions to check DLQ status so administrators can inspect failures:

```text id="v9qy16"
sqs:GetQueueAttributes
sqs:ReceiveMessage
```

---

#### Textract Permissions Required

If using standard document processing:

```text id="y67vdq"
textract:DetectDocumentText
textract:AnalyzeDocument
```

If using asynchronous Textract jobs:

```text id="g0su31"
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

In this workshop, you can grant both sets of permissions to support flexible OCR workflows.

---

#### Step 1: Open the IAM Policies Console

Log in to the AWS Console, search for:

```text id="42e6u7"
IAM
```

Select:

```text id="4724h2"
Policies
```

Then click:

```text id="lwe5j1"
Create policy
```

Select the tab:

```text id="p2ycpr"
JSON
```

---

#### Step 2: Retrieve the SQS Queue ARN

Navigate to the Amazon SQS Console, open your queue:

```text id="n86z5j"
docmind-document-processing-queue
```

Copy the ARN (e.g., `arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-queue`).

If utilizing a DLQ, copy its ARN too (e.g., `arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-dlq`).

---

#### Step 3: SQS and Textract Policy JSON Document

Replace `123456789012`, region, and queue names with your actual AWS account details.

```json id="d1e0ui"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindSQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": [
        "arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-queue",
        "arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-dlq"
      ]
    },
    {
      "Sid": "DocuMindTextractAccess",
      "Effect": "Allow",
      "Action": [
        "textract:DetectDocumentText",
        "textract:AnalyzeDocument",
        "textract:StartDocumentTextDetection",
        "textract:GetDocumentTextDetection",
        "textract:StartDocumentAnalysis",
        "textract:GetDocumentAnalysis"
      ],
      "Resource": "*"
    }
  ]
}
```

*Note: Textract operations require `Resource: "*"` configurations. SQS resources must be locked to your project's specific queue ARNs.*

---

#### Step 4: Name the Policy

Specify the policy name:

```text id="rknhdg"
DocuMindSQSTextractPolicy
```

Proposed description:

```text id="t32mfr"
Allows DocuMind AI backend and worker to send, receive and delete SQS messages and call Amazon Textract OCR.
```

Then click:

```text id="rc5bfm"
Create policy
```

---

#### Step 5: Verify Environment Variables

Make sure the backend/worker `.env` includes:

```env id="n93qvn"
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

If you are not using a DLQ in your environment, `AWS_SQS_DLQ_URL` can be omitted.

---

#### Quick Test of SQS Permissions

After attaching the policy to the role, test posting a message:

```bash id="82rhzg"
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_test_001","action":"PROCESS_DOCUMENT"}'
```

Test retrieving the message:

```bash id="77h1ye"
aws sqs receive-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

---

#### Quick Test of Textract Permissions

With documents uploaded to S3, the worker can call Textract. If permissions are missing, you will see:

```text id="rnewye"
AccessDeniedException
```

When successful, Textract returns OCR blocks containing parsed words:

```text id="03bc90"
PAGE
LINE
WORD
```

---

#### Troubleshoot Scenarios

| Error                            | Cause                                  | Solution                                      |
| -------------------------------- | -------------------------------------- | --------------------------------------------- |
| `AccessDenied` on SQS send       | Missing `sqs:SendMessage`              | Check SQS Policy configurations               |
| Worker does not receive messages | Missing `sqs:ReceiveMessage`           | Add ReceiveMessage action in policy           |
| Worker processes duplicate tasks | Worker fails to invoke `DeleteMessage` | Verify DeleteMessage action and worker logic  |
| `AccessDeniedException` Textract | Missing Textract permissions           | Verify Textract Policy                        |
| `QueueDoesNotExist`              | Incorrect SQS URL or Region            | Verify `.env` properties                      |
| `InvalidS3ObjectException`       | Textract cannot access S3 object       | Verify S3 policy matches the target s3Key     |

---

#### Completion Checklist

You have completed this step when:

* The SQS IAM Policy is created.
* The policy grants `sqs:SendMessage`.
* The policy grants `sqs:ReceiveMessage`.
* The policy grants `sqs:DeleteMessage`.
* The policy grants `sqs:GetQueueAttributes`.
* The policy grants Amazon Textract execution rights.
* SQS resource limits target the correct queue ARNs.
* The policy is ready to be attached to the IAM Role.

---

#### Expected Outcome

Following this step, the backend can post messages to SQS, and the worker can retrieve those queue items to perform OCR using Amazon Textract. This finishes the asynchronous processing pipeline configuration.
