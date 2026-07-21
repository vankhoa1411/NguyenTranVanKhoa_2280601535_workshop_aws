---
title : "CloudWatch Logs"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.12.1. </b> "
---

#### Overview

In this step, we will configure **Amazon CloudWatch Logs** for the **DocuMind AI** system.

CloudWatch Logs enables the backend and worker to record critical events during operation, such as document uploads, SQS message queuing, Textract OCR processing, Gemini/OpenAI AI Analysis execution, worker errors, IAM permission errors, and security events.

Centralizing logs helps developers and administrators diagnose issues, trace the document processing pipeline, and evaluate system health.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create a CloudWatch Log Group for DocuMind AI.
* Configure the backend and worker to write logs to CloudWatch.
* Categorize logs by processing module.
* Verify log records for uploads, SQS, OCR, and AI Analysis.
* Lay the foundation for monitoring metrics and alarms.
* Prevent logging of sensitive information.

---

#### The Role of CloudWatch Logs in DocuMind AI

In the **DocuMind AI** system, CloudWatch Logs is used to track:

* Backend API initialization and shutdown.
* Document Upload API execution.
* S3 upload results.
* SQS message sending and receiving.
* OCR Worker operations.
* Amazon Textract OCR completion statuses.
* AI Gateway provider selection decisions.
* Gemini/OpenAI API response statuses.
* Notification creation events.
* Audit Log creation events.
* Security events and unauthorized access attempts.
* System exceptions thrown during document processing.

---

#### CloudWatch Logs Architecture

Logging pipeline:

```text id="9y4qoc"
Backend API / Worker
  |
  v
Logger Service
  |
  v
Amazon CloudWatch Logs
  |
  v
Log Group: /docmind/application
```

![cloudwatch-logs](/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.1-cloudwatch-logs/cloudwatch-logs.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the CloudWatch Log Group `/docmind/application` containing logs written by the backend or worker.
> Save the image at the path:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.1-cloudwatch-logs/cloudwatch-logs.png`

---

#### Proposed Log Group

Application log group:

```text id="v9x1po"
/docmind/application
```

You can optionally split this into separate log groups:

```text id="z7ba8h"
/docmind/backend
/docmind/worker
/docmind/security
```

For this workshop, we will use a single consolidated log group:

```text id="x3ft4h"
/docmind/application
```

---

#### Step 1: Open the CloudWatch Console

Log in to the AWS Console, search for:

```text id="f2kzdt"
CloudWatch
```

Select:

```text id="nl5rpd"
Logs
```

Then click:

```text id="gs14ve"
Log groups
```

---

#### Step 2: Create Log Group

Click:

```text id="x3bmiq"
Create log group
```

Input the log group name:

```text id="dglyo3"
/docmind/application
```

Select a retention setting.

For development/workshop use:

```text id="vluk2o"
Retention: 7 days
```

For production use:

```text id="hf3xti"
Retention: 30 days or 90 days
```

Then click:

```text id="khamn3"
Create
```

---

#### Step 3: Configure Environment Variables

Add to the backend `.env` file:

```env id="ib3x4r"
AWS_REGION="ap-southeast-1"
CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-dev"
```

If configuring a separate stream for the worker:

```env id="p82d8i"
CLOUDWATCH_WORKER_LOG_STREAM="worker-dev"
```

---

#### Step 4: IAM Permissions Required

The backend/worker requires:

```text id="foa1tp"
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

These permissions were prepared in step **5.11 - IAM Role and Policy**.

---

#### Step 5: Log Backend Events

The backend should output log entries for key execution points:

```text id="4m5vz2"
[APP_STARTED]
[DATABASE_CONNECTED]
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[DOCUMENT_STATUS_REQUESTED]
[USER_NOTIFICATION_CREATED]
[AUDIT_LOG_CREATED]
```

Example JSON log:

```json id="mm4btk"
{
  "level": "info",
  "event": "DOCUMENT_UPLOAD_STARTED",
  "message": "User started uploading document.",
  "userId": "user_001",
  "timestamp": "2026-06-10T10:00:00.000Z"
}
```

---

#### Step 6: Log Worker Events

The worker should log:

```text id="xvm1il"
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_STATUS_COMPLETED]
[SQS_MESSAGE_DELETED]
```

On execution errors:

```text id="fojqul"
[DOCUMENT_PROCESSING_FAILED]
[TEXTRACT_OCR_FAILED]
[AI_PROVIDER_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[SQS_MESSAGE_RETRY_PENDING]
```

---

#### Step 7: Do Not Log Sensitive Data

Ensure these parameters are never written to logs:

```text id="0es8lv"
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
DATABASE_URL containing password
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
full document contents
JWT tokens
Passwords
```

Only record relevant metadata keys:

```text id="fpk9zp"
documentId
userId
fileName
status
provider
model
errorCode
```

---

#### Step 8: Verify Logs in CloudWatch

After launching the backend/worker, navigate to:

```text id="zjshpf"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Open the target log stream:

```text id="yv7gdl"
backend-dev
worker-dev
```

Confirm logs are updating.

---

#### Step 9: Verify Logs by Uploading a Document

Manual test flow:

1. Launch backend services.
2. Launch worker processes.
3. Upload a sample PDF via frontend UI or Postman.
4. Verify backend log shows `DOCUMENT_UPLOAD_STARTED`.
5. Verify S3 upload log shows `S3_UPLOAD_SUCCESS`.
6. Verify SQS log shows `SQS_SEND_MESSAGE_SUCCESS`.
7. Verify worker logs show `TEXTRACT_OCR_COMPLETED`.
8. Verify worker logs show `AI_ANALYSIS_COMPLETED`.

---

#### Troubleshoot Scenarios

| Error                   | Cause                                    | Solution                                  |
| ----------------------- | ---------------------------------------- | ----------------------------------------- |
| Log group missing       | Group not created or region mismatch     | Verify AWS region settings                |
| Log stream missing      | App failed to create log stream          | Check logger client initialization       |
| `AccessDeniedException` | Missing IAM log permissions              | Check IAM policy for log actions          |
| Logs do not update      | App failed to post events                | Verify logger configuration               |
| Secrets in logs         | Log contains raw config payloads         | Mask secret values before outputting logs |
| Incorrect timestamp     | Server timezone misconfiguration         | Write logs using ISO UTC timestamps       |

---

#### Completion Checklist

You have completed this step when:

* The CloudWatch Log Group is created.
* Backend writes logs to CloudWatch.
* Worker writes logs to CloudWatch.
* Log entries appear for uploads, SQS, OCR, and AI Analysis.
* Error details are logged clearly.
* No API keys or credentials are written to logs.
* Administrators can use logs to debug the processing pipeline.

---

#### Expected Outcome

Following this step, DocuMind AI has a centralized logging setup using CloudWatch Logs. The backend and worker write essential execution events, facilitating debugging, monitoring, and operations management.
