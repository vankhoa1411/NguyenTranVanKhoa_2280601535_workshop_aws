---
title : "Audit Log"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.10.3. </b> "
---

#### Overview

In this step, we will build the **Audit Log** feature for the **DocuMind AI** system.

The Audit Log tracks critical actions performed by users, Administrators, and system services. This is a vital component in document processing systems as it enables activity tracing, error diagnostics, security auditing, and compliance verification.

In DocuMind AI, the Audit Log records events such as logins, document uploads, OCR extractions, AI Analysis tasks, user role modifications, document deletions, worker errors, and administrator operations.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create the `AuditLog` database model in PostgreSQL.
* Implement an audit logging helper service.
* Log critical user, administrator, and worker actions.
* Display the Audit Log within the Admin Panel.
* Implement filters by user, action, timestamp, and severity level.
* Facilitate debugging of document processing errors.
* Build the foundation for security monitoring and compliance.

---

#### The Role of Audit Logs in DocuMind AI

The Audit Log answers essential questions:

* Who uploaded a document?
* Which documents were processed?
* When did OCR extraction complete?
* Which AI provider was used for analysis?
* Who viewed or deleted a document?
* What configuration changes did an Admin perform?
* At which stage did a document processing error occur?
* Have there been abnormal access attempts?

---

#### Audit Log Flow

Audit logging pipeline:

```text id="i6rbpd"
User / Admin / Worker Action
  |
  v
Audit Log Service
  |
  v
PostgreSQL + Prisma
  |
  v
Admin Audit Log Page
```

![audit-log](/images/5-Workshop/5.10-Notification-and-Admin/5.10.3-audit-log/audit-log.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the Audit Log table in the Admin Panel or DBeaver/Prisma Studio showing events such as login, upload document, OCR completed, and AI analysis completed.
> Save the image at the path:
> `/static/images/5-Workshop/5.10-Notification-and-Admin/5.10.3-audit-log/audit-log.png`

---

#### AuditLog Model

Example Prisma model:

```prisma id="e345un"
model AuditLog {
  id          String   @id @default(uuid())
  userId      String?
  action      String
  target      String?
  description String?
  ipAddress   String?
  userAgent   String?
  metadata    Json?
  severity    String   @default("info")
  createdAt   DateTime @default(now())
 
  user User? @relation(fields: [userId], references: [id], onDelete: SetNull)
}
```

Fields description:

| Field         | Description                               |
| ------------- | ----------------------------------------- |
| `userId`      | User who triggered the action             |
| `action`      | Action type name                          |
| `target`      | Impacted resource category                |
| `description` | Summary description of the event          |
| `ipAddress`   | IP address of the source request          |
| `userAgent`   | Client user agent signature               |
| `metadata`    | Optional JSON payload                     |
| `severity`    | Event level classification                |
| `createdAt`   | Log creation timestamp                    |

---

#### Audited Actions

Recommended system actions to log:

```text id="8p00i2"
USER_LOGIN
USER_LOGOUT
USER_REGISTER
DOCUMENT_UPLOADED
DOCUMENT_VIEWED
DOCUMENT_DELETED
DOCUMENT_PROCESSING_STARTED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
AI_PROVIDER_FALLBACK_USED
ADMIN_VIEWED_USER
ADMIN_CHANGED_ROLE
ADMIN_BROADCAST_NOTIFICATION
SECURITY_FORBIDDEN_ACCESS
```

---

#### Audit Log Service

Create the file:

```text id="6pdf87"
src/services/audit-log.service.ts
```

Example service:

```ts id="ys4z6q"
import { prisma } from "../prisma/client";
 
export async function createAuditLog(params: {
  userId?: string;
  action: string;
  target?: string;
  description?: string;
  ipAddress?: string;
  userAgent?: string;
  metadata?: any;
  severity?: string;
}) {
  return prisma.auditLog.create({
    data: {
      userId: params.userId,
      action: params.action,
      target: params.target,
      description: params.description,
      ipAddress: params.ipAddress,
      userAgent: params.userAgent,
      metadata: params.metadata || {},
      severity: params.severity || "info"
    }
  });
}
```

---

#### Log User Login

In the login controller:

```ts id="nl9rej"
await createAuditLog({
  userId: user.id,
  action: "USER_LOGIN",
  target: "Auth",
  description: "User logged in successfully.",
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
  severity: "info"
});
```

---

#### Log Document Upload

Upon successfully uploading a document:

```ts id="n5d4u4"
await createAuditLog({
  userId,
  action: "DOCUMENT_UPLOADED",
  target: "Document",
  description: "User uploaded a document.",
  metadata: {
    documentId: document.id,
    fileName: document.fileName,
    s3Key: document.s3Key
  },
  severity: "info"
});
```

---

#### Log OCR Completion

In the background worker:

```ts id="gzenak"
await createAuditLog({
  userId,
  action: "OCR_COMPLETED",
  target: "Document",
  description: "Amazon Textract OCR completed.",
  metadata: {
    documentId,
    provider: "AWS_TEXTRACT"
  },
  severity: "info"
});
```

---

#### Log AI Analysis Completion

```ts id="txlxxn"
await createAuditLog({
  userId,
  action: "AI_ANALYSIS_COMPLETED",
  target: "Document",
  description: "AI analysis completed successfully.",
  metadata: {
    documentId,
    provider: aiResult.provider,
    model: aiResult.model
  },
  severity: "info"
});
```

---

#### Log Security Failures

If a user tries to access a document they do not own:

```ts id="4xwh2z"
await createAuditLog({
  userId: user.id,
  action: "SECURITY_FORBIDDEN_ACCESS",
  target: "Document",
  description: "User attempted to access a document without permission.",
  metadata: {
    documentId
  },
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
  severity: "warning"
});
```

---

#### Admin Audit Log APIs

Endpoints exposed to administrators:

| API                                | Purpose                              |
| ---------------------------------- | ------------------------------------ |
| `GET /api/admin/audit-logs`        | Fetch audit logs list                |
| `GET /api/admin/audit-logs/:id`    | Fetch details of a specific audit log|
| `GET /api/admin/audit-logs/export` | Export audit logs                    |
| `DELETE /api/admin/audit-logs/:id` | Delete a log entry (if authorized)   |

All administrative endpoints must invoke:

```text id="zfwzqp"
requireAuth
requireAdmin
```

---

#### Audit Log Query Filters

Administrators should be able to filter logs by:

```text id="o8xrt1"
userId
action
severity
dateFrom
dateTo
target
```

Example request:

```text id="j1uz3z"
GET /api/admin/audit-logs?action=DOCUMENT_UPLOADED&severity=info
```

---

#### Audit Log Response Format

Example payload:

```json id="z1ahmd"
{
  "success": true,
  "data": [
    {
      "id": "audit_001",
      "userId": "user_001",
      "action": "DOCUMENT_UPLOADED",
      "target": "Document",
      "description": "User uploaded a document.",
      "metadata": {
        "documentId": "doc_001",
        "fileName": "invoice-demo.pdf"
      },
      "severity": "info",
      "createdAt": "2026-06-10T10:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

---

#### Admin Panel Audit Log UI Layout

The Admin Panel should render audit logs in a tabular layout:

| Time       | User            | Action            | Target   | Severity | Detail |
| ---------- | --------------- | ----------------- | -------- | -------- | ------ |
| 2026-06-10 | demo@docmind.ai | DOCUMENT_UPLOADED | Document | info     | View   |

Required features:

* Text search.
* Filter by action types.
* Filter by severity levels.
* Date-range selectors.
* Modal overlay to inspect JSON metadata payloads.
* Data export options (CSV/JSON).

---

#### Audit Log Security Safeguards

When writing audit logs, follow these constraints:

* Do not log user passwords.
* Do not log JWT tokens.
* Do not log Gemini/OpenAI API keys.
* Do not log AWS secret credentials.
* Do not log the complete text content of documents if sensitive.
* Only store essential metadata parameters.
* Restrict read access to Administrators only.

---

#### Verification Steps

Manual verification steps:

1. Log in with a standard user.
2. Confirm the `USER_LOGIN` log is created.
3. Upload a sample document.
4. Confirm the `DOCUMENT_UPLOADED` log is created.
5. Start the background worker process.
6. Verify the `OCR_COMPLETED` log.
7. Verify the `AI_ANALYSIS_COMPLETED` log.
8. Attempt an unauthorized document view.
9. Verify the `SECURITY_FORBIDDEN_ACCESS` log.
10. Log in with an Admin account and open the Audit Log page to verify formatting.

---

#### Troubleshoot Scenarios

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| Logs are missing                     | Audit service not called                 | Check controller and worker logic         |
| Logs lack a `userId`                 | Request context missing user details     | Verify auth middleware configuration      |
| Admin cannot view log lists          | Missing endpoint or wrong permissions    | Validate `requireAdmin` helper logic      |
| Metadata payloads are too large      | Log contains full body payloads          | Store only key reference identifiers      |
| Sensitive values exposed             | Plaintext keys or credentials logged     | Mask or clean the logged attributes       |
| Slow query speeds on log queries     | Large log tables                         | Implement pagination, filters, and indexes|

---

#### Completion Checklist

You have completed this step when:

* The `AuditLog` model is defined in the Prisma Schema.
* The audit logging service helper is operational.
* Logins and logouts generate audit logs.
* Document uploads generate audit logs.
* OCR and AI Analysis events generate audit logs.
* Document processing failures are audited.
* Unauthorized access attempts are audited.
* The Admin Panel successfully renders the audit logs.
* The log list supports filtering and pagination.
* No credentials or sensitive data are saved in logs.

---

#### Expected Outcome

Following this step, DocuMind AI has a fully configured Audit Log system to trace user, administrator, and worker actions. This builds system transparency, aids debugging, improves security, and provides compliance tools for the document processing system.
