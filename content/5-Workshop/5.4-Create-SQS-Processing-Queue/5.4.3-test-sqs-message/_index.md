---
title : "Test Sending and Receiving SQS Messages"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.4.3. </b> "
---

#### Overview

After creating the **SQS Processing Queue** and the **Dead Letter Queue**, the next step is to test sending and receiving messages in the **DocuMind AI** system.

In this step, we will verify whether the backend can send messages to SQS after document uploads, and whether the worker can retrieve messages from the queue in preparation for OCR and AI analysis.

Testing SQS ensures that the asynchronous pipeline functions correctly before proceeding to deeper integrations with **Amazon Textract**, the **Gemini API**, and the **OpenAI ChatGPT API**.

---

#### Objectives of This Step

Upon completing this step, you will:

* Send a test message to SQS.
* Verify that the message appears in the queue.
* Retrieve the message using the AWS Console or worker.
* Understand the document processing message structure.
* Troubleshoot IAM permission issues or incorrect Queue URLs.
* Confirm that the system is ready to proceed to the Textract OCR step.

---

#### Message Testing Flow

The message sending and receiving flow in DocuMind AI:

```text
Backend API / AWS Console
  |
  v
Amazon SQS Processing Queue
  |
  v
Worker polls message
  |
  v
Parse documentId + s3Key
  |
  v
Prepare for OCR processing
```

![test-sqs-message](/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.3-test-sqs-message/test-sqs-message.png)

> ⚠️ **Screenshot Suggestion:**
> Capture screenshots of the message being sent to SQS and the worker or AWS Console successfully receiving the message.
> Save the image at the path:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.3-test-sqs-message/test-sqs-message.png`

---

#### Prerequisites Before Testing

Before testing, make sure that:

* The main SQS queue is created.
* You have the Queue URL.
* The DLQ is created (if applicable).
* The Queue URL has been added to `.env`.
* The backend or AWS CLI has `sqs:SendMessage` permissions.
* The worker has `sqs:ReceiveMessage` and `sqs:DeleteMessage` permissions.
* The AWS region in `.env` matches the queue's region.

Example `.env`:

```env
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

---

#### Step 1: Send a Test Message via AWS Console

In the AWS Console:

```text
Amazon SQS
```

Select the queue:

```text
docmind-document-processing-queue
```

Select:

```text
Send and receive messages
```

In the **Message body** section, enter the sample message:

```json
{
  "documentId": "doc_test_001",
  "userId": "user_test_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_test_001/doc_test_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Then click:

```text
Send message
```

If successful, AWS will display a notification indicating that the message was sent to the queue.

---

#### Step 2: Verify the Message in the Queue

After sending the message, return to the queue details page and check:

```text
Available messages
```

If the count has increased, the message is in the queue waiting to be processed by the worker.

---

#### Step 3: Receive the Message via AWS Console

On the page:

```text
Send and receive messages
```

Click:

```text
Poll for messages
```

If successful, you will see the message you just sent.

Verify that the message contains:

* `documentId`
* `userId`
* `s3Bucket`
* `s3Key`
* `action`
* `status`

---

#### Step 4: Send a Message via AWS CLI

You can also send a message using the AWS CLI:

```bash
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_test_001","userId":"user_test_001","s3Bucket":"documind-document-storage","s3Key":"uploads/user_test_001/doc_test_001/invoice-demo.pdf","action":"PROCESS_DOCUMENT","status":"QUEUED"}'
```

If successful, the AWS CLI will return a `MessageId`.

---

#### Step 5: Receive the Message via AWS CLI

Test receiving the message:

```bash
aws sqs receive-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

If a message is retrieved, the CLI will output the message body along with a `ReceiptHandle`.

---

#### Step 6: Delete the Message After Successful Processing

In production, the worker should only delete the message after successful processing.

To test deleting a message via the AWS CLI, use the `ReceiptHandle` from the receive-message step:

```bash
aws sqs delete-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --receipt-handle "your-receipt-handle"
```

If the worker encounters an error, it should not delete the message. When the visibility timeout expires, the message will return to the queue for retry.

---

#### Step 7: Test Backend Message Sending After Uploads

After the backend successfully uploads a document to S3, it should push a message to SQS.

The upload API response should look like:

```json
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED"
  }
}
```

Expected backend logs:

```text
[DOCUMENT_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_STARTED]
[SQS_SEND_MESSAGE_SUCCESS]
[DOCUMENT_STATUS_UPDATED_TO_QUEUED]
```

---

#### Step 8: Test Worker Message Retrieval

When the worker starts, the expected logs are:

```text
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
```

At this stage, the worker only needs to retrieve and parse the message successfully. Calling Textract OCR will be implemented in section 5.5.

The worker must extract the following fields:

```text
documentId
userId
s3Bucket
s3Key
action
```

If the message is missing critical fields, the worker should log an error and skip processing.

---

#### Step 9: Test Error Handling and the DLQ

To test the DLQ, send an invalid message (e.g., missing `s3Key`):

```json
{
  "documentId": "doc_error_001",
  "userId": "user_test_001",
  "action": "PROCESS_DOCUMENT"
}
```

The worker should handle the error and avoid calling `DeleteMessage`. After retries exceed the `Maximum receives` threshold, the message will be forwarded to the DLQ.

Verify the DLQ:

```text
docmind-document-processing-dlq
```

If the message appears in the DLQ, the redrive policy is functioning correctly.

---

#### Common Errors

| Error                    | Cause                                         | Solution                                 |
| ------------------------ | --------------------------------------------- | ---------------------------------------- |
| `AccessDenied`           | Missing SQS permissions                       | Verify the IAM Policy                    |
| `InvalidAddress`         | Incorrect Queue URL                           | Double-check and copy the Queue URL      |
| `QueueDoesNotExist`      | Queue is in a different region or deleted     | Check the AWS region configuration       |
| No messages retrieved    | Messages are currently invisible              | Wait for the visibility timeout to expire |
| Duplicate message retrieval| Message not deleted after successful processing | Call `DeleteMessage` upon success        |
| Message not sent to DLQ  | Redrive policy not configured                 | Verify the DLQ and Maximum receives      |
| JSON parse error         | Message body is not valid JSON                | Verify the message format                |

---

#### Completion Checklist

You have completed this step when:

* A test message is successfully sent to SQS.
* The queue displays the message count.
* The AWS Console or CLI retrieves the message.
* The message contains `documentId`, `userId`, `s3Bucket`, and `s3Key`.
* The backend sends a message after a successful document upload.
* The worker can poll messages from SQS.
* Failed messages are retried or sent to the DLQ.
* The system is ready to proceed to section 5.5 - Integrate Textract OCR.

---

#### Expected Outcome

Following this step, the asynchronous pipeline of DocuMind AI is operational at a basic level. The backend can queue document tasks, the worker can receive messages, and the system is ready to integrate Amazon Textract to perform OCR on documents in the next section.
