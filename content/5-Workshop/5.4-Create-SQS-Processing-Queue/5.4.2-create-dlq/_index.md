---
title : "Create Dead Letter Queue"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.4.2. </b> "
---

#### Overview

In this step, we will create a **Dead Letter Queue (DLQ)** for the document processing queue of **DocuMind AI**.

A Dead Letter Queue is used to store messages that fail processing multiple times. For example, if a worker receives a message from SQS but continuously encounters errors due to a non-existent file on S3, missing Textract permissions, or invalid message data, that message will be moved to the DLQ after a specific number of retries.

Implementing a DLQ makes it easier to monitor errors, prevents faulty messages from being retried indefinitely, and protects the document processing pipeline.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create a Dead Letter Queue for DocuMind AI.
* Link the DLQ to the main processing queue.
* Configure the maximum number of retries before a message is moved to the DLQ.
* Understand how to inspect failed messages in the DLQ.
* Prepare error-handling mechanisms for the OCR/AI worker.

---

#### The Role of Dead Letter Queue in DocuMind AI

In the DocuMind AI system, the DLQ is used to:

* Store messages that fail processing multiple times.
* Assist in debugging errors related to OCR, S3, Textract, or AI providers.
* Avoid infinite retries of faulty messages.
* Allow Admins or Developers to review failed documents.
* Increase the stability of the asynchronous processing pipeline.

Example errors that might move a message to the DLQ:

* The file does not exist in S3.
* The worker lacks `s3:GetObject` permissions.
* The worker lacks permissions to call Amazon Textract.
* The message is missing the `documentId` or `s3Key`.
* The file format is not supported for OCR.
* The AI provider is down or continuously failing.
* The worker crashes before finishing execution.

---

#### DLQ Architecture

The processing flow with a Dead Letter Queue:

```text
Main SQS Queue
  |
  v
Worker processes message
  |
  |-- Success --> Delete message from queue
  |
  |-- Failed multiple times --> Dead Letter Queue
```

![dead-letter-queue](/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.2-create-dead-letter-queue/dead-letter-queue.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the Redrive policy configuration of SQS in the AWS Console, or draw a diagram illustrating Main Queue → Worker → Dead Letter Queue when message processing fails.
> Save the image at the path:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.2-create-dead-letter-queue/dead-letter-queue.png`

---

#### Step 1: Create the Dead Letter Queue

Log in to the AWS Console:

```text
Amazon SQS
```

Select:

```text
Create queue
```

Select the queue type:

```text
Standard
```

Enter the DLQ name:

```text
docmind-document-processing-dlq
```

Recommended configuration:

| Component                 | Recommended Configuration   |
| ------------------------- | --------------------------- |
| Queue type                | Standard                    |
| Message retention period  | 4 days or 14 days           |
| Receive message wait time | 20 seconds                  |
| Encryption                | SSE-SQS                     |

Since the DLQ is used to hold failed messages for debugging purposes, you can configure a longer retention period than the main queue if desired.

---

#### Step 2: Copy the DLQ ARN

After successfully creating the DLQ, open the queue details page and copy:

```text
ARN
```

For example:

```text
arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-dlq
```

This ARN will be used to configure the Redrive policy on the main queue.

---

#### Step 3: Link DLQ to the Main Queue

Open the main queue:

```text
docmind-document-processing-queue
```

Select the tab or section:

```text
Dead-letter queue
```

Click:

```text
Edit
```

Toggle:

```text
Enable
```

Choose the DLQ you just created:

```text
docmind-document-processing-dlq
```

---

#### Step 4: Configure Maximum Receives

In the **Maximum receives** section, set the number of retries before a message is moved to the DLQ.

Recommended setting:

```text
3
```

This means that if a worker retrieves and fails to process a message 3 times, SQS will automatically move the message to the Dead Letter Queue.

Configuration:

```text
Maximum receives: 3
```

---

#### Step 5: Update Environment Variables (Optional)

If your backend or admin dashboard needs to display the DLQ status, add the DLQ URL to `.env`:

```env
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

If the worker only needs to process the main queue, this environment variable is not immediately required for the demo.

---

#### Step 6: Configure Error Handling in the Worker

When the worker fails to process a message, it should not delete the message from the queue immediately. If the worker does not call `DeleteMessage`, the message will return to the main queue after the visibility timeout expires.

Pseudo flow:

```text
Receive message
  |
  v
Process OCR/AI
  |
  |-- Success: DeleteMessage
  |
  |-- Failed: Do not delete message
  |
  v
SQS retry
```

After the retries exceed the configured `Maximum receives`, SQS will automatically forward the message to the DLQ.

---

#### Step 7: Inspect DLQ in the AWS Console

In the SQS Console, select the DLQ:

```text
docmind-document-processing-dlq
```

Verify the following details:

* Available messages
* Messages in flight
* Oldest message age
* Queue URL
* ARN

If there are messages in the DLQ, it indicates that jobs have repeatedly failed processing and need to be investigated.

---

#### Example Failed Message

A failed message inside the DLQ may look like this:

```json
{
  "documentId": "doc_error_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_error_001/missing-file.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

The underlying cause might be that the `s3Key` does not exist or the worker lacks permissions to download the file from S3.

---

#### Related IAM Permissions

Typically, the backend/worker needs permissions for the main queue:

```text
sqs:SendMessage
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

If the admin dashboard needs to read DLQ messages, you may need to add:

```text
sqs:GetQueueAttributes
sqs:ReceiveMessage
```

Do not permit deleting messages from the DLQ arbitrarily without an inspection mechanism or audit logging.

---

#### Completion Checklist

You have completed this step when:

* The Dead Letter Queue is created.
* The DLQ type matches the main queue type (Standard).
* Encryption is enabled on the DLQ.
* The Redrive policy is linked to the main queue.
* The Maximum receives value is configured.
* You know how to inspect failed messages in the DLQ.
* The worker is designed to keep messages in SQS upon processing failure.

---

#### Expected Outcome

Following this step, the DocuMind AI system has a mechanism to capture and store jobs that repeatedly fail document processing. This simplifies debugging, prevents infinite processing loops, and enhances the reliability of the OCR/AI pipeline.
