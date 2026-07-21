---
title : "Integrate Textract OCR"
date : 2026-06-10
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

#### Integrate Amazon Textract OCR

In this section, we will integrate **Amazon Textract** into the **DocuMind AI** system to extract text from documents uploaded to **Amazon S3**.

After a document is stored in S3 and its processing message is sent to **Amazon SQS**, the worker will retrieve the message from the queue, extract the `s3Bucket` and `s3Key` information, and then call **Amazon Textract** to perform OCR. The OCR results will be saved to **PostgreSQL via Prisma** and will serve as input data for subsequent AI analysis using the **Gemini API** or **OpenAI ChatGPT API** in the next section.

Amazon Textract plays a critical role in the pipeline because the system must convert PDF documents or images into text before the AI can summarize, classify, extract information, or support document Q&A.

---

#### The Role of Amazon Textract in DocuMind AI

In the **DocuMind AI** project, Amazon Textract is used to:

* Extract text from PDF, PNG, JPG, or JPEG documents.
* Convert unstructured documents into textual data.
* Provide the input content for the Gemini/OpenAI AI Gateway.
* Support AI Analysis, AI Chat/RAG, Semantic Search, and Enterprise Sandbox features.
* Reduce manual reading and data entry operations.
* Automate the OCR step within the document processing workflow.

Textract does not replace the AI provider. Textract only performs OCR, while document understanding, summarization, classification, and Q&A are handled by Gemini/OpenAI in later sections.

---

#### OCR Processing Architecture

The OCR processing flow in this section:

```text
Amazon SQS
  |
  v
OCR Worker
  |
  v
Amazon S3 retrieves document
  |
  v
Amazon Textract OCR
  |
  v
PostgreSQL + Prisma stores OCR Result
  |
  v
Prepare data for Gemini/OpenAI
```

![textract-ocr-flow](/static/images/5-Workshop/5.5-Integrate-Textract-OCR/textract-ocr-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw or capture a diagram illustrating the OCR processing flow consisting of these components: Amazon SQS, OCR Worker, Amazon S3, Amazon Textract, and PostgreSQL + Prisma.
> Save the image at the path:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/textract-ocr-flow.png`

---

#### OCR Processing Workflow

The OCR workflow in DocuMind AI operates as follows:

1. The user uploads a document to the system.
2. The backend stores the document in Amazon S3.
3. The backend sends a processing message to Amazon SQS.
4. The OCR Worker retrieves the message from SQS.
5. The worker extracts the `documentId`, `userId`, `s3Bucket`, and `s3Key`.
6. The worker invokes Amazon Textract to perform OCR on the document.
7. The OCR results are saved in PostgreSQL.
8. The document status is updated to `OCR_COMPLETED`.
9. The OCR text is passed to the AI Analysis step.

Document statuses used in this section:

```text
QUEUED
PROCESSING
OCR_COMPLETED
FAILED
```

After integrating AI in section 5.6, the system can expand these to:

```text
AI_ANALYZING
COMPLETED
```

---

#### Supported Document Formats

In this workshop, documents should be in formats supported by Amazon Textract:

```text
PDF
PNG
JPG
JPEG
```

Do not upload the following formats directly to the OCR pipeline:

```text
DOCX
XLSX
PPTX
EXE
SH
BAT
```

If you need to process DOCX, XLSX, or PPTX files, convert them to PDF before uploading them to the system.

---

#### Detailed Content

In section **5.5 - Integrate Textract OCR**, we will perform the following steps:

* [Configure Amazon Textract Permissions](5.5.1-configure-textract-permission/)
* [Create OCR Worker](5.5.2-create-ocr-worker/)
* [Test OCR Processing](5.5.3-test-ocr-processing/)

---

#### OCR Data Stored in the Database

Upon successful processing by Textract, the system should save the OCR results in a dedicated table, for example, `OCRResult`.

Example OCR data:

```json
{
  "id": "ocr_001",
  "documentId": "doc_001",
  "provider": "AWS_TEXTRACT",
  "rawText": "Extracted text from document...",
  "createdAt": "2026-06-10T10:00:00.000Z"
}
```

The `Document` table must also be updated with the new status:

```text
OCR_COMPLETED
```

If OCR fails, the status should be updated to:

```text
FAILED
```

and the system should log the error so that users or Admins can investigate it.

---

#### Related Environment Variables

The backend/worker requires the following environment variables:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"

DATABASE_URL="postgresql://username:password@localhost:5432/docmind"
```

If you want to configure a separate region for Textract, you can add:

```env
AWS_TEXTRACT_REGION="ap-southeast-1"
```

In production, the backend/worker should use an **IAM Role** instead of hardcoding AWS Access Keys in the `.env` file.

---

#### Necessary IAM Permissions

The worker requires the following key permissions:

```text
s3:GetObject
s3:ListBucket
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
textract:DetectDocumentText
textract:AnalyzeDocument
```

If using Textract as an asynchronous job, you might also need:

```text
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

These permissions will be configured in detail during the **IAM Role and Policy** section of the workshop.

---

#### Logging During OCR Processing

The worker should log at key processing checkpoints:

```text
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[DOCUMENT_STATUS_UPDATED_TO_OCR_COMPLETED]
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

Do not log the full document content if it contains sensitive information.

---

#### Common Errors

| Error                           | Cause                                       | Solution                                 |
| ------------------------------- | ------------------------------------------- | ---------------------------------------- |
| `AccessDeniedException`         | Worker lacks permissions to invoke Textract | Check the IAM Policy                     |
| `InvalidS3ObjectException`      | Textract cannot read the S3 file            | Check bucket, key, region, and S3 rights  |
| `NoSuchKey`                     | Incorrect `s3Key` or file does not exist    | Verify the object in S3                  |
| `UnsupportedDocumentException`  | Invalid file format                         | Use PDF, PNG, JPG, or JPEG               |
| `QueueDoesNotExist`             | Incorrect Queue URL or region               | Check the `AWS_SQS_QUEUE_URL`            |
| OCR text is empty               | Blur file or contains no readable text      | Use a clearer document file              |

---

#### Completion Checklist

You have completed this step when:

* The worker can retrieve messages from Amazon SQS.
* The worker retrieves the correct document from Amazon S3.
* The worker successfully invokes Amazon Textract.
* OCR text is successfully extracted from the document.
* OCR results are saved in PostgreSQL via Prisma.
* The document status is updated to `OCR_COMPLETED`.
* The message is deleted from SQS upon successful processing.
* The system is ready to proceed to section 5.6 to integrate Gemini/OpenAI AI Analysis.

---

#### Connection to the Next Step

Once OCR is complete, the system will have obtained the textual content of the document. In the next section, this content will be sent to the **AI Gateway** to be analyzed by the **Google Gemini API** or the **OpenAI ChatGPT API**.

The subsequent processing flow:

```text
OCR Text
  |
  v
AI Gateway
  |
  v
Gemini API / OpenAI ChatGPT API
  |
  v
AI Analysis Result
  |
  v
PostgreSQL + Prisma
```

Thus, section 5.5 is responsible for converting raw documents into text, forming the foundation for AI Analysis, AI Chat/RAG, and Enterprise Sandbox features of DocuMind AI.
