---
title : "Create SQS Processing Queue"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

#### Create Amazon SQS Processing Queue

In this section, we will set up **Amazon SQS** to build an asynchronous document processing mechanism for the **DocuMind AI** system.

After a user successfully uploads a document to **Amazon S3**, the backend will not perform OCR and AI processing immediately within the same request. Instead, the backend will create a message containing the document information and send it to the **Amazon SQS Queue**. The worker will retrieve messages from the queue and execute subsequent processing steps, such as calling **Amazon Textract OCR** and sending the extracted content to the **Gemini/OpenAI AI Gateway**.

This design pattern improves system stability, prevents the upload request from waiting too long, and decouples the document upload phase from the background document processing pipeline.

---

#### The Role of Amazon SQS in DocuMind AI

In the **DocuMind AI** project, Amazon SQS is used to:

* Receive document processing jobs after they are uploaded to S3.
* Decouple the communication asynchronously between the Backend API and the Worker.
* Prevent system overload when many users upload documents simultaneously.
* Allow workers to process documents sequentially based on the queue.
* Support retry mechanisms when OCR or AI analysis encounters errors.
* Integrate with a Dead Letter Queue (DLQ) to store messages that fail processing multiple times.
* Increase the overall stability of the document processing pipeline.

Amazon SQS does not directly execute OCR or AI analysis. It only serves as the intermediate message queue to forward messages from the backend to the worker.

---

#### Queue Processing Architecture

The processing flow in this section:

```text id="9rtww8"
User
  |
  v
React Frontend
  |
  v
Backend API Node.js/Express
  |
  v
Amazon S3 stores documents
  |
  v
Amazon SQS receives processing message
  |
  v
Worker polls message from SQS
```

![sqs-processing-queue](/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/sqs-processing-queue.png)

> ⚠️ **Screenshot Suggestion:**
> Draw or capture a diagram illustrating the processing flow consisting of these components: React Frontend, Backend API, Amazon S3, Amazon SQS Queue, and Worker.
> Save the image at the path:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/sqs-processing-queue.png`

---

#### Document Processing Message

After successfully uploading a document to S3, the backend will send a message to SQS containing the necessary details for the worker to process.

Example message:

```json id="f37bll"
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Where:

| Field         | Description                                  |
| ------------- | -------------------------------------------- |
| `documentId`  | The document ID stored in PostgreSQL         |
| `userId`      | The ID of the user who uploaded the document |
| `s3Bucket`    | The name of the S3 bucket storing the file   |
| `s3Key`       | The object key path on S3                    |
| `action`      | The type of task to be processed             |
| `status`      | The current status of the document           |

---

#### Why Async Processing?

If the backend handles the entire workflow in a single request:

```text id="5ff0uc"
Upload file
→ OCR using Textract
→ Call Gemini/OpenAI
→ Save results
→ Return response
```

the user might have to wait for a very long time, especially when the file is large or the AI provider response is slow.

With SQS, the process is split into:

```text id="s7wqsb"
Upload file
→ Save to S3
→ Send job to SQS
→ Return response immediately to the user
→ Worker processes in the background
```

This allows the UI to display statuses such as:

```text id="2j86hv"
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Users can track the processing status on the Dashboard or receive notifications when the document is completed.

---

#### Detailed Content

In section **5.4 - Create SQS Processing Queue**, we will perform the following steps:

* [Create SQS Queue](5.4.1-create-sqs-queue/)
* [Create Dead Letter Queue](5.4.2-create-dead-letter-queue/)
* [Test Sending and Receiving SQS Messages](5.4.3-test-sqs-message/)

---

#### Recommended SQS Configuration

When creating the SQS queue for DocuMind AI, you can use the following configuration:

| Component                 | Recommended Configuration            |
| ------------------------- | ------------------------------------ |
| Queue type                | Standard                             |
| Visibility timeout        | 300 seconds                          |
| Message retention period  | 4 days                               |
| Delivery delay            | 0 seconds                            |
| Receive message wait time | 20 seconds                           |
| Encryption                | SSE-SQS                              |
| Dead Letter Queue         | Enable for better error handling     |

For OCR and AI workflows, a **Standard Queue** is suitable because the system does not require strict message ordering across documents. Each document has its own unique `documentId`, so the worker can process them independently.

---

#### Related Environment Variables

The backend needs environment variables containing the Queue URLs:

```env id="xwf8mo"
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
```

If not using a Dead Letter Queue in the demo, you can temporarily omit the variable:

```env id="nbl0pl"
AWS_SQS_DLQ_URL
```

---

#### Necessary IAM Permissions

The backend needs permissions to send messages to SQS:

```text id="0m3jkq"
sqs:SendMessage
```

The worker needs permissions to receive and delete messages:

```text id="ze71vu"
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

These permissions will be configured in more detail during the **IAM Role and Policy** section of the workshop.

---

#### Expected Outcomes

Upon completion of this section, the system should achieve the following results:

* An SQS Standard Queue has been created for DocuMind AI.
* A Dead Letter Queue has been created if applicable.
* The backend can send messages to SQS after document uploads.
* The worker can receive messages from SQS.
* The message contains the full `documentId`, `userId`, `s3Bucket`, and `s3Key`.
* The system is ready to proceed to document OCR processing using Amazon Textract.

---

#### Connection to the Next Step

After the message is sent to Amazon SQS, the worker will retrieve the message and start document processing. In the next section, we will integrate **Amazon Textract OCR** to extract text from documents stored on S3.

The subsequent processing flow:

```text id="splxg6"
Amazon SQS
  |
  v
Worker
  |
  v
Amazon S3 retrieves document
  |
  v
Amazon Textract OCR
  |
  v
PostgreSQL + Prisma stores OCR Result
```

Thus, section 5.4 acts as the background task orchestration layer for the entire DocuMind AI document processing pipeline.
