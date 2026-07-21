---
title : "Kiểm tra Full Workflow"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.13.3. </b> "
---

#### Overview

In this step, we will verify the **full workflow** of the **DocuMind AI** system after completing backend deployment and production configuration.

The full workflow encompasses user authentication, document uploading, file persistence in Amazon S3, job queueing in Amazon SQS, Textract OCR processing, AI Analysis via Gemini/OpenAI, database storage in PostgreSQL, and real-time visualization on the frontend Dashboard.

This step validates that the entire system functions end-to-end.

---

#### Objectives of This Step

Upon completing this step, you will:

* Verify user authentication endpoints function.
* Test document uploading from the frontend UI or Postman.
* Confirm document files are successfully saved in Amazon S3.
* Confirm processing jobs are queued in Amazon SQS.
* Verify the background worker retrieves and processes SQS messages.
* Verify Amazon Textract OCR extraction succeeds.
* Verify Gemini/OpenAI parses and analyzes extracted text.
* Confirm that processing results are saved in PostgreSQL database tables.
* Confirm the Dashboard displays document statuses and results.
* Review notifications, audit logs, and security logs.

---

#### Full Workflow Architecture

```text id="full-workflow"
User
  |
  v
Frontend Dashboard
  |
  v
Backend API on EC2
  |
  |-- Upload file --> Amazon S3
  |-- Save metadata --> PostgreSQL + Prisma
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
       OpenAI API                  Gemini API
           |--------------|--------------|
                          v
                Save AIAnalysis
                          |
                          v
                   Frontend Dashboard
```

![test-full-workflow](/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.3-test-full-workflow/test-full-workflow.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the frontend Dashboard showing successfully processed documents, the S3 bucket objects view, active SQS queue metrics, and the worker terminal log outputs displaying OCR and AI completion events.
> Save the image at the path:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.3-test-full-workflow/test-full-workflow.png`

---

#### Pre-requisites Before Testing

Before testing, ensure:

* The Backend API is running on the EC2 instance.
* The Worker process is active on the server.
* PostgreSQL database services are active.
* Prisma schema migrations have been applied.
* The S3 bucket exists.
* SQS queues are created.
* EC2 IAM Roles are configured with proper policy scopes.
* Amazon Textract possesses permissions to read from the S3 bucket.
* Gemini/OpenAI API keys are active and funded.
* The frontend configuration points to the EC2 backend URL.
* CloudWatch log streams are active (if configured).

---

#### Step 1: Verify Backend API Health

Invoke the health check endpoint:

```text id="health-url"
GET http://your-ec2-public-ip:3000/api/health
```

Expected response payload:

```json id="health-ok"
{
  "success": true,
  "message": "DocuMind AI backend is running"
}
```

Verify database connectivity:

```text id="db-health-url"
GET http://your-ec2-public-ip:3000/api/health/database
```

Expected response payload:

```json id="db-health-ok"
{
  "success": true,
  "message": "Database connection is healthy"
}
```

---

#### Step 2: Verify Login Endpoint

Submit credentials via authentication APIs:

```text id="login-api"
POST /api/auth/login
```

Request payload:

```json id="login-body"
{
  "email": "demo@docmind.ai",
  "password": "your-password"
}
```

Expected response payload:

```json id="login-response"
{
  "success": true,
  "accessToken": "jwt-token",
  "user": {
    "email": "demo@docmind.ai",
    "role": "USER"
  }
}
```

---

#### Step 3: Test Document Uploads

Post a document file using Postman or the frontend:

```text id="upload-api"
POST /api/documents/upload
```

Body structure:

```text id="upload-form"
multipart/form-data
file: invoice-demo.pdf
```

Request headers:

```text id="auth-header"
Authorization: Bearer your_jwt_token
```

Expected response payload:

```json id="upload-response"
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED"
  }
}
```

---

#### Step 4: Verify S3 Storage Files

Navigate to the Amazon S3 Console:

* `Amazon S3`
* `→ documind-document-storage`
* `→ uploads/`

Confirm that the uploaded file object is present.

Or verify via the AWS CLI:

```bash id="s3-ls-upload"
aws s3 ls s3://documind-document-storage/uploads/ --recursive
```

---

#### Step 5: Verify SQS Message Queues

Navigate to the Amazon SQS Console:

* `Amazon SQS`
* `→ docmind-document-processing-queue`

Check message counts.

*Note: If the background worker process is running, messages are consumed and processed quickly, meaning the queue may report empty. In this scenario, inspect worker logs.*

---

#### Step 6: Verify Background Worker Logs

Access the EC2 instance terminal and view worker logs:

```bash id="pm2-worker-log"
pm2 logs documind-worker
```

Expected log events:

```text id="worker-log-expected"
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

---

#### Step 7: Verify Database Entries

Launch Prisma Studio to inspect database records:

```bash id="prisma-studio"
npx prisma studio
```

Inspect the tables:

* `Document`
* `OCRResult`
* `AIAnalysis`
* `Notification`
* `AuditLog`

Verify that the document record status transitions to:

```text id="completed-status"
COMPLETED
```

---

#### Step 8: Verify Dashboard Integration

Open the frontend Dashboard:

1. Log in to your account.
2. Open the Document Dashboard.
3. Locate the recently uploaded document.
4. Confirm that the status is reported as `COMPLETED`.
5. Open the Document Details view.
6. Verify the extracted OCR text is visible.
7. Verify the AI Analysis summary is populated.
8. Test chat functionality (if integrated).

---

#### Step 9: Verify Notifications

Check the notification interface:

```text id="notification-check"
Document uploaded successfully.
AI analysis completed.
```

If a processing error occurred, verify failure notifications are dispatched:

```text id="notification-failed"
Document processing failed.
```

---

#### Step 10: Verify Audit Logs

As an administrator, review Audit Log action records:

```text id="audit-actions"
USER_LOGIN
DOCUMENT_UPLOADED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
```

On execution errors:

```text id="audit-error-actions"
DOCUMENT_PROCESSING_FAILED
AI_PROVIDER_FALLBACK_USED
SECURITY_FORBIDDEN_ACCESS
```

---

#### Step 11: Verify CloudWatch Logs

Navigate to the CloudWatch Console:

* `CloudWatch`
* `→ Logs`
* `→ Log groups`
* `→ /docmind/application`

Verify the log events written by backend and worker processes.

---

#### Step 12: Verify AI Chat/RAG APIs

Post a chat query request:

```text id="chat-api"
POST /api/ai/chat
```

Request payload:

```json id="chat-body"
{
  "documentId": "doc_001",
  "question": "Summarize this document.",
  "provider": "openai"
}
```

Expected response payload:

```json id="chat-response"
{
  "success": true,
  "answer": "This document is an invoice...",
  "provider": "openai",
  "model": "gpt-4.1-mini"
}
```

---

#### Full Workflow Verification Matrix

| Step           | Expected Outcome                           |
| -------------- | ------------------------------------------ |
| Backend health | API returns success response               |
| User login     | Returns valid JWT token                    |
| Upload         | Document uploads successfully              |
| S3 Storage     | Object appears in the S3 bucket            |
| SQS Queue      | Message is sent and picked up by worker    |
| Worker Logs    | Worker prints full processing logs         |
| Textract OCR   | OCR text payload is created                |
| AI Gateway     | Gemini/OpenAI returns summary results      |
| Database       | Document, OCR, and AI records are updated  |
| Dashboard      | Status and analysis values populate UI     |
| Notifications  | Dispatches success alerts                  |
| Audit Log      | Records key operations                     |

---

#### Troubleshoot Scenarios

| Error                         | Cause                                    | Solution                                  |
| ----------------------------- | ---------------------------------------- | ----------------------------------------- |
| Health check API fails        | Server is offline or port is blocked     | Check PM2 processes and Security Groups   |
| Authentication fails          | Invalid user keys or JWT settings        | Verify Auth API configurations            |
| Upload fails                  | CORS headers, file sizes, or S3 policies  | Check S3 IAM policies and CORS settings   |
| S3 object is missing          | S3 client upload method triggered errors | Review S3 console and backend error logs  |
| SQS message not sent          | Backend SQS client failed to queue job   | Inspect backend SQS configurations        |
| Worker ignores messages       | Worker process offline or wrong queue URL| Check PM2 process lists and SQS URL values|
| Textract operations fail      | File format error or IAM permissions error| Check S3 path and Textract IAM policies   |
| AI gateway triggers errors    | Wrong API keys or account quota exceeded | Check Gemini/OpenAI console budgets       |
| Status remains unchanged      | Worker failed to write updates to DB     | Check Prisma update transaction queries   |
| Dashboard does not update     | Polling or WebSockets are disconnected    | Verify document status polling interval   |

---

#### Completion Checklist

You have completed this step when:

* The Backend API is accessible on the EC2 instance.
* The SQS Worker runs stably.
* User login succeeds.
* Documents upload successfully.
* Uploaded files appear in the S3 bucket.
* Processing jobs are queued in SQS.
* SQS Worker fetches and consumes queue messages.
* Textract OCR extraction completes successfully.
* AI Gateway returns Gemini/OpenAI analysis summaries.
* Relational database tables update successfully.
* Frontend Dashboard visualizes document analysis detail.
* User notifications and Audit Logs are generated.
* Production logs appear in CloudWatch.

---

#### Expected Outcome

Following this step, DocuMind AI has verified end-to-end functionality. The system processes document payloads from upload to OCR, AI analysis, database storage, and real-time visualization on the Dashboard.
