---
title : "Deployment and Test"
date : 2026-06-10
weight : 13
chapter : false
pre : " <b> 5.13. </b> "
---

#### System Deployment and Testing

In this section, we will perform the **Deployment and Test** phase for the **DocuMind AI** system.

Now that the core components—including Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, PostgreSQL + Prisma, Backend API, Worker, Frontend Dashboard, Notifications, Admin Panel, IAM Roles, Monitoring, and Security—are ready, the final phase is deploying the backend services to a server environment and validating the end-to-end workflow.

The objective of this section is to ensure that the system executes stably on a production-like server environment rather than just running locally.

---

#### The Role of Deployment in DocuMind AI

Deployment transitions the application from a local development workspace to a live server.

In DocuMind AI, the backend and worker must be deployed to:

* Accept user requests from the frontend client.
* Process login operations and user authorization.
* Upload document payloads to Amazon S3.
* Queue document processing tasks in Amazon SQS.
* Run background worker loops to pull jobs from SQS.
* Call Amazon Textract to perform OCR extractions.
* Call Gemini/OpenAI to run AI Analysis tasks.
* Persist results to PostgreSQL.
* Return analysis details back to the user Dashboard.

---

#### The Role of Testing in DocuMind AI

Testing verifies that all components integrate and execute correctly.

Areas to test include:

* Backend API health checks.
* User authentication flows.
* Document upload operations.
* S3 object storage persistence.
* SQS message queueing.
* Worker processing loops.
* Textract OCR extraction.
* Gemini/OpenAI AI Analysis execution.
* PostgreSQL database persistence.
* Document Dashboard updating.
* User notifications.
* System audit logs.
* Admin Panel features.
* CloudWatch log streams.
* IAM permission guards.
* Common troubleshoot scenarios.

---

#### Deployment and Testing Architecture

System workflow after deployment:

```text id="deployment-test-flow"
Frontend Dashboard
  |
  v
Backend API on EC2
  |
  |-- Upload file --> Amazon S3
  |-- Send job --> Amazon SQS
  |-- Save metadata --> PostgreSQL + Prisma
  |
  v
SQS Worker on EC2
  |
  |-- Read file from S3
  |-- OCR using Amazon Textract
  |-- Analyze using Gemini/OpenAI
  |-- Save OCRResult + AIAnalysis
  |
  v
Frontend shows completed result
```

![deployment-test-flow](/static/images/5-Workshop/5.13-Deployment-and-Test/deployment-test-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the Frontend calling the Backend API on EC2, the backend uploading files to S3, queueing messages in SQS, the worker executing Textract OCR, calling Gemini/OpenAI, and saving results to PostgreSQL.
> Save the image at the path:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/deployment-test-flow.png`

---

#### Detailed Content

In section **5.13 - Deployment and Test**, we will perform the following steps:

* [Deploy Backend on EC2](5.13.1-deploy-backend-ec2/)
* [Configure Production Environment](5.13.2-configure-env-production/)
* [Test Full Workflow](5.13.3-test-full-workflow/)
* [Common Errors](5.13.4-common-errors/)

---

#### Deployed Components

The core components to deploy in this workshop are:

```text id="deploy-components"
Backend API
SQS Worker
```

The Backend API is responsible for:

* Authentication API endpoints.
* Document Upload API endpoints.
* Document Status API endpoints.
* OCR Result API endpoints.
* AI Analysis API endpoints.
* Notification API endpoints.
* Admin Panel API endpoints.

The SQS Worker is responsible for:

* Fetching messages from SQS.
* Reading document files from S3.
* Invoking Textract OCR.
* Calling the AI Gateway.
* Saving processing results.
* Updating document statuses.
* Creating notifications and audit logs.

---

#### Deployment Stack

Technologies used for deployment:

| Technology      | Description                               |
| --------------- | ----------------------------------------- |
| Amazon EC2      | Server hosting the backend and worker     |
| Ubuntu Server   | OS running on the EC2 instance            |
| Node.js         | Runtime environment for application code  |
| PM2             | Process manager for backend and worker    |
| Git             | Code versioning and repository clone      |
| PostgreSQL      | Primary relational database               |
| Prisma          | Database migrations and client query tool |
| AWS IAM Role    | Authorizes EC2 instances to access AWS    |
| CloudWatch Logs | Stores production application logs        |

---

#### Production Settings to Prepare

Required configuration parameter groups:

```text id="production-config-groups"
App Config
Database Config
AWS Config
AI Provider Config
Auth Config
CORS Config
CloudWatch Config
Secrets Manager Config
```

Example `.env.production` setup:

```env id="sample-env-production"
NODE_ENV="production"
PORT=3000

DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"

AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"

AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"

JWT_SECRET="your-strong-jwt-secret"

CORS_ORIGIN="https://your-frontend-domain.com"

CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-production"
CLOUDWATCH_WORKER_LOG_STREAM="worker-production"
```

If utilizing AWS Secrets Manager, the production configuration only requires:

```env id="sample-secrets-env"
NODE_ENV="production"
PORT=3000

USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

---

#### Note on PostgreSQL Database

In this workshop, DocuMind AI utilizes:

```text id="postgres-prisma"
PostgreSQL + Prisma
```

Amazon RDS is not used in this section.

PostgreSQL can run:

* On the same EC2 instance hosting the backend (for demo environments).
* On a separate database server.
* On managed database services for scalability.

Ensure `DATABASE_URL` is configured correctly so the backend can establish connections.

---

#### Note on IAM Role

When running backend services on EC2, attach the IAM Role created in step 5.11 to the instance.

The role must possess access privileges for:

* Amazon S3.
* Amazon SQS.
* Amazon Textract.
* CloudWatch Logs.
* AWS Secrets Manager.

When utilizing the IAM Role, omit access keys from environment files:

```env id="bad-aws-keys"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Verify the active IAM Role using:

```bash id="sts-check"
aws sts get-caller-identity
```

Expected output assumed-role:

```text id="assumed-role"
assumed-role/DocuMindBackendWorkerRole
```

---

#### Managing Applications via PM2

After compiling backend services, manage execution via PM2.

Launch Backend API:

```bash id="pm2-backend"
pm2 start dist/server.js --name documind-backend
```

Launch Worker Process:

```bash id="pm2-worker"
pm2 start dist/workers/document.worker.js --name documind-worker
```

Verify active processes:

```bash id="pm2-list"
pm2 list
```

Inspect logs:

```bash id="pm2-logs"
pm2 logs documind-backend
pm2 logs documind-worker
```

Save process lists to reload on server restarts:

```bash id="pm2-save"
pm2 startup
pm2 save
```

---

#### Verification Steps

E2E validation steps after deployment:

1. Test backend health check APIs.
2. Log in with a user account.
3. Upload a sample document.
4. Verify document payload persists in S3.
5. Verify document processing message is queued in SQS.
6. Check worker log outputs.
7. Confirm Textract OCR results are returned.
8. Confirm AI Analysis summaries are generated.
9. Verify database tables store processing details.
10. Check frontend Dashboard status updates.
11. Confirm User Notifications are received.
12. Review the Audit Log.
13. Inspect CloudWatch logs.

---

#### Verification of Processing Statuses

During document processing, status properties must progress:

```text id="status-flow"
PENDING
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
```

On execution failures:

```text id="failed-status"
FAILED
```

The frontend Dashboard displays these statuses to inform users of progress.

---

#### Health Check APIs

The backend exposes a health check endpoint:

```text id="health-api"
GET /api/health
```

Expected response:

```json id="health-response"
{
  "success": true,
  "message": "DocuMind AI backend is running"
}
```

Verify database connectivity via:

```text id="database-health-api"
GET /api/health/database
```

Expected response:

```json id="database-health-response"
{
  "success": true,
  "message": "Database connection is healthy"
}
```

---

#### Database Records Validation

Upon successful E2E execution, verify records in PostgreSQL tables:

| Table          | Expected Data                                       |
| -------------- | --------------------------------------------------- |
| `User`         | User profiles and credential records                |
| `Document`     | Document metadata and processing statuses           |
| `OCRResult`    | Text parsed by Amazon Textract                      |
| `AIAnalysis`   | Extraction summaries returned by Gemini/OpenAI      |
| `Notification` | User and admin alert records                        |
| `AuditLog`     | Trace logs of critical actions                      |

Inspect tables via Prisma Studio:

```bash id="prisma-studio"
npx prisma studio
```

---

#### Production Logging Reference

Log statements that should appear:

```text id="production-logs"
[APP_STARTED]
[DATABASE_CONNECTED]
[SQS_WORKER_STARTED]
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_MESSAGE_RECEIVED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_STATUS_COMPLETED]
```

On operational errors:

```text id="error-logs"
[DOCUMENT_PROCESSING_FAILED]
[TEXTRACT_OCR_FAILED]
[AI_PROVIDER_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[SECURITY_FORBIDDEN_ACCESS]
```

---

#### Deployment Troubleshoot Categories

Common deployment and testing bottlenecks:

| Category        | Example Issues                                     |
| --------------- | -------------------------------------------------- |
| EC2             | SSH access failures, blocked security group ports  |
| Node.js         | Compilation errors, missing dependencies           |
| PM2             | Backend or worker process loop crashes             |
| PostgreSQL      | Incorrect `DATABASE_URL`, migration errors         |
| Prisma          | Un-generated client types, missing schema tables   |
| S3              | `AccessDenied` errors, incorrect bucket names      |
| SQS             | Typos in SQS Queue URL, worker fails to fetch      |
| Textract        | Unsupported image sizes, `InvalidS3ObjectException` |
| AI Provider     | Invalid API keys, quota exhaustion, model mismatches|
| IAM             | Insufficient IAM Policy rights                      |
| CORS            | Client requests blocked by CORS headers            |
| CloudWatch      | Logging exceptions or stream failures              |

Detailed troubleshoot solutions are documented in **5.13.4 - Common Errors**.

---

#### Pre-Completion Checklist

Confirm before finishing this section:

* Backend APIs run on EC2.
* SQS Worker runs on EC2.
* Environment variables or Secrets Manager settings are active.
* PostgreSQL connection is established.
* Prisma migrations are applied in production.
* EC2 instance has correct IAM Roles attached.
* Health check endpoint returns success.
* Authentication and user login works.
* File uploads complete successfully.
* Document payloads store in S3.
* Messages are queued in SQS.
* Worker processes retrieve queue items.
* Textract OCR extracts text.
* Gemini/OpenAI completes document analysis.
* Database updates records.
* Frontend Dashboard reports `COMPLETED`.
* Notifications and Audit Logs are generated.
* CloudWatch records application logs.

---

#### Expected Outcome

Following this section, DocuMind AI has been deployed and verified end-to-end. The Backend API and Worker run on EC2, connecting PostgreSQL, Amazon S3, Amazon SQS, Amazon Textract, and Gemini/OpenAI to update the document Dashboard in real-time.

This confirms that the system is ready for demonstrations, reports, or scaling to a live production environment.
