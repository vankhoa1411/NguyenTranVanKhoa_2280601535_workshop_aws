---
title : "Build Document Upload API"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.8.1. </b> "
---

#### Overview

In this step, we will build the **Document Upload API** for the **DocuMind AI** system.

This API allows users to upload documents to the system. The backend receives the file, validates its format and size, uploads it to **Amazon S3**, stores the metadata in **PostgreSQL via Prisma**, and sends a processing job to **Amazon SQS** so that the worker can perform OCR and AI Analysis in subsequent steps.

This is the most critical API in the pipeline, as the entire document processing cycle is initiated by the upload step.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create the document upload API endpoint.
* Validate the file type and file size before uploading.
* Upload files to Amazon S3.
* Store document metadata in PostgreSQL.
* Generate the correct `s3Bucket` and `s3Key` references for the document.
* Dispatch document processing tasks to Amazon SQS.
* Return a clear response payload to the frontend.

---

#### Document Upload API Workflow

The upload API processing flow in DocuMind AI:

```text id="1e340w"
React Frontend / Postman
  |
  v
POST /api/documents/upload
  |
  v
Validate file + user
  |
  v
Upload file to Amazon S3
  |
  v
Save document metadata to PostgreSQL
  |
  v
Send processing message to Amazon SQS
  |
  v
Return response to user
```

![document-upload-api](/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.1-document-upload-api/document-upload-api.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the Postman or frontend screen showing a successful document upload, alongside backend logs displaying the file uploaded to S3 and the message queued in SQS.
> Save the image at the path:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.1-document-upload-api/document-upload-api.png`

---

#### Recommended Endpoint

The document upload API:

```text id="jlm42d"
POST /api/documents/upload
```

Request content type:

```text id="i1yk76"
multipart/form-data
```

Upload form fields:

| Key           | Type | Description              |
| ------------- | ---- | ------------------------ |
| `file`        | File | PDF, PNG, JPG, or JPEG file |
| `title`       | Text | Optional document title  |
| `description` | Text | Optional description     |

If the API requires authentication, include the JWT token:

```text id="jaqfiv"
Authorization: Bearer your_jwt_token
```

---

#### Supported File Types

The backend should only accept formats supported by Amazon Textract:

```text id="pxe6br"
PDF
PNG
JPG
JPEG
```

Valid MIME types:

```text id="4idhln"
application/pdf
image/png
image/jpeg
```

Do not allow:

```text id="757wrh"
DOCX
XLSX
PPTX
EXE
SH
BAT
JS
```

If you need to process DOCX, XLSX, or PPTX, they should be converted to PDF before entering the OCR pipeline.

---

#### File Size Limit

For workshop/demo environments, you can enforce:

```text id="6bz1qx"
Max file size: 10MB
```

For larger documents, you can increase this to:

```text id="jzrn1k"
Max file size: 20MB
```

Response when the file is too large:

```json id="5i1531"
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

---

#### Recommended Backend File Structure

You can organize the source code as follows:

```text id="tokybt"
src/
├── controllers/
│   └── document.controller.ts
├── routes/
│   └── document.routes.ts
├── services/
│   ├── document.service.ts
│   ├── s3.service.ts
│   └── sqs.service.ts
├── middlewares/
│   ├── auth.middleware.ts
│   └── upload.middleware.ts
└── prisma/
    └── client.ts
```

File descriptions:

| File                     | Description                              |
| ------------------------ | ---------------------------------------- |
| `document.controller.ts` | Handles the upload request/response      |
| `document.routes.ts`     | Declares the upload route                |
| `document.service.ts`    | Stores metadata and updates document states|
| `s3.service.ts`          | Handles file uploads to Amazon S3        |
| `sqs.service.ts`         | Handles queuing messages in Amazon SQS   |
| `upload.middleware.ts`   | Processes multipart/form-data payloads   |
| `auth.middleware.ts`     | Validates user authentication states     |

---

#### Configuring Multer Upload

If using `multer`, install the package:

```bash id="h25yo8"
npm install multer
npm install --save-dev @types/multer
```

Example middleware setup:

```ts id="swzf9y"
import multer from "multer";
 
const storage = multer.memoryStorage();
 
export const upload = multer({
  storage,
  limits: {
    fileSize: 10 * 1024 * 1024
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ["application/pdf", "image/png", "image/jpeg"];
 
    if (!allowedTypes.includes(file.mimetype)) {
      return cb(new Error("Unsupported file type"));
    }
 
    cb(null, true);
  }
});
```

---

#### Upload Files to S3

Example S3 upload service:

```ts id="n328gn"
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
 
const s3Client = new S3Client({
  region: process.env.AWS_REGION
});
 
export async function uploadFileToS3(params: {
  buffer: Buffer;
  fileName: string;
  mimeType: string;
  userId: string;
  documentId: string;
}) {
  const bucket = process.env.AWS_S3_BUCKET_NAME!;
 
  const s3Key = `uploads/${params.userId}/${params.documentId}/${params.fileName}`;
 
  await s3Client.send(
    new PutObjectCommand({
      Bucket: bucket,
      Key: s3Key,
      Body: params.buffer,
      ContentType: params.mimeType
    })
  );
 
  return {
    s3Bucket: bucket,
    s3Key
  };
}
```

---

#### Save Metadata to PostgreSQL

After a successful S3 upload, write the metadata to the `Document` table.

Example:

```ts id="dxi3cq"
const document = await prisma.document.create({
  data: {
    userId,
    fileName: file.originalname,
    fileType: file.mimetype,
    fileSize: file.size,
    s3Bucket,
    s3Key,
    status: "UPLOADED"
  }
});
```

Once the message is sent to SQS, update the status:

```ts id="77zvuc"
await prisma.document.update({
  where: { id: document.id },
  data: { status: "QUEUED" }
});
```

---

#### Queue Messages in SQS

Message structure sent to the queue:

```json id="x6bkcb"
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Example SQS queue sender service:

```ts id="r2eqza"
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";
 
const sqsClient = new SQSClient({
  region: process.env.AWS_REGION
});
 
export async function sendDocumentProcessingMessage(payload: any) {
  await sqsClient.send(
    new SendMessageCommand({
      QueueUrl: process.env.AWS_SQS_QUEUE_URL!,
      MessageBody: JSON.stringify(payload)
    })
  );
}
```

---

#### Expected API Response

Upon a successful upload:

```json id="fn8ubf"
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "fileType": "application/pdf",
    "fileSize": 204800,
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED",
    "createdAt": "2026-06-10T10:00:00.000Z"
  }
}
```

When the file is missing from the payload:

```json id="2ojr5h"
{
  "success": false,
  "message": "No file uploaded."
}
```

When an unsupported file format is provided:

```json id="a7sm44"
{
  "success": false,
  "message": "Unsupported file type. Only PDF, PNG, JPG and JPEG are allowed."
}
```

---

#### Postman Verification

Create a request:

```text id="x6369y"
Method: POST
URL: http://localhost:3000/api/documents/upload
```

Body parameters:

```text id="5sbjcu"
form-data
```

Add field:

| Key    | Type | Value            |
| ------ | ---- | ---------------- |
| `file` | File | invoice-demo.pdf |

If authenticated, include:

```text id="brkhqd"
Authorization: Bearer your_jwt_token
```

---

#### Expected Backend Logs

The backend should print logs detailing the process:

```text id="hff4l5"
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[DOCUMENT_METADATA_SAVED]
[SQS_SEND_MESSAGE_SUCCESS]
[DOCUMENT_STATUS_UPDATED_TO_QUEUED]
```

On execution errors:

```text id="kec972"
[DOCUMENT_UPLOAD_FAILED]
[S3_UPLOAD_FAILED]
[SQS_SEND_MESSAGE_FAILED]
```

---

#### Common Errors

| Error                   | Cause                         | Solution                               |
| ----------------------- | ----------------------------- | -------------------------------------- |
| `No file uploaded`      | Field `file` missing          | Check your form-data request keys      |
| `Unsupported file type` | Unsupported file format       | Upload PDF, PNG, JPG, or JPEG files    |
| `AccessDenied`          | Missing S3 bucket permissions | Verify IAM S3 policy configuration     |
| `NoSuchBucket`          | Incorrect S3 bucket name      | Check your `.env` bucket configuration |
| `QueueDoesNotExist`     | Incorrect SQS Queue URL       | Check your `AWS_SQS_QUEUE_URL` value   |
| `Unauthorized`          | Invalid JWT token             | Re-login and include the Authorization header|

---

#### Completion Checklist

You have completed this step when:

* The upload API is active.
* Backend validates the file type and file size.
* Files are uploaded to S3.
* Document metadata is saved to PostgreSQL.
* Messages are successfully sent to SQS.
* The document status is updated to `QUEUED`.
* The API returns clear response payloads.
* Backend logs represent the entire upload process.

---

#### Expected Outcome

Following this step, DocuMind AI has a fully operational document upload API. Users can upload documents, which are saved in Amazon S3, indexed in PostgreSQL, and queued in Amazon SQS for subsequent OCR and AI Analysis processing by the worker.
