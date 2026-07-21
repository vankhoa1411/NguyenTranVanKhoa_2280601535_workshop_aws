---
title : "Backend API and Worker"
date : 2026-06-10
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---

#### Build Backend API and Worker

In this section, we will build the core backend components for the **DocuMind AI** system, including the **Document Upload API**, **SQS Worker Processing**, and the **Document Status API**.

Once the foundational services such as **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Gemini/OpenAI**, and **PostgreSQL + Prisma** are ready, the backend will act as the orchestrator of the entire document processing flow. The backend receives files from users, uploads documents to S3, stores metadata in PostgreSQL, pushes messages into SQS, and provides APIs for the frontend to monitor processing progress.

The worker is a background process that retrieves messages from SQS, invokes Textract to perform OCR on the document, and forwards the extracted OCR text to the AI Gateway for analysis using Gemini or OpenAI. The final results are stored in PostgreSQL and displayed on the Dashboard.

---

#### The Role of Backend API in DocuMind AI

In the **DocuMind AI** project, the Backend API is responsible for:

* Authenticating users using JWT.
* Receiving document uploads from the frontend.
* Validating file formats and sizes.
* Uploading documents to Amazon S3.
* Saving document metadata to PostgreSQL via Prisma.
* Dispatching document processing jobs to Amazon SQS.
* Providing APIs to retrieve document lists.
* Providing APIs to monitor document processing status.
* Returning OCR and AI Analysis results to the frontend.
* Enforcing access controls on documents between Users and Admins.

---

#### The Role of the Worker in DocuMind AI

The worker is a background process that does not interact directly with user requests. The worker's responsibilities include:

* Polling messages from Amazon SQS.
* Parsing message attributes such as `documentId`, `userId`, `s3Bucket`, and `s3Key`.
* Updating the document status to `PROCESSING`.
* Invoking Amazon Textract to perform OCR.
* Storing the OCR results in PostgreSQL.
* Sending the OCR text to the AI Gateway.
* Requesting Gemini or OpenAI to analyze the document.
* Saving the AI Analysis results in PostgreSQL.
* Updating the document status to `COMPLETED` or `FAILED`.
* Deleting messages from SQS upon successful processing.
* Writing execution logs to the console or CloudWatch.

---

#### Backend API and Worker Architecture

The overall processing flow in this section:

```text
React Frontend
  |
  v
Backend API Node.js/Express
  |
  |-- Upload file --> Amazon S3
  |
  |-- Save metadata --> PostgreSQL + Prisma
  |
  |-- Send job --> Amazon SQS
                         |
                         v
                    SQS Worker
                         |
                         v
                  Amazon Textract OCR
                         |
                         v
                    AI Gateway
                         |
          |--------------|--------------|
          v                             v
     Gemini API                  OpenAI ChatGPT API
          |--------------|--------------|
                         v
               PostgreSQL + Prisma
                         |
                         v
              Dashboard / Notification
```

![backend-api-worker](/static/images/5-Workshop/5.8-Backend-API-and-Worker/backend-api-worker.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the processing flow involving the following components: React Frontend, Backend API, Amazon S3, PostgreSQL + Prisma, Amazon SQS, SQS Worker, Amazon Textract, AI Gateway, Gemini API, and OpenAI ChatGPT API.
> Save the image at the path:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/backend-api-worker.png`

---

#### Document Processing Workflow

The document processing workflow in DocuMind AI:

1. The user logs into the system.
2. The user uploads a document from the frontend.
3. The backend validates the file type and file size.
4. The backend uploads the file to Amazon S3.
5. The backend saves the document metadata in PostgreSQL.
6. The backend sends a processing message to Amazon SQS.
7. The worker retrieves the message from SQS.
8. The worker invokes Amazon Textract to perform OCR.
9. The worker stores the OCR Result in PostgreSQL.
10. The worker forwards the OCR text to the AI Gateway.
11. Gemini or OpenAI analyzes the document.
12. The worker stores the AI Analysis results in PostgreSQL.
13. The worker updates the document status to `COMPLETED`.
14. The frontend calls the Document Status API to display the results.
15. The Notification System sends a notification to the user or Admin.

---

#### Document Processing Statuses

Throughout the Backend API and Worker processes, document statuses are updated across the pipeline:

```text
PENDING
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Definitions:

| Status          | Description                                  |
| --------------- | -------------------------------------------- |
| `PENDING`       | Initial metadata creation                    |
| `UPLOADED`      | The file is successfully uploaded to S3      |
| `QUEUED`        | Processing job sent to SQS                   |
| `PROCESSING`    | Background worker has started processing     |
| `OCR_COMPLETED` | Textract OCR is completed                    |
| `AI_ANALYZING`  | Gemini/OpenAI analysis is in progress        |
| `COMPLETED`     | The document has finished processing         |
| `FAILED`        | The processing flow failed                   |

The Frontend Dashboard uses these statuses to display the processing progress to the user.

---

#### Detailed Content

In section **5.8 - Backend API and Worker**, we will perform the following steps:

* [Build Document Upload API](5.8.1-document-upload-api/)
* [Process SQS Worker](5.8.2-sqs-worker-processing/)
* [Build Document Status API](5.8.3-document-status-api/)

---

#### Recommended Source Code Structure

You can organize the backend structure as follows:

```text
src/
├── controllers/
│   └── document.controller.ts
├── routes/
│   └── document.routes.ts
├── services/
│   ├── document.service.ts
│   ├── s3.service.ts
│   ├── sqs.service.ts
│   ├── textract.service.ts
│   └── ai/
│       ├── ai.gateway.ts
│       ├── gemini.service.ts
│       └── openai.service.ts
├── workers/
│   └── document.worker.ts
├── middlewares/
│   ├── auth.middleware.ts
│   └── upload.middleware.ts
├── prisma/
│   └── client.ts
└── config/
    └── aws.config.ts
```

---

#### Core APIs

Core APIs constructed in this section:

| API                               | Purpose                                  |
| --------------------------------- | ---------------------------------------- |
| `POST /api/documents/upload`      | Uploads a document to the system         |
| `GET /api/documents`              | Retrieves the user's document list       |
| `GET /api/documents/:id`          | Retrieves details of a specific document |
| `GET /api/documents/:id/status`   | Monitors document processing status      |
| `GET /api/documents/:id/ocr`      | Retrieves the OCR result text            |
| `GET /api/documents/:id/analysis` | Retrieves the AI Analysis results        |

For the Admin Panel, these can be extended with:

| API                                   | Purpose                                    |
| ------------------------------------- | ------------------------------------------ |
| `GET /api/admin/documents`            | Allows Admin to view all documents         |
| `GET /api/admin/documents/:id/status` | Allows Admin to inspect any document status|
| `GET /api/admin/processing-errors`    | Allows Admin to inspect processing errors  |

---

#### Related Environment Variables

The backend and worker require the following environment variables:

```env
NODE_ENV="development"
PORT=3000

DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"

AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"

AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

> ⚠️ **Security Warning:**
> Do not commit your actual `.env` file to GitHub. In production, use AWS Secrets Manager and IAM Roles instead of hardcoded credentials.

---

#### Required Logging

The backend and worker should write logs at key stages:

```text
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[DOCUMENT_METADATA_SAVED]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_STATUS_COMPLETED]
[DOCUMENT_PROCESSING_FAILED]
```

If CloudWatch is integrated, these logs should be forwarded to the log group:

```text
/docmind/application
```

Avoid logging API keys, JWT tokens, or sensitive document content.

---

#### Security Guidelines

When building the Backend API and Worker, ensure the following constraints:

* Only authenticated users are allowed to upload documents.
* Users can only access their own documents.
* Admins can view all documents if authorized.
* Uploaded files must be validated for format and size.
* Do not expose S3 objects publicly directly.
* Do not expose Gemini/OpenAI API keys to the frontend.
* Do not hardcode AWS Access Keys in production.
* The worker must only delete SQS messages upon successful processing.
* Processing errors must be logged, and the document status updated to `FAILED`.

---

#### Expected Outcomes

Upon completing this section, the system should achieve:

* The backend provides document upload APIs.
* Uploaded files are saved to Amazon S3.
* Metadata is stored in PostgreSQL.
* Processing messages are successfully queued in Amazon SQS.
* The worker can receive messages from SQS.
* The worker processes OCR with Amazon Textract.
* The worker calls the AI Gateway to analyze documents.
* OCR results and AI Analysis results are stored in PostgreSQL.
* The Document Status API returns the correct processing state.
* The frontend can monitor the document processing state.

---

#### Expected Outcome

Following this section, DocuMind AI has a complete backend pipeline from document upload to OCR extraction and AI Analysis. This is the core engine connecting AWS services, PostgreSQL, Gemini/OpenAI, and the frontend dashboard into an automated document processing flow.
