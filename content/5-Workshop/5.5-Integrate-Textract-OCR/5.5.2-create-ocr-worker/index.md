---
title : "Create OCR Worker"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.5.2. </b> "
---

#### Overview

In this step, we will build the **OCR Worker** for the **DocuMind AI** system.

The OCR Worker is a background process whose task is to retrieve messages from **Amazon SQS**, fetch documents from **Amazon S3**, invoke **Amazon Textract** to perform OCR, and then store the OCR results in **PostgreSQL via Prisma**.

The worker helps offload heavy document processing tasks from the Backend API. Consequently, when a user uploads a document, the backend can respond quickly while the OCR and AI analysis are processed asynchronously in the background.

---

#### Objectives of This Step

Upon completing this step, you will:

* Understand the role of the OCR Worker in the DocuMind AI pipeline.
* Create a worker to retrieve messages from Amazon SQS.
* Parse messages to extract `documentId`, `userId`, `s3Bucket`, and `s3Key`.
* Invoke Amazon Textract to perform OCR on documents.
* Store OCR results in PostgreSQL.
* Update document statuses in the database.
* Prepare the OCR data to be sent to the Gemini/OpenAI AI Gateway in the next step.

---

#### The Role of the OCR Worker

In the DocuMind AI system, the OCR Worker performs the following tasks:

* Polls messages from Amazon SQS.
* Validates whether the message format is correct.
* Updates the document status to `PROCESSING`.
* Fetches the document from Amazon S3 using `s3Bucket` and `s3Key`.
* Invokes Amazon Textract to perform OCR.
* Stores the OCR text in the `OCRResult` table.
* Updates the document status to `OCR_COMPLETED`.
* Forwards the OCR text to the AI Analysis step if the system has integrated it.
* Logs processing details to CloudWatch or the console.

Worker flow:

```text
Amazon SQS
  |
  v
OCR Worker
  |
  v
Amazon S3
  |
  v
Amazon Textract
  |
  v
PostgreSQL + Prisma
```

![ocr-worker](/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.2-create-ocr-worker/ocr-worker.png)

> ⚠️ **Screenshot Suggestion:**
> Capture a screenshot of the worker logs when retrieving SQS messages and invoking Textract successfully, or draw a diagram representing SQS → Worker → S3 → Textract → PostgreSQL.
> Save the image at the path:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.2-create-ocr-worker/ocr-worker.png`

---

#### SQS Message Input

The worker retrieves messages from SQS in the following format:

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

Key fields:

| Field         | Description                                  |
| ------------- | -------------------------------------------- |
| `documentId`  | Identifies the document in the database      |
| `userId`      | Identifies the owner of the document         |
| `s3Bucket`    | The S3 bucket containing the original file   |
| `s3Key`       | The object key path on S3                    |
| `action`      | The type of task to be processed             |

If the message is missing `documentId` or `s3Key`, the worker should log an error and skip processing.

---

#### Install Necessary AWS SDK Packages

If the backend/worker uses Node.js, install the required packages:

```bash
npm install @aws-sdk/client-sqs @aws-sdk/client-textract @aws-sdk/client-s3
```

If using Prisma:

```bash
npm install @prisma/client
```

---

#### Recommended Directory Structure

You can organize the worker files as follows:

```text
src/
├── workers/
│   └── document.worker.ts
├── services/
│   ├── textract.service.ts
│   ├── sqs.service.ts
│   └── document.service.ts
├── prisma/
│   └── client.ts
└── config/
    └── aws.config.ts
```

File descriptions:

| File                  | Role                                         |
| --------------------- | -------------------------------------------- |
| `document.worker.ts`  | Main worker file that retrieves and coordinates tasks |
| `textract.service.ts` | Handles calls to Amazon Textract OCR         |
| `sqs.service.ts`      | Manages receiving and deleting SQS messages  |
| `document.service.ts` | Updates document statuses and saves OCR text |
| `aws.config.ts`       | Configures AWS client instances              |

---

#### OCR Worker Logic

Pseudo flow:

```text
Start worker
  |
  v
Poll message from SQS
  |
  v
Parse message JSON
  |
  v
Validate documentId + s3Key
  |
  v
Update document status = PROCESSING
  |
  v
Call Textract OCR
  |
  v
Save OCRResult to PostgreSQL
  |
  v
Update document status = OCR_COMPLETED
  |
  v
Delete message from SQS
```

In case of errors:

```text
Error
  |
  v
Log error
  |
  v
Update document status = FAILED
  |
  v
Do not delete message if SQS retry is desired
```

In the demo, if you want to avoid continuous retries, you can delete the message after updating the status to `FAILED`. However, in production, using retries and DLQs is recommended.

---

#### Example Function to Call Textract

Example of OCR logic using Textract:

```ts
import { TextractClient, DetectDocumentTextCommand } from "@aws-sdk/client-textract";
 
const textractClient = new TextractClient({
  region: process.env.AWS_REGION
});
 
export async function extractTextFromS3(s3Bucket: string, s3Key: string) {
  const command = new DetectDocumentTextCommand({
    Document: {
      S3Object: {
        Bucket: s3Bucket,
        Name: s3Key
      }
    }
  });
 
  const response = await textractClient.send(command);
 
  const lines = response.Blocks
    ?.filter((block) => block.BlockType === "LINE")
    .map((block) => block.Text)
    .filter(Boolean) || [];
 
  return lines.join("\n");
}
```

---

#### Example Worker Code for Processing Messages

Shortened example:

```ts
async function processDocumentMessage(messageBody: string) {
  const payload = JSON.parse(messageBody);
 
  const { documentId, userId, s3Bucket, s3Key } = payload;
 
  if (!documentId || !s3Bucket || !s3Key) {
    throw new Error("Invalid SQS message payload");
  }
 
  await prisma.document.update({
    where: { id: documentId },
    data: { status: "PROCESSING" }
  });
 
  const extractedText = await extractTextFromS3(s3Bucket, s3Key);
 
  await prisma.oCRResult.create({
    data: {
      documentId,
      rawText: extractedText,
      provider: "AWS_TEXTRACT"
    }
  });
 
  await prisma.document.update({
    where: { id: documentId },
    data: { status: "OCR_COMPLETED" }
  });
 
  return extractedText;
}
```

Note: The Prisma model name might vary depending on your schema definition (e.g., `OCRResult`, `OcrResult`, or `oCRResult`). Adjust the naming to match your actual schema.

---

#### Document Statuses

During the OCR pipeline, the following statuses can be used:

```text
QUEUED
PROCESSING
OCR_COMPLETED
FAILED
```

After integrating AI Analysis, you can expand these to include:

```text
AI_ANALYZING
COMPLETED
```

---

#### Storing OCR Results in the Database

OCR results should be saved in a dedicated table, for example, `OCRResult`:

```json
{
  "id": "ocr_001",
  "documentId": "doc_001",
  "rawText": "Extracted text from invoice...",
  "provider": "AWS_TEXTRACT",
  "createdAt": "2026-06-10T10:00:00.000Z"
}
```

If the document does not contain readable text, the worker should log an error or warning to let the user know the document could not be OCR-ed.

---

#### Logging in the Worker

The worker should output logs at key stages:

```text
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[DOCUMENT_STATUS_UPDATED]
[SQS_MESSAGE_DELETED]
```

In case of errors:

```text
[TEXTRACT_OCR_FAILED]
[DOCUMENT_PROCESSING_FAILED]
```

If CloudWatch is integrated, these logs should be sent to the log group:

```text
/docmind/application
```

---

#### Completion Checklist

You have completed this step when:

* The worker successfully polls messages from SQS.
* The worker parses `documentId`, `s3Bucket`, and `s3Key` correctly.
* The worker successfully invokes Amazon Textract.
* OCR text is successfully extracted.
* OCR results are saved in PostgreSQL.
* The document status is updated to `OCR_COMPLETED`.
* SQS messages are deleted upon successful processing.
* The worker writes clear log statements for each step.

---

#### Expected Outcome

Following this step, the DocuMind AI system has an OCR Worker operational for background document processing. Documents uploaded to S3 can be retrieved, processed via Amazon Textract, and saved to PostgreSQL, preparing the data for subsequent AI analysis using Gemini/OpenAI.
