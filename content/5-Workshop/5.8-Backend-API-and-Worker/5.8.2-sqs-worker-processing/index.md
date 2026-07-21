---
title : "Process SQS Worker"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.8.2. </b> "
---

#### Overview

In this step, we will build and verify the **SQS Worker Processing** component for the **DocuMind AI** system.

The SQS Worker is a background process responsible for polling messages from **Amazon SQS**, reading document information, invoking **Amazon Textract** to perform OCR, and sending the OCR results to the **AI Gateway** to analyze with Gemini or OpenAI. The final results will be stored in **PostgreSQL via Prisma**.

Using a background worker prevents the backend API from executing slow OCR and AI operations directly inside upload request cycles, thereby optimizing performance and user experience.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create a worker process to poll messages from Amazon SQS.
* Parse messages containing `documentId`, `userId`, `s3Bucket`, and `s3Key`.
* Update document status at each stage of the processing pipeline.
* Invoke Amazon Textract to perform OCR.
* Forward extracted OCR text to the AI Gateway.
* Save `OCRResult` and `AIAnalysis` records in PostgreSQL.
* Delete processed messages from SQS upon successful execution.
* Handle execution failures and manage retries or Dead Letter Queues (DLQ).

---

#### SQS Worker Processing Flow

The worker processing flow in DocuMind AI:

```text id="t52jxz"
Amazon SQS
  |
  v
SQS Worker
  |
  v
Update Document status = PROCESSING
  |
  v
Amazon Textract OCR
  |
  v
Save OCRResult
  |
  v
AI Gateway
  |
  v
Gemini / OpenAI
  |
  v
Save AIAnalysis
  |
  v
Update Document status = COMPLETED
  |
  v
Delete SQS message
```

![sqs-worker-processing](/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.2-sqs-worker-processing/sqs-worker-processing.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the worker logs when it polls a message from SQS, executes OCR, invokes the AI Gateway, and updates the document status to `COMPLETED`.
> Save the image at the path:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.2-sqs-worker-processing/sqs-worker-processing.png`

---

#### Recommended Directory Structure

You can organize the source code as follows:

```text id="o281pn"
src/
├── workers/
│   └── document.worker.ts
├── services/
│   ├── sqs.service.ts
│   ├── textract.service.ts
│   ├── document.service.ts
│   └── ai/
│       └── ai.gateway.ts
└── prisma/
    └── client.ts
```

---

#### Input Message Format

The worker consumes messages matching this format:

```json id="krmh93"
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

The worker should validate the presence of these required fields:

```text id="20zhhb"
documentId
userId
s3Bucket
s3Key
action
```

If essential fields are missing, the worker should log an error and update the document status to `FAILED`.

---

#### Worker Processing Logic

Workflow:

```text id="3e0cpi"
Start worker
  |
  v
Poll message from SQS
  |
  v
Parse message JSON
  |
  v
Validate payload
  |
  v
Update document status = PROCESSING
  |
  v
Call Textract OCR
  |
  v
Save OCRResult
  |
  v
Update document status = OCR_COMPLETED
  |
  v
Update document status = AI_ANALYZING
  |
  v
Call AI Gateway
  |
  v
Save AIAnalysis
  |
  v
Update document status = COMPLETED
  |
  v
Delete message from SQS
```

On error:

```text id="6h4nw7"
Catch error
  |
  v
Log error
  |
  v
Update document status = FAILED
  |
  v
Do not delete message if retry is needed
```

---

#### Example Worker Implementation

```ts id="1a1txt"
async function processMessage(message: any) {
  const payload = JSON.parse(message.Body);
 
  const { documentId, userId, s3Bucket, s3Key } = payload;
 
  if (!documentId || !userId || !s3Bucket || !s3Key) {
    throw new Error("Invalid SQS message payload");
  }
 
  await prisma.document.update({
    where: { id: documentId },
    data: { status: "PROCESSING" }
  });
 
  const ocrText = await textractService.extractTextFromS3(s3Bucket, s3Key);
 
  await prisma.oCRResult.create({
    data: {
      documentId,
      provider: "AWS_TEXTRACT",
      rawText: ocrText
    }
  });
 
  await prisma.document.update({
    where: { id: documentId },
    data: { status: "AI_ANALYZING" }
  });
 
  const aiResult = await analyzeDocumentWithGateway(ocrText);
 
  await prisma.aIAnalysis.create({
    data: {
      documentId,
      provider: aiResult.provider,
      model: aiResult.model,
      documentType: aiResult.documentType,
      summary: aiResult.summary,
      entities: aiResult.entities,
      keywords: aiResult.keywords,
      importantInformation: aiResult.importantInformation,
      confidence: aiResult.confidence
    }
  });
 
  await prisma.document.update({
    where: { id: documentId },
    data: { status: "COMPLETED" }
  });
 
  await sqsService.deleteMessage(message.ReceiptHandle);
}
```

Make sure the Prisma model names (such as `ocrResult` or `aiAnalysis`) match the exact case of your generated Prisma Client.

---

#### Retries and Dead Letter Queue

When the worker encounters transient errors, such as an AI provider timeout or Textract service issue, you can let SQS retry the processing by not deleting the message from the queue.

If the message fails after exceeding the maximum receive count, SQS automatically routes it to the Dead Letter Queue (DLQ).

Retry flow:

```text id="zdh503"
Worker failed
  |
  v
Do not DeleteMessage
  |
  v
SQS retry
  |
  v
Exceed maximum receives
  |
  v
Move to DLQ
```

For demo environments, if you wish to avoid repeated retries, you can immediately update the document status to `FAILED` and delete the SQS message. However, using a DLQ is recommended in production environments for debugging.

---

#### Document Processing Statuses

The worker updates statuses sequentially:

```text id="eyyzzv"
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

The Frontend Dashboard consumes these statuses to display progress updates to users.

---

#### Required Logging

 the worker should log key events:

```text id="71tm9q"
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[DOCUMENT_STATUS_COMPLETED]
[SQS_MESSAGE_DELETED]
```

On execution errors:

```text id="h5b3gq"
[DOCUMENT_PROCESSING_FAILED]
[SQS_MESSAGE_RETRY_PENDING]
[SQS_MESSAGE_MOVED_TO_DLQ]
```

---

#### Related Environment Variables

```env id="i3rjmx"
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_S3_BUCKET_NAME="documind-document-storage"
 
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
 
AI_PROVIDER="openai"
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
 
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

---

#### How to Run the Worker

Add these scripts to `package.json`:

```json id="3ri26b"
{
  "scripts": {
    "worker": "ts-node src/workers/document.worker.ts",
    "worker:dev": "tsx watch src/workers/document.worker.ts"
  }
}
```

Start the worker:

```bash id="ljov8s"
npm run worker
```

Or for development mode:

```bash id="qx5vih"
npm run worker:dev
```

---

#### Completion Checklist

You have completed this step when:

* The worker successfully polls messages from SQS.
* The worker parses `documentId`, `s3Bucket`, and `s3Key`.
* The worker updates the status to `PROCESSING`.
* The worker invokes Textract OCR.
* The worker stores the `OCRResult` in the database.
* The worker calls the AI Gateway.
* The worker stores the `AIAnalysis` in the database.
* The worker updates the status to `COMPLETED`.
* The worker deletes messages from SQS upon successful execution.
* Processing errors are handled, and the status updated to `FAILED` or the message moved to the DLQ.

---

#### Expected Outcome

Following this step, DocuMind AI has a complete document processing worker. Uploaded documents are successfully queued, and the background worker executes OCR extraction and AI Analysis, storing the structured results in PostgreSQL for display on the Dashboard.
