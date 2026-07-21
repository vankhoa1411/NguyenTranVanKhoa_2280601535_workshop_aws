---
title : "Build Document Status API"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.8.3. </b> "
---

#### Overview

In this step, we will build the **Document Status API** for the **DocuMind AI** system.

After a user uploads a document, the OCR extraction and AI Analysis processes are handled asynchronously by the background worker. Therefore, the frontend needs an API to check the document's processing status in real time or via a polling mechanism.

The Document Status API allows the frontend to determine which stage a document is in: uploaded, pending, OCR in progress, AI analysis running, completed, or failed.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an API endpoint to retrieve the processing status of a document.
* Return the current processing status of a document.
* Return the OCR extraction results once available.
* Return the AI Analysis results once completed.
* Support frontend polling or dashboard refreshes.
* Enable users to monitor document processing progress.
* Allow Admins to inspect failed documents.

---

#### Document Status API Workflow

The status checking flow:

```text id="fdr36l"
React Frontend Dashboard
  |
  v
GET /api/documents/:id/status
  |
  v
Backend API
  |
  v
PostgreSQL + Prisma
  |
  v
Return document status + progress
```

![document-status-api](/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.3-document-status-api/document-status-api.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the Postman or Dashboard screen displaying the document status as `QUEUED`, `PROCESSING`, `AI_ANALYZING`, or `COMPLETED`.
> Save the image at the path:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.3-document-status-api/document-status-api.png`

---

#### Recommended Endpoints

Retrieve document status:

```text id="rpwha8"
GET /api/documents/:id/status
```

Retrieve document details:

```text id="8g65mo"
GET /api/documents/:id
```

Retrieve document lists for the authenticated user:

```text id="0gi75b"
GET /api/documents
```

Admin-specific endpoints:

```text id="vu32mo"
GET /api/admin/documents/:id/status
```

---

#### Standard Status Responses

Example when the document is queued:

```json id="l8etff"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "QUEUED",
    "progress": 25,
    "message": "Document is waiting for OCR processing."
  }
}
```

Example when OCR is running:

```json id="72pwpz"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "PROCESSING",
    "progress": 45,
    "message": "OCR processing is running."
  }
}
```

Example when AI analysis is running:

```json id="vvqkz4"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "AI_ANALYZING",
    "progress": 75,
    "message": "AI analysis is running."
  }
}
```

Example upon successful completion:

```json id="zy0x3y"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "COMPLETED",
    "progress": 100,
    "message": "Document processing completed.",
    "ocrResult": {
      "provider": "AWS_TEXTRACT",
      "rawTextPreview": "Invoice No: INV-2026-001..."
    },
    "aiAnalysis": {
      "provider": "openai",
      "model": "gpt-4.1-mini",
      "documentType": "Invoice",
      "summary": "This document is an invoice from AWS.",
      "confidence": 0.9
    }
  }
}
```

Example on execution failure:

```json id="0pkmfh"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "FAILED",
    "progress": 0,
    "message": "Document processing failed.",
    "error": "Unsupported document format."
  }
}
```

---

#### Status to Progress Mapping

The frontend can use the progress value to display a progress bar.

| Status          | Progress | Message             |
| --------------- | -------: | ------------------- |
| `PENDING`       |        5 | Metadata created    |
| `UPLOADED`      |       15 | File uploaded to S3 |
| `QUEUED`        |       25 | Waiting for worker  |
| `PROCESSING`    |       45 | OCR processing      |
| `OCR_COMPLETED` |       60 | OCR completed       |
| `AI_ANALYZING`  |       75 | AI analysis running |
| `COMPLETED`     |      100 | Completed           |
| `FAILED`        |        0 | Processing failed   |

---

#### Access Control Enforcement

The Document Status API must validate user permissions:

* Users can only access their own documents.
* Admins can view all documents.
* If a user requests a document they do not own, return `403 Forbidden`.
* If a document is not found, return `404 Not Found`.

Example forbidden response:

```json id="9hrl2d"
{
  "success": false,
  "message": "You do not have permission to access this document."
}
```

---

#### Example Backend Logic

```ts id="7fmj2z"
export async function getDocumentStatus(documentId: string, user: any) {
  const document = await prisma.document.findUnique({
    where: { id: documentId },
    include: {
      ocrResult: true,
      aiAnalysis: true
    }
  });
 
  if (!document) {
    throw new Error("Document not found");
  }
 
  if (user.role !== "ADMIN" && document.userId !== user.id) {
    throw new Error("Forbidden");
  }
 
  return {
    documentId: document.id,
    fileName: document.fileName,
    status: document.status,
    progress: mapStatusToProgress(document.status),
    ocrResult: document.ocrResult,
    aiAnalysis: document.aiAnalysis
  };
}
```

---

#### Progress Mapper Function

```ts id="aobnz2"
function mapStatusToProgress(status: string) {
  const progressMap: Record<string, number> = {
    PENDING: 5,
    UPLOADED: 15,
    QUEUED: 25,
    PROCESSING: 45,
    OCR_COMPLETED: 60,
    AI_ANALYZING: 75,
    COMPLETED: 100,
    FAILED: 0
  };
 
  return progressMap[status] ?? 0;
}
```

---

#### Postman Verification

Create a request:

```text id="5gs0q7"
GET http://localhost:3000/api/documents/doc_001/status
```

Header if authenticated:

```text id="kz2axv"
Authorization: Bearer your_jwt_token
```

Expected output:

```json id="9qs7k8"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "status": "COMPLETED",
    "progress": 100
  }
}
```

---

#### Frontend Polling Implementation

The frontend should poll this API periodically while a document's status remains incomplete.

Example client-side logic:

```text id="mx13jn"
If status is NOT COMPLETED or FAILED:
  re-trigger status API after 3-5 seconds
If COMPLETED:
  display AI Analysis summary results
If FAILED:
  display error alert details
```

To avoid overloading the backend, keep request frequencies reasonable.

Recommended settings:

```text id="rq73uj"
Polling interval: 3-5 seconds
```

---

#### Required Logging

The backend should print logs:

```text id="d1ci6i"
[DOCUMENT_STATUS_REQUESTED]
[DOCUMENT_STATUS_RETURNED]
[DOCUMENT_NOT_FOUND]
[DOCUMENT_ACCESS_FORBIDDEN]
```

Do not log passwords or full document contents.

---

#### Completion Checklist

You have completed this step when:

* The document status API endpoint is functional.
* The API returns the correct status name.
* The API computes the correct progress value.
* Users can only access their own documents.
* Admins can view documents belonging to any user.
* The frontend can poll the document status.
* When a document's status is `COMPLETED`, the response payload includes OCR and AI Analysis results.
* When a document's status is `FAILED`, the response payload includes a clear error message.

---

#### Expected Outcome

Following this step, DocuMind AI has a Document Status API enabling the frontend to monitor document processing progress. Users can view the step-by-step progress in the pipeline, and Admins can identify and inspect failed documents.
