---
title : "Create SQS Queue"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.4.1. </b> "
---

#### Overview

In this step, we will create an **Amazon SQS Queue** to serve as the document processing queue for the **DocuMind AI** system.

After a user successfully uploads a document to **Amazon S3**, the backend will send a message to SQS. This message contains information such as `documentId`, `userId`, `s3Bucket`, `s3Key`, and the type of task to be processed. The worker will retrieve messages from the queue to proceed with OCR processing using **Amazon Textract** and document analysis using **Gemini/OpenAI** in later steps.

Using SQS decouples the document upload phase from the background document processing pipeline, resulting in faster backend response times and increased system stability when many documents are uploaded concurrently.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an SQS Standard Queue for DocuMind AI.
* Understand the role of SQS in the document processing pipeline.
* Configure the queue appropriately for OCR/AI processing workers.
* Obtain the Queue URL to configure in your backend.
* Prepare for creating the Dead Letter Queue in the next section.

---

#### The Role of SQS Queue in DocuMind AI

In the DocuMind AI system, the SQS Queue is used to:

* Receive document processing jobs after S3 upload.
* Pass messages from the Backend API to the Worker.
* Enable asynchronous OCR and AI processing.
* Reduce user wait times during document uploads.
* Allow workers to process documents sequentially based on the queue.
* Support retry mechanisms when temporary processing errors occur.

The processing flow in this step:

```text
Backend API
  |
  v
Amazon SQS Queue
  |
  v
Worker background processing
```

![create-sqs-queue](/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.1-create-sqs-queue/create-sqs-queue.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the SQS Queue creation page in the AWS Console, or draw a diagram illustrating the Backend API sending a message to SQS and the Worker receiving messages from SQS.
> Save the image at the path:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.1-create-sqs-queue/create-sqs-queue.png`

---

#### Step 1: Access Amazon SQS

Log in to the AWS Console, then search for the service:

```text
SQS
```

Select:

```text
Amazon Simple Queue Service
```

Then click:

```text
Create queue
```

---

#### Step 2: Select Queue Type

In the **Type** section, select:

```text
Standard
```

For DocuMind AI, a **Standard Queue** is suitable because each document has its own unique `documentId` and the system does not require strict first-in-first-out (FIFO) execution.

You do not need a FIFO Queue for this workshop, unless the system requires strict sequential processing for every document or user.

---

#### Step 3: Name the Queue

Enter a name for the queue based on the project:

```text
docmind-document-processing-queue
```

Alternatively, you can use other names:

```text
documind-document-processing-queue
docmind-ai-processing-queue
docmind-ocr-analysis-queue
```

The queue name should clearly indicate its purpose: document processing, OCR, and AI analysis.

---

#### Step 4: Configure the Queue

Recommended configuration:

| Component                 | Recommended Configuration |
| ------------------------- | ------------------------- |
| Visibility timeout        | 300 seconds               |
| Message retention period  | 4 days                    |
| Delivery delay            | 0 seconds                 |
| Receive message wait time | 20 seconds                |
| Maximum message size      | 256 KB                    |
| Encryption                | SSE-SQS                   |

Quick explanations:

* **Visibility timeout**: The period during which a message is hidden after a worker retrieves it. If the worker takes time to process, set this long enough to prevent duplicate processing.
* **Message retention period**: The duration a message remains in the queue if it is not processed.
* **Receive message wait time**: Set this to 20 seconds to enable long polling, which minimizes empty responses.
* **Encryption**: Enable SSE-SQS to protect messages in the queue.

---

#### Step 5: Configure Access Policy

For the demo environment, you can keep the default:

```text
Only the queue owner
```

This means only the AWS account that owns the queue has permissions to perform operations on it.

In production, permissions to send and receive messages should be managed via **IAM Roles and Policies**. The backend requires `sqs:SendMessage` permissions, while the worker requires `sqs:ReceiveMessage`, `sqs:DeleteMessage`, and `sqs:GetQueueAttributes` permissions.

---

#### Step 6: Create the Queue

After verifying the configurations, click:

```text
Create queue
```

Once successfully created, AWS will redirect you to the queue details page.

---

#### Step 7: Retrieve the Queue URL

On the queue details page, locate:

```text
URL
```

Copy the Queue URL, for example:

```text
https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue
```

Add this URL to the backend `.env` file:

```env
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
```

---

#### Step 8: Define the Message Format

Messages sent to the queue should have a clear structure:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

The worker will use the `documentId` and `s3Key` to retrieve the document from S3, perform OCR, and update the results in PostgreSQL.

---

#### Step 9: Verify the Queue Configuration

On the queue details page, check the following details:

| Component                 | Expected State             |
| ------------------------- | -------------------------- |
| Queue type                | Standard                   |
| Visibility timeout        | 300 seconds                |
| Encryption                | Enabled                    |
| Receive message wait time | 20 seconds                 |
| Queue URL                 | Copied to backend `.env`   |

---

#### Completion Checklist

You have completed this step when:

* The SQS Standard Queue is successfully created.
* The queue name matches the project naming convention.
* The Queue URL is copied to the backend `.env` file.
* Encryption is enabled on the queue.
* Long polling is enabled on the queue.
* The backend is ready to send messages to the queue.

---

#### Expected Outcome

Following this step, the DocuMind AI system has an intermediate queue to accept document processing jobs. This sets the foundation for the worker to receive messages, download files from S3, and execute OCR processing using Amazon Textract in subsequent steps.
