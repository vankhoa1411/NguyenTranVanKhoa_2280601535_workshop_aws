---
title : "Monitoring and Security"
date : 2026-06-10
weight : 12
chapter : false
pre : " <b> 5.12. </b> "
---

#### Build Monitoring and Security

In this section, we will configure the **Monitoring** and **Security** components for the **DocuMind AI** system.

Once the system has a complete document processing pipeline—including Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, PostgreSQL + Prisma, Backend API, Worker, Notifications, and the Admin Panel—the next logical step is to introduce monitoring and security protections.

Monitoring enables developers and Administrators to track the operations of the backend, worker, SQS queues, AI providers, and document processing errors. Security protects APIs, manages secrets safely, checks unauthorized access attempts, and mitigates risks from malicious payloads.

---

#### The Role of Monitoring in DocuMind AI

In the **DocuMind AI** system, Monitoring is used to:

* Record backend application logs.
* Record worker application logs.
* Validate document uploads.
* Monitor message queues in Amazon SQS.
* Monitor OCR extraction in Amazon Textract.
* Monitor AI Analysis tasks via Gemini/OpenAI.
* Record document processing failures.
* Track AI provider fallbacks.
* Assist administrators in diagnostic operations.
* Populate the Admin Monitoring Dashboard.

Critical log events can be recorded in **Amazon CloudWatch Logs** to enable easy searching and inspection when issues occur.

---

#### The Role of Security in DocuMind AI

Security features protect system data and restrict resource access.

In DocuMind AI, Security is focused on:

* Eliminating hardcoded API keys in source files.
* Storing configurations in AWS Secrets Manager.
* Maintaining S3 buckets in a non-public configuration.
* Restricting document viewing to the document owner.
* Enforcing role checks on Admin endpoints (`role = ADMIN`).
* Auditing critical operations in the Audit Log.
* Recording unauthorized access attempts.
* Deploying AWS WAF to mitigate SQL Injection, XSS, and bot risks.
* Obfuscating user passwords, JWT tokens, API keys, and sensitive file content in logs.

---

#### Monitoring and Security Architecture

System workflow:

```text id="kotk24"
Frontend / User Request
  |
  v
AWS WAF
  |
  v
Backend API / Worker
  |
  |-- Write logs --> CloudWatch Logs
  |-- Read secrets --> AWS Secrets Manager
  |-- Create Audit Log --> PostgreSQL + Prisma
  |-- Create Admin Notification --> PostgreSQL + Prisma
  |
  v
Admin Panel / Monitoring Dashboard
```

![monitoring-security-flow](/static/images/5-Workshop/5.12-Monitoring-and-Security/monitoring-security-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating AWS WAF protecting requests entering the system, the Backend/Worker posting logs to CloudWatch, retrieving secrets from Secrets Manager, and saving Audit Logs/Admin Notifications in PostgreSQL via Prisma.
> Save the image at the path:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/monitoring-security-flow.png`

---

#### Detailed Content

In section **5.12 - Monitoring and Security**, we will perform the following steps:

* [CloudWatch Logs](5.12.1-cloudwatch-logs/)
* [Secrets Manager](5.12.2-secrets-manager/)
* [WAF Protection](5.12.3-waf-protection/)
* [Test Security Logs](5.12.4-test-security-logs/)

---

#### CloudWatch Logs

**Amazon CloudWatch Logs** stores application and worker events.

Proposed Log Group Name:

```text id="4967r7"
/docmind/application
```

Key audited events:

```text id="zek0rw"
[APP_STARTED]
[DATABASE_CONNECTED]
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_PROCESSING_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[SECURITY_FORBIDDEN_ACCESS]
```

CloudWatch allows developers and administrators to quickly diagnose errors when the document processing pipeline misbehaves.

---

#### Secrets Manager

**AWS Secrets Manager** stores sensitive credentials securely.

Stored secrets:

```text id="4t8fyv"
DATABASE_URL
JWT_SECRET
OPENAI_API_KEY
GEMINI_API_KEY
AI_PROVIDER
AWS_SQS_QUEUE_URL
```

Proposed secret key name:

```text id="74ey09"
docmind/backend
```

In production, the backend configures:

```env id="0enioh"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

Do not store plaintext credentials inside production environment files.

---

#### AWS WAF

**AWS WAF** protects web applications from malicious web requests.

WAF can be attached to:

```text id="bffnkx"
Amazon CloudFront
Application Load Balancer
API Gateway
```

In DocuMind AI, WAF mitigates:

```text id="k2yjp2"
SQL Injection
Cross-Site Scripting
Bad bot traffic
Known bad inputs
Suspicious request patterns
```

Recommended Managed Rule Groups:

```text id="jew7uq"
AWSManagedRulesCommonRuleSet
AWSManagedRulesKnownBadInputsRuleSet
AWSManagedRulesSQLiRuleSet
AmazonIpReputationList
```

During initial setup, configure rule actions to `Count` mode to check query parameters before shifting actions to `Block`.

---

#### Security Logs

Security logs capture critical authorization and access events:

```text id="wl8s4p"
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

These security events should be written to:

* CloudWatch Logs.
* PostgreSQL AuditLog tables.
* Admin Notifications (for critical errors).

Example unauthorized view event:

```text id="8q4oap"
[SECURITY_FORBIDDEN_ACCESS]
```

Audit Log database record:

```json id="ygyh6k"
{
  "action": "SECURITY_FORBIDDEN_ACCESS",
  "target": "Document",
  "severity": "warning",
  "metadata": {
    "documentId": "doc_001"
  }
}
```

---

#### Key Security Components

| Component          | Role                                           |
| ------------------ | ---------------------------------------------- |
| CloudWatch Logs    | Stores application, worker, and security logs  |
| Secrets Manager    | Stores credentials and API keys                |
| AWS WAF            | Filters and secures web request payloads       |
| AuditLog           | Tracks system events in PostgreSQL             |
| Admin Notification | Alerts administrators to operational issues    |
| IAM Role           | Securely authorizes backend/worker services    |
| Admin Panel        | Visualizes operational status and error reports|

---

#### Logging Guidelines

When configuring loggers, ensure these parameters are never written in plaintext:

```text id="fu1xam"
password
JWT token
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
DATABASE_URL containing password
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
full document content
```

Store only key reference metadata:

```text id="9vr2jw"
documentId
userId
fileName
status
provider
model
errorCode
requestId
```

Example secure log format:

```json id="jv6z99"
{
  "level": "info",
  "event": "AI_ANALYSIS_COMPLETED",
  "documentId": "doc_001",
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "timestamp": "2026-06-10T10:00:00.000Z"
}
```

---

#### Monitoring via Admin Panel

The Admin Panel can display:

```text id="g4r0bv"
Total Requests
Upload Success
Processing Failed
AI Provider Fallback Count
Unread Admin Alerts
WAF Counted Requests
WAF Blocked Requests
Recent Security Events
```

If real-time dashboard widgets are not built, administrators can monitor status via:

* CloudWatch Logs.
* Admin Notifications.
* Audit Logs.
* Processing Errors list.

---

#### End-to-End Verification Steps

Manual verification steps:

1. Launch backend services.
2. Launch worker processes.
3. Upload a valid document.
4. Verify upload logs appear in CloudWatch.
5. Verify worker queue polling logs.
6. Verify Textract OCR logs.
7. Verify AI Analysis execution logs.
8. Trigger a failed login attempt.
9. Attempt unauthorized document viewing.
10. Upload a malformed file type.
11. Trigger an AI fallback event.
12. Simulate SQL Injection/XSS payloads against backend endpoints.
13. Verify Audit Logs and Admin Notifications are generated.
14. Inspect log contents to confirm no credentials are leaked.

---

#### Troubleshoot Scenarios

| Error                             | Cause                                    | Solution                                  |
| --------------------------------- | ---------------------------------------- | ----------------------------------------- |
| Logs missing in CloudWatch        | Logger initialization failure or IAM error| Check CloudWatch logs IAM policy          |
| Secrets Manager retrieval fails   | Incorrect secret name or missing policy  | Verify secret path and IAM role           |
| WAF fails to block traffic        | Not attached to resource group           | Check associated resources in WAF console |
| Security actions are not logged   | Middleware missing logging hooks         | Verify auth/authorization middleware logic|
| Admin alerts fail to trigger      | Notification service was not called      | Verify notification trigger triggers      |
| Plaintext keys in logs            | Logging raw parameters or environment     | Clean and scrub logged payloads           |
| WAF blocks legitimate traffic     | Rule configurations are too strict       | Set rule action to Count mode             |

---

#### Expected Outcomes

Upon completing this section, the system should achieve:

* CloudWatch Log Group created.
* Backend/worker posting logs to CloudWatch.
* Secrets Manager storing system configurations.
* Backend retrieving keys dynamically from Secrets Manager in production.
* AWS WAF configured to inspect application traffic.
* WAF metrics enabled.
* Security access exceptions logged.
* PostgreSQL AuditLog updated with security events.
* Admin Alerts triggered on critical errors.
* Log files are free of plain text credentials.

---

#### Expected Outcome

Following this section, DocuMind AI has a complete security and monitoring layer. The system writes centralized logs, manages secrets safely, filters threats using AWS WAF, and provides administrators with operational and security logs.
