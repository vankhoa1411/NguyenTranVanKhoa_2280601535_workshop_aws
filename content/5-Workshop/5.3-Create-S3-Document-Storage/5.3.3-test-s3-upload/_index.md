---
title : "Test Document Upload to S3"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.3.3. </b> "
---

#### Overview

After creating the S3 bucket and configuring CORS and security settings, the next step is to test the **document upload to Amazon S3** feature for the **DocuMind AI** system.

In this step, we will verify if the backend can receive files from users, upload them to S3, store metadata in **PostgreSQL + Prisma**, and prepare to create document processing jobs for **Amazon SQS**.

This is a critical step because if uploading files to S3 fails, subsequent steps such as the **SQS Processing Queue**, **Textract OCR**, **Gemini/OpenAI AI Analysis**, **Notifications**, and the **Dashboard** will not function correctly.

---

#### Objectives of This Step

Upon completing this step, you will:

* Test if the backend can upload files to S3.
* Confirm that files appear in the S3 bucket.
* Verify that the `s3Key` is generated correctly.
* Verify that document metadata is stored in PostgreSQL.
* Check that the initial document status is `UPLOADED`, `PENDING`, or `QUEUED`.
* Verify error handling when uploading files with unsupported formats.
* Prepare for the step of sending messages to SQS in section 5.4.

---

#### Upload Test Flow

Document upload flow in DocuMind AI:

```text
User
  |
  v
React Frontend / Postman
  |
  v
Backend API: POST /api/documents/upload
  |
  v
Validate file type + file size
  |
  v
Upload file to Amazon S3
  |
  v
Save metadata to PostgreSQL
  |
  v
Return document response to frontend
```

![test-s3-upload](/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.3-test-s3-upload/test-s3-upload.png)

> ⚠️ **Screenshot Suggestion:**
> Capture a screenshot of the successful file upload response from Postman or the frontend, along with an image of the file appearing in the S3 bucket.
> Save the image at the path:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.3-test-s3-upload/test-s3-upload.png`

---

#### Prerequisites Before Testing

Before testing the upload, make sure you have completed the following:

* The S3 bucket is created.
* Block Public Access is enabled on the bucket.
* Encryption is enabled on the bucket.
* CORS is configured if the frontend uploads directly or uses pre-signed URLs.
* The backend is configured with the `AWS_REGION` environment variable.
* The backend is configured with the `AWS_S3_BUCKET_NAME` environment variable.
* Local AWS CLI is configured or the backend runs with an IAM Role.
* PostgreSQL and Prisma are ready.
* The document upload API has been implemented.

Example backend `.env` configuration:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"

DATABASE_URL="postgresql://username:password@localhost:5432/docmind"
```

If using OpenAI/Gemini in later steps, you can also prepare:

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

---

#### Step 1: Prepare a Test File

Prepare a valid file to test the upload.

Supported formats:

```text
PDF
PNG
JPG
JPEG
```

Example test files:

```text
invoice-demo.pdf
receipt-demo.jpg
contract-demo.pdf
```

Do not use the following files to test this step if the system does not yet support conversion:

```text
DOCX
XLSX
PPTX
EXE
SH
BAT
```

For DocuMind AI, PDF or image files are preferred since subsequent steps will process them using **Amazon Textract**.

---

#### Step 2: Start the Backend

Open your terminal in the backend directory and run:

```bash
npm install
```

Then start the server:

```bash
npm run dev
```

Or if your project uses a different start command:

```bash
npm start
```

Verify that the backend is running, for example:

```text
Server is running on port 3000
```

Expected upload API endpoint:

```text
POST http://localhost:3000/api/documents/upload
```

---

#### Step 3: Test Uploading via Postman

Open Postman and create a new request:

```text
Method: POST
URL: http://localhost:3000/api/documents/upload
```

In the **Body** tab, select:

```text
form-data
```

Add the following field:

| Key  | Type | Value            |
| ---- | ---- | ---------------- |
| file | File | invoice-demo.pdf |

If the API requires authentication, add the token in the **Authorization** tab:

```text
Bearer Token
```

Or add the header:

```text
Authorization: Bearer your_jwt_token
```

Then click:

```text
Send
```

---

#### Step 4: Expected API Response

Upon a successful upload, the API should return a response similar to the following:

```json
{
  "success": true,
  "message": "Document uploaded successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "fileType": "application/pdf",
    "fileSize": 204800,
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "UPLOADED",
    "createdAt": "2026-06-10T10:00:00.000Z"
  }
}
```

Some implementations may set the initial status to:

```text
PENDING
```

or:

```text
QUEUED
```

This is valid if your backend sends the job to SQS immediately after uploading.

---

#### Step 5: Verify the File in Amazon S3

Go to the AWS Console:

```text
Amazon S3
```

Select the project bucket:

```text
documind-document-storage
```

Open the corresponding object folder, for example:

```text
uploads/user_001/doc_001/
```

Verify that the file is present:

```text
invoice-demo.pdf
```

Verify the following details:

* Object key
* Size
* Last modified
* Encryption
* Object URL

Note: The Object URL may exist, but it does not mean the file is public. If the bucket is private, accessing the Object URL directly will return Access Denied, which is the correct behavior.

---

#### Step 6: Verify Metadata in PostgreSQL

After a successful upload, query the `Document` table (or equivalent table) in PostgreSQL.

Expected data sample:

```text
id: doc_001
userId: user_001
fileName: invoice-demo.pdf
fileType: application/pdf
fileSize: 204800
s3Bucket: documind-document-storage
s3Key: uploads/user_001/doc_001/invoice-demo.pdf
status: UPLOADED
createdAt: 2026-06-10
```

If using Prisma Studio, run:

```bash
npx prisma studio
```

Then open your browser and inspect the `Document` table.

---

#### Step 7: Check Backend Logs

In the backend terminal, check the logs generated after the upload.

Expected logs:

```text
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[DOCUMENT_METADATA_SAVED]
```

If CloudWatch is integrated, you can also check the log group:

```text
/docmind/application
```

Avoid logging full document content or sensitive information.

---

#### Step 8: Test Unsupported File Type Handling

Try uploading an invalid file format, for example:

```text
test.docx
script.sh
malware.exe
```

The backend should return a clear error:

```json
{
  "success": false,
  "message": "Unsupported file type. Only PDF, PNG, JPG and JPEG are allowed."
}
```

Blocking unsupported file types prevents errors in Amazon Textract, which does not support formats like DOCX, XLSX, or PPTX.

---

#### Step 9: Test File Size Limit Handling

Try uploading a file that exceeds the allowed size limit.

For example, if the backend limit is 10MB, upload a file larger than 10MB.

Expected response:

```json
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

The backend should validate file sizes before uploading to S3 to save unnecessary bandwidth and storage resources.

---

#### Step 10: Verify S3 Permissions

If the upload fails due to AWS permissions, you might see errors like:

```text
AccessDenied
```

or:

```text
User is not authorized to perform: s3:PutObject
```

Troubleshooting steps:

* Ensure local AWS credentials are correct if running locally.
* Ensure the IAM Role has `s3:PutObject` permission.
* Ensure the policy points to the correct bucket name.
* Ensure `AWS_REGION` is correct.
* Ensure `AWS_S3_BUCKET_NAME` is correct.

Minimum permissions required:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

---

#### Step 11: Prepare for Sending Messages to SQS

While the main goal here is successful upload to S3, the backend should be ready to send a message to SQS in the next section.

Expected message payload for SQS:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT"
}
```

If the backend does not send SQS messages yet, that logic will be handled in section **5.4 - Create SQS Processing Queue**.

---

#### Common Errors

| Error                    | Cause                                   | Solution                                      |
| ------------------------ | --------------------------------------- | --------------------------------------------- |
| `AccessDenied`           | Missing S3 permissions                  | Verify the IAM Policy or IAM Role             |
| `NoSuchBucket`           | Incorrect bucket name                   | Check the `AWS_S3_BUCKET_NAME` configuration |
| `PermanentRedirect`      | Incorrect bucket region                 | Check the `AWS_REGION` configuration          |
| `CORS policy error`      | Origin is not allowed                   | Verify CORS settings in S3 or backend         |
| `File too large`         | File size exceeds limit                 | Reduce file size or increase backend limits   |
| `Unsupported file type`  | File format not allowed                 | Use PDF, PNG, JPG, or JPEG                    |
| `Unauthorized`           | Missing JWT token                       | Log in and send the Authorization header      |

---

#### Completion Checklist

You have completed this step when:

* A PDF or image file is successfully uploaded from Postman/frontend.
* The file appears in the S3 bucket.
* The object key matches the expected directory structure.
* Document metadata is successfully saved in PostgreSQL.
* The backend returns a response containing the `documentId`, `s3Bucket`, `s3Key`, and `status`.
* Unsupported file types are blocked.
* Excessively large files are blocked.
* No `AccessDenied`, `NoSuchBucket`, or region mismatch errors occur.
* The system is ready to proceed to section 5.4 to create the SQS queue.

---

#### Expected Outcome

Following this step, the DocuMind AI system is capable of successfully uploading and storing documents on Amazon S3. The document file is managed using its `s3Key`, its metadata is saved in PostgreSQL, and the system is ready to feed documents into the asynchronous processing pipeline utilizing Amazon SQS, Amazon Textract, and the AI Gateway.
