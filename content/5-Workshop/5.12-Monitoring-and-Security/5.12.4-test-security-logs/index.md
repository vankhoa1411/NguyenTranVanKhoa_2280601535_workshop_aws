---
title : "Kiểm tra Security Logs"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.12.4. </b> "
---

#### Overview

In this step, we will verify the **Security Logs** for the **DocuMind AI** system.

After configuring CloudWatch Logs, Secrets Manager, and AWS WAF, it is critical to verify that security events are recorded accurately. These events include failed logins, unauthorized access attempts, unsupported file uploads, AI provider failures, SQL Injection/XSS payloads flagged by WAF, or worker processing errors.

Security Logs enable administrators to detect anomalies early and investigate system incidents.

---

#### Objectives of This Step

Upon completing this step, you will:

* Verify log outputs for failed logins.
* Verify log outputs for unauthorized access attempts.
* Verify log outputs for unsupported file type uploads.
* Verify log outputs for worker processing failures.
* Verify log outputs for AI provider fallback events.
* Verify WAF metrics and sampled request logs.
* Ensure log payloads do not leak sensitive credentials.
* Set up security monitoring parameters in the Admin Panel.

---

#### The Role of Security Logs

Security logs capture critical system events:

* Failed logins.
* Forbidden access attempts.
* Unauthorized API calls.
* Unsupported file formats.
* Suspicious input payloads.
* WAF blocked/counted requests.
* AI provider failures.
* Secrets Manager access exceptions.
* IAM access denied errors.
* Worker processing errors.

---

#### Security Logs Flow

Security event detection and logging flow:

```text id="b5owud"
Frontend / User Request
  |
  v
AWS WAF
  |
  v
Backend Security Middleware
  |
  v
CloudWatch Logs
  |
  v
AuditLog / Admin Notification
  |
  v
Admin Panel
```

![test-security-logs](/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.4-test-security-logs/test-security-logs.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the CloudWatch Logs or Admin Audit Log interface displaying events like `SECURITY_FORBIDDEN_ACCESS`, `LOGIN_FAILED`, `UNSUPPORTED_FILE_UPLOAD`, or WAF `CountedRequests`.
> Save the image at the path:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.4-test-security-logs/test-security-logs.png`

---

#### Audited Security Events

Recommended events to trace:

```text id="gyr3d3"
LOGIN_FAILED
UNAUTHORIZED_REQUEST
SECURITY_FORBIDDEN_ACCESS
UNSUPPORTED_FILE_UPLOAD
FILE_SIZE_EXCEEDED
AI_PROVIDER_FALLBACK_USED
DOCUMENT_PROCESSING_FAILED
IAM_ACCESS_DENIED
WAF_REQUEST_COUNTED
WAF_REQUEST_BLOCKED
```

---

#### Step 1: Verify Failed Login Logging

Post a login request with an incorrect password:

```text id="281kkh"
POST /api/auth/login
```

Request payload:

```json id="fzpc0i"
{
  "email": "demo@docmind.ai",
  "password": "wrong-password"
}
```

Expected response:

```json id="ohs337"
{
  "success": false,
  "message": "Invalid email or password."
}
```

Expected log message:

```text id="3b2wml"
[LOGIN_FAILED]
```

Audit Log database entry:

```text id="6c929e"
USER_LOGIN_FAILED
```

---

#### Step 2: Verify Unauthorized Request Logging

Query an authenticated API endpoint without providing a JWT token:

```text id="u91uu4"
GET /api/documents
```

Expected response:

```json id="o4lvyh"
{
  "success": false,
  "message": "Unauthorized."
}
```

Expected log message:

```text id="o68dqt"
[UNAUTHORIZED_REQUEST]
```

---

#### Step 3: Verify Forbidden Access Logging

Authenticate as User A, then attempt to retrieve a document owned by User B:

```text id="l6yjxh"
GET /api/documents/doc_of_user_b/status
```

Expected response:

```json id="y81s56"
{
  "success": false,
  "message": "You do not have permission to access this document."
}
```

Expected log message:

```text id="yngsmn"
[SECURITY_FORBIDDEN_ACCESS]
```

Audit Log database payload:

```json id="fov8yl"
{
  "action": "SECURITY_FORBIDDEN_ACCESS",
  "target": "Document",
  "severity": "warning",
  "metadata": {
    "documentId": "doc_of_user_b"
  }
}
```

---

#### Step 4: Verify Unsupported File Upload Logging

Attempt to upload an invalid file extension (e.g., `.exe`, `.docx`, `.sh`):

Expected response:

```json id="r5czc0"
{
  "success": false,
  "message": "Unsupported file type. Only PDF, PNG, JPG and JPEG are allowed."
}
```

Expected log message:

```text id="6k7re1"
[UNSUPPORTED_FILE_UPLOAD]
```

If multiple unsupported file attempts occur from the same user, trigger an Admin Alert:

```text id="ld4xsm"
Multiple unsupported file uploads detected.
```

---

#### Step 5: Verify Max File Size Logging

Attempt to upload a document exceeding the file size limit (e.g., > 10MB):

Expected response:

```json id="e4ndqk"
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

Expected log message:

```text id="0mt07c"
[FILE_SIZE_EXCEEDED]
```

---

#### Step 6: Verify AI Provider Fallback Logging

Simulate a failure in your primary AI provider by misconfiguring its API key:

```env id="ms47th"
AI_PROVIDER="openai"
OPENAI_API_KEY="wrong-key"
```

Trigger an AI Analysis task.

Expected log messages:

```text id="h3lofi"
[AI_PROVIDER_SELECTED] openai
[AI_PROVIDER_PRIMARY_FAILED] openai
[AI_PROVIDER_FALLBACK_USED] gemini
```

Expected Admin Notification:

```text id="xs72z7"
AI provider fallback used.
```

Expected Audit Log entry:

```text id="twuddi"
AI_PROVIDER_FALLBACK_USED
```

---

#### Step 7: Verify Worker Failure Logging

Post a document processing job containing an invalid S3 Key to SQS:

```json id="5ztv1e"
{
  "documentId": "doc_error_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/not-found.pdf",
  "action": "PROCESS_DOCUMENT"
}
```

Expected worker log outputs:

```text id="v4vvlo"
[DOCUMENT_PROCESSING_FAILED]
[INVALID_S3_OBJECT]
```

Expected database document status updates:

```text id="6uqqjw"
FAILED
```

Expected Admin Alert:

```text id="dfx46o"
Document processing failed.
```

---

#### Step 8: Verify WAF SQL Injection Filtering

Simulate a SQL Injection query parameters payload:

```text id="xlk3bp"
/api/auth/login?email=' OR '1'='1
```

* If WAF is in `Count` mode, the request completes but increments the `CountedRequests` metric.
* If WAF is in `Block` mode, access is blocked and increments the `BlockedRequests` metric.

Validate counters in CloudWatch:

```text id="sjsl6x"
CloudWatch
→ Metrics
→ AWS/WAFV2
```

---

#### Step 9: Verify WAF XSS Filtering

Simulate an XSS payload in a chat query:

```json id="qr96tp"
{
  "question": "<script>alert('xss')</script>"
}
```

Verify WAF sampled requests or metrics.

---

#### Step 10: Verify Logs in CloudWatch

Open the CloudWatch Console:

```text id="0sbuj2"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Verify the presence of log entries:

```text id="d07q5a"
LOGIN_FAILED
UNAUTHORIZED_REQUEST
SECURITY_FORBIDDEN_ACCESS
UNSUPPORTED_FILE_UPLOAD
AI_PROVIDER_FALLBACK_USED
DOCUMENT_PROCESSING_FAILED
```

---

#### Step 11: Inspect Audit Logs in the Admin Panel

The Admin Panel should show:

| Time       | Action                     | Severity | Target   | Detail |
| ---------- | -------------------------- | -------- | -------- | ------ |
| 2026-06-10 | SECURITY_FORBIDDEN_ACCESS  | warning  | Document | View   |
| 2026-06-10 | DOCUMENT_PROCESSING_FAILED | error    | Document | View   |
| 2026-06-10 | AI_PROVIDER_FALLBACK_USED  | warning  | AI       | View   |

---

#### Step 12: Verify No Sensitive Data Exposure

Verify that CloudWatch logs and PostgreSQL Audit Log tables exclude:

```text id="8icawt"
password
JWT token
OPENAI_API_KEY
GEMINI_API_KEY
DATABASE_URL full password
AWS credentials
full document content
```

If present, configure the application logger to mask or drop these properties.

---

#### Troubleshoot Scenarios

| Error                       | Cause                                   | Solution                                      |
| --------------------------- | --------------------------------------- | --------------------------------------------- |
| Security logs are empty     | Logger or audit service not called      | Check auth middleware configuration           |
| Forbidden access not audited| Log statement missing in routing guard  | Add audit log trigger in permission checks    |
| WAF metrics missing         | CloudWatch metrics visibility disabled  | Enable Web ACL visibility metrics             |
| WAF does not block traffic  | WAF is not Associated with ALB/CloudFront| Link resources in AWS WAF Console             |
| Secrets in log output       | Logger log toàn bộ request              | Filter or mask sensitive parameters           |
| Admin alerts are missing    | Alert not sent to notification service  | Verify notification trigger execution flow    |
| Worker failures not logged  | Logger missing inside catch blocks      | Add logging statement in worker catch blocks  |

---

#### Completion Checklist

You have completed this step when:

* Failed login attempts are logged.
* Unauthorized API calls are logged.
* Forbidden access attempts are saved to the Audit Log.
* Unsupported file extensions are logged.
* File uploads exceeding size limits are logged.
* AI provider fallbacks are logged and generate Admin notifications.
* Document processing failures are logged.
* SQLi/XSS requests register in WAF metrics or blocks.
* Logs are visible in the CloudWatch Log Group.
* Audit logs are visible in the Admin Panel.
* No credentials or sensitive data are saved in logs.

---

#### Expected Outcome

Following this step, DocuMind AI has security log verification. Administrators can monitor logins, access exceptions, payload security, WAF metrics, and background job status, establishing a secure operational footing.
