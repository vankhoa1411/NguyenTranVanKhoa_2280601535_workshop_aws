---
title : "Notification and Admin"
date : 2026-06-10
weight : 10
chapter : false
pre : " <b> 5.10. </b> "
---

#### Build Notification and Admin

In this section, we will build the **Notification** system and the **Admin Panel** for the **DocuMind AI** project.

Once the system has a complete document processing pipeline—from uploads, S3 storage, SQS messaging, Textract OCR, and Gemini/OpenAI analysis—users need to be notified when their documents are processed successfully or unsuccessfully. Concurrently, Administrators require a dedicated space to monitor system operations, inspect errors, review audit logs, and manage critical system events.

This section makes DocuMind AI complete in terms of operation, monitoring, and administration.

---

#### The Role of Notifications in DocuMind AI

In the **DocuMind AI** system, Notifications are used to alert users and Admins about key events.

For users, notifications provide updates on:

* Successful document uploads.
* Documents queued for processing.
* Textract OCR extraction completion.
* Gemini/OpenAI AI Analysis completion.
* Document processing failures.
* Prompting users to open their Dashboard to view results.

For Administrators, notifications help monitor:

* Document processing failures.
* Background worker errors when processing queue messages.
* AI provider errors or fallback occurrences.
* Messages redirected to the Dead Letter Queue (DLQ).
* Unusual user activity or security-sensitive events.
* System warnings that require administrative intervention.

---

#### The Role of the Admin Panel

The Admin Panel is the control center for administrators to monitor and manage the system.

Administrators can:

* View overall user statistics.
* View total document upload counts.
* Monitor documents in pending, processing, completed, or failed states.
* Read Admin-specific alerts and notifications.
* Review system-wide audit logs.
* Inspect OCR, worker, or AI provider exceptions.
* Track security events.
* Manage user profiles and roles.

The Admin Panel makes the system easy to monitor when many users and documents are being processed asynchronously.

---

#### Notification and Admin Architecture

General workflow:

```text id="nleq36"
Backend API / Worker
  |
  |-- Create User Notification
  |-- Create Admin Notification
  |-- Create Audit Log
  |
  v
PostgreSQL + Prisma
  |
  |-----------------------------|
  v                             v
User Notification Center       Admin Panel
                                |
                                v
                         Audit Log / System Alerts
```

![notification-admin-flow](/static/images/5-Workshop/5.10-Notification-and-Admin/notification-admin-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the Backend API and Worker generating User Notifications, Admin Notifications, and Audit Logs, storing them in PostgreSQL via Prisma, and displaying them on the User Notification Center and Admin Panel.
> Save the image at the path:
> `/static/images/5-Workshop/5.10-Notification-and-Admin/notification-admin-flow.png`

---

#### Notification Workflow

The notification flow in DocuMind AI:

1. The user uploads a document.
2. The backend uploads the file to Amazon S3.
3. The backend saves the metadata to PostgreSQL.
4. The backend generates a User Notification: upload successful.
5. The backend sends a message to Amazon SQS.
6. The worker retrieves the message and starts processing.
7. The worker invokes Amazon Textract OCR.
8. When OCR completes, the worker stores the OCR Result.
9. The worker calls Gemini/OpenAI to analyze the document.
10. When AI Analysis completes, the worker stores the AIAnalysis.
11. The worker updates the document status to `COMPLETED`.
12. The worker generates a User Notification: analysis completed.
13. If an error occurs, the worker creates a User Notification and an Admin Notification.
14. The Admin can inspect the error details in the Admin Panel.

---

#### Detailed Content

In section **5.10 - Notification and Admin**, we will perform the following steps:

* [User Notifications](5.10.1-user-notifications/)
* [Admin Notifications](5.10.2-admin-notifications/)
* [Audit Log](5.10.3-audit-log/)
* [Admin Panel](5.10.4-admin-panel/)

---

#### Related Database Tables

This section relies on the following key PostgreSQL tables:

| Table          | Purpose                                              |
| -------------- | ---------------------------------------------------- |
| `Notification` | Stores notifications for Users and Admins            |
| `AuditLog`     | Tracks critical actions and system events            |
| `User`         | Identifies notification recipients and roles         |
| `Document`     | Links notifications and audits to specific documents |
| `AIAnalysis`   | Provides provider/model details upon analysis finish |

---

#### Notification Model

The notification model is shared between Users and Admins.

Example:

```prisma id="zx48rq"
model Notification {
  id         String   @id @default(uuid())
  userId     String?
  roleTarget String?
  type       String
  severity   String
  title      String
  message    String
  metadata   Json?
  isRead     Boolean  @default(false)
  createdAt  DateTime @default(now())
  readAt     DateTime?
 
  user User? @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

Usage rules:

* If the notification is targeted at a specific user, specify `userId`.
* If targeted at Admins, set `roleTarget = "ADMIN"`.
* For system-wide notifications, you can set `roleTarget` or use a custom broadcast mechanism.

---

#### Audit Log Model

The Audit Log tracks critical system-wide operations.

Example:

```prisma id="x54zh5"
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

Key audited events:

```text id="aud5ew"
USER_LOGIN
USER_LOGOUT
DOCUMENT_UPLOADED
DOCUMENT_PROCESSING_STARTED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
AI_PROVIDER_FALLBACK_USED
SECURITY_FORBIDDEN_ACCESS
ADMIN_CHANGED_ROLE
```

---

#### Notification Types

User notifications:

```text id="nbu2cc"
DOCUMENT_UPLOADED
DOCUMENT_QUEUED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
```

Admin notifications:

```text id="xjmjdv"
DOCUMENT_PROCESSING_FAILED
TEXTRACT_OCR_FAILED
AI_PROVIDER_FAILED
AI_PROVIDER_FALLBACK
SQS_DLQ_MESSAGE_DETECTED
SECURITY_ALERT
SYSTEM_ERROR
```

---

#### Severity Levels

Both Notifications and Audit Logs categorize events by severity:

| Severity   | Description                        |
| ---------- | ---------------------------------- |
| `info`     | Normal operational updates         |
| `success`  | Task completed successfully        |
| `warning`  | Non-fatal issues requiring attention|
| `error`    | Actionable execution errors        |
| `critical` | Critical errors requiring immediate intervention |

Examples:

```text id="dh6qgh"
success: AI analysis completed
warning: AI provider fallback used
error: Document processing failed
critical: Message moved to Dead Letter Queue
```

---

#### Notification and Admin APIs

User APIs:

| API                                   | Purpose                      |
| ------------------------------------- | ---------------------------- |
| `GET /api/notifications`              | Fetch user notifications     |
| `GET /api/notifications/unread-count` | Get unread notification count|
| `PUT /api/notifications/:id/read`     | Mark a notification as read  |
| `PUT /api/notifications/read-all`     | Mark all notifications as read|
| `DELETE /api/notifications/:id`       | Delete a notification        |

Admin APIs:

| API                                       | Purpose                          |
| ----------------------------------------- | -------------------------------- |
| `GET /api/admin/overview`                 | Fetch summary dashboard statistics|
| `GET /api/admin/notifications`            | Fetch Admin notifications        |
| `GET /api/admin/audit-logs`               | Fetch system audit logs          |
| `GET /api/admin/users`                    | Fetch user accounts list         |
| `GET /api/admin/documents`                | Fetch all system documents       |
| `GET /api/admin/processing-errors`        | Fetch document processing errors |
| `POST /api/admin/notifications/broadcast` | Broadcast alerts to multiple users|

All admin endpoints must enforce:

```text id="lam3om"
requireAuth
requireAdmin
```

---

#### User Notification Center Interface

The frontend should incorporate:

* A bell notification icon.
* An unread count badge overlay.
* A dropdown list displaying notifications.
* Distinct read and unread visual indicators.
* A "mark as read" toggle.
* A "mark all as read" quick action.
* Navigation links to Document Details if the notification metadata includes a `documentId`.

Example:

```text id="hi3k4n"
AI analysis completed
Your document invoice-demo.pdf has been analyzed successfully.
View document
```

---

#### Admin Panel Interface

The Admin Panel should partition features into tabs:

```text id="in754a"
Overview
Users
Documents
Processing Errors
Notifications
Audit Logs
Security
Monitoring
```

The Overview tab displays:

```text id="v9ujzz"
Total Users
Total Documents
Processing Documents
Completed Documents
Failed Documents
Unread Admin Alerts
Default AI Provider
Queue Health
```

Admin Notifications show warnings filtered by severity:

```text id="l41vqr"
info
warning
error
critical
```

Audit Logs render a structured tracking table:

```text id="69lckz"
Time | User | Action | Target | Severity | Detail
```

---

#### Real-time Notifications (Optional)

If implementing real-time alerts, you can utilize Socket.IO.

Workflow:

```text id="9bmkid"
Backend/Worker creates notification
  |
  v
Emit socket event
  |
  v
Frontend receives notification
  |
  v
Show toast + update unread badge
```

Proposed client socket rooms:

```text id="in8pvm"
user:{userId}
role:ADMIN
```

Proposed socket events:

```text id="fvc9ox"
notification:new
admin:alert
```

If real-time synchronization is not implemented, the frontend can query the notification API periodically (polling) or refresh when users open the Notification Center.

---

#### Security Guidelines

Ensure the following constraints are maintained:

* Users must only access notifications addressed to them.
* Standard users must be blocked from accessing the Admin Panel.
* Admin endpoints must validate `ADMIN` role attributes.
* Do not log passwords, JWT tokens, API keys, or AWS secret keys.
* Do not expose Gemini/OpenAI API keys to the client.
* Store only minimal, non-sensitive metadata in Audit Logs.
* Ensure notifications do not display sensitive document contents.
* Log all critical administrative operations in the Audit Log.

---

#### Required Logging

The backend and worker should output logs:

```text id="u0ms65"
[USER_NOTIFICATION_CREATED]
[ADMIN_NOTIFICATION_CREATED]
[AUDIT_LOG_CREATED]
[ADMIN_PANEL_OVERVIEW_REQUESTED]
[ADMIN_NOTIFICATION_READ]
[SECURITY_FORBIDDEN_ACCESS_LOGGED]
```

On execution errors:

```text id="7id36w"
[NOTIFICATION_CREATE_FAILED]
[AUDIT_LOG_CREATE_FAILED]
[ADMIN_ACCESS_DENIED]
```

---

#### End-to-End Verification Steps

Manual verification steps:

1. Log in with a standard user account.
2. Upload a sample document.
3. Verify that a User Notification is created for the upload.
4. Start the background worker process.
5. Verify that a notification appears when AI Analysis finishes.
6. Trigger a document processing failure (e.g., upload a malformed file).
7. Verify that a User Notification highlights the processing error.
8. Log in with an Admin account.
9. Open the Admin Panel.
10. Check if an Admin Notification displays the processing failure alert.
11. Check if the Audit Log records login, upload, OCR, and AI Analysis events.
12. Attempt to access the Admin Panel with a standard user account.
13. Verify that a `403 Forbidden` response blocks access.

---

#### Troubleshoot Scenarios

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| User notifications are missing       | Notification service not invoked         | Verify backend/worker execution logs      |
| Users see other users' alerts        | Query filters missing the `userId` key   | Validate auth filters in database queries |
| Admin alerts are missing             | Query filters missing the `roleTarget` key| Check admin notification query endpoints  |
| Audit logs are empty                 | Audit service not invoked                | Verify controller/worker audit logs       |
| Standard users access Admin Panel    | Missing path guards or middleware checks | Verify client router and backend role validations|
| Unread badge count is incorrect      | Query did not filter by `isRead=false`   | Verify unread API query parameters        |
| Large database metadata storage      | Verbose metadata payloads                | Store only key reference identifiers      |
| Exposure of sensitive information    | Tokens or API keys logged in plaintext   | Clean and scrub logged payloads           |

---

#### Expected Outcomes

Upon completing this section, the system should achieve:

* Users receive notifications for uploads.
* Users receive notifications when OCR/AI completes.
* Users receive notifications on document processing failures.
* Admins receive alerts for system errors.
* The system logs all critical operational events.
* The Admin Panel renders stats, alerts, documents, and audit logs.
* Non-admin users are blocked from admin resources.
* Admins can inspect execution issues and monitor system health.
* Sensitive details are excluded from alerts and logs.

---

#### Expected Outcome

Following this section, DocuMind AI has a complete notification and administration layer. Users receive clear updates about their documents, and Administrators have the monitoring utilities to track errors, audits, and operational health, ensuring the document processing pipeline remains stable.
