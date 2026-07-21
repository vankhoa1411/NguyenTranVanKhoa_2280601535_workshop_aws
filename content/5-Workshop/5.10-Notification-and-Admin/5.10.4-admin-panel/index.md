---
title : "Admin Panel"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.10.4. </b> "
---

#### Overview

In this step, we will build the **Admin Panel** for the **DocuMind AI** system.

The Admin Panel is the interface for administrators to monitor system activity, manage users, review uploaded documents, inspect processing errors, track notifications, view audit logs, and monitor critical components such as SQS workers, AI Providers, CloudWatch Logs, or WAF (if integrated).

The Admin Panel makes the system easy to operate, especially when many users are uploading documents and multiple processing jobs are running asynchronously.

---

#### Objectives of This Step

Upon completing this step, you will:

* Build the Admin Panel user interface.
* Restrict access to users with the `ADMIN` role only.
* Display system overview statistics.
* Render the list of users.
* Render the list of documents.
* Render Admin notifications.
* Render system audit logs.
* Track document processing failures.
* Prepare dashboards for monitoring and security.

---

#### The Role of the Admin Panel in DocuMind AI

The Admin Panel helps administrators to:

* View the total user count.
* View the total number of uploaded documents.
* Monitor documents currently processing.
* Monitor completed document processing tasks.
* Track document processing errors.
* Monitor active AI providers.
* Inspect OCR, AI, and worker exceptions.
* View Admin notifications.
* Inspect audit logs.
* Manage user roles if necessary.
* Track security-related events.

---

#### Admin Panel Architecture

Data flow:

```text id="zlifx7"
Admin User
  |
  v
Admin Panel Frontend
  |
  v
Admin API
  |
  v
PostgreSQL + Prisma
  |
  v
System Events / Notifications / Audit Logs
```

![admin-panel](/static/images/5-Workshop/5.10-Notification-and-Admin/5.10.4-admin-panel/admin-panel.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the Admin Panel dashboard showing overview cards (Total Users, Total Documents, Processing, Completed, Failed) along with Admin Notifications and Audit Logs sections.
> Save the image at the path:
> `/static/images/5-Workshop/5.10-Notification-and-Admin/5.10.4-admin-panel/admin-panel.png`

---

#### Admin Panel Screens

The Admin Panel layout consists of:

| Screen            | Purpose                                           |
| ----------------- | ------------------------------------------------- |
| Overview          | General system dashboard and stats                |
| Users             | View and manage user accounts                     |
| Documents         | Monitor all uploaded documents                    |
| Processing Errors | Inspect failures in OCR, AI, or worker execution |
| Notifications     | View Admin-specific alerts                        |
| Audit Logs        | Trace system-wide historical actions              |
| Security          | Inspect security alarms                           |
| Monitoring        | View log status and system health (if configured) |

---

#### Admin Overview Cards

The Overview dashboard should render the following cards:

```text id="ps8xdg"
Total Users
Total Documents
Processing Documents
Completed Documents
Failed Documents
Unread Admin Alerts
AI Provider Status
Queue Health
```

Example API response:

```json id="im6ne8"
{
  "success": true,
  "data": {
    "totalUsers": 25,
    "totalDocuments": 120,
    "processingDocuments": 5,
    "completedDocuments": 100,
    "failedDocuments": 3,
    "unreadAdminAlerts": 4,
    "defaultAIProvider": "openai"
  }
}
```

---

#### Recommended Admin APIs

Core administrative endpoints:

| API                                | Purpose                                   |
| ---------------------------------- | ----------------------------------------- |
| `GET /api/admin/overview`          | Fetch overall summary statistics          |
| `GET /api/admin/users`             | Fetch users list                          |
| `GET /api/admin/documents`         | Fetch all uploaded documents              |
| `GET /api/admin/documents/:id`     | Fetch details of a specific document      |
| `GET /api/admin/processing-errors` | Fetch document processing errors          |
| `GET /api/admin/notifications`     | Fetch Admin notifications list            |
| `GET /api/admin/audit-logs`        | Fetch system audit logs                   |
| `PUT /api/admin/users/:id/role`    | Update a user's role (USER/ADMIN)         |

All admin endpoints must enforce:

```text id="o41eo1"
requireAuth
requireAdmin
```

---

#### Admin Verification Middleware

The backend must validate user roles:

```ts id="k6z9yv"
export function requireAdmin(req: any, res: any, next: any) {
  if (!req.user || req.user.role !== "ADMIN") {
    return res.status(403).json({
      success: false,
      message: "Admin permission required."
    });
  }
 
  next();
}
```

The frontend routing must guard admin paths:

```text id="817nzt"
If user.role !== ADMIN:
  Redirect to Dashboard or render a 403 Forbidden screen
```

---

#### User Management

The Users management table should render:

| Column     | Description                                  |
| ---------- | -------------------------------------------- |
| Name       | User's full name                             |
| Email      | Registered email address                     |
| Role       | USER / ADMIN                                 |
| Status     | Active / Inactive                            |
| Created At | Registration date                            |
| Actions    | View Details, Toggle Role                    |

Toggling user roles must generate an Audit Log entry:

```text id="5uy0ni"
ADMIN_CHANGED_ROLE
```

---

#### Document Management

The Documents monitoring table should render:

| Column      | Description                               |
| ----------- | ----------------------------------------- |
| File Name   | Uploaded filename                         |
| Owner       | User email of the owner                   |
| Status      | Processing status                         |
| Provider    | AI provider used (OpenAI/Gemini)          |
| Uploaded At | Upload timestamp                          |
| Actions     | View Detail                               |

Admin filtering capabilities should include:

```text id="fxedvj"
status
user
provider
date
failed only
```

---

#### Processing Errors

The Processing Errors screen lists documents with processing failures:

```text id="ngvsxt"
FAILED documents
Textract errors
AI provider errors
SQS/DLQ errors
Permission errors
```

Example item:

```json id="qjta42"
{
  "documentId": "doc_001",
  "fileName": "contract-demo.pdf",
  "status": "FAILED",
  "error": "UnsupportedDocumentException",
  "createdAt": "2026-06-10T10:00:00.000Z"
}
```

---

#### Admin Notifications Panel

The notifications log must highlight alerts using severity badges:

```text id="7lk2h7"
info
warning
error
critical
```

Critical alerts should visually stand out so admins can act quickly.

Examples:

```text id="j5riay"
Critical: Message detected in Dead Letter Queue.
Warning: AI provider fallback used.
Error: Document processing failed.
```

---

#### Audit Logs Panel

The Audit Logs table should support:

* Text search.
* Filter by action types.
* Filter by users.
* Filter by severity levels.
* Date-range filtering.
* Pagination.
* Inspecting JSON metadata.
* Export options.

Audited actions:

```text id="iy33sp"
USER_LOGIN
DOCUMENT_UPLOADED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
SECURITY_FORBIDDEN_ACCESS
ADMIN_CHANGED_ROLE
```

---

#### Security and Monitoring (Optional)

If integrating CloudWatch and WAF, the Admin Panel can display:

* CloudWatch log summaries.
* WAF blocked/allowed request statistics.
* Recent security alert lists.
* Real-time status of AI providers.
* SQS queue approximate message counts.
* DLQ message count indicators.
* Secrets Manager configuration status.

If not implemented, display the placeholder:

```text id="xq6f8o"
Monitoring integration will be configured in the next workshop section.
```

---

#### Recommended Layout Design

The Admin Panel layout should follow:

```text id="jv7ccx"
Sidebar Navigation
  - Overview
  - Users
  - Documents
  - Notifications
  - Audit Logs
  - Security
  - Monitoring
 
Main Content View
  - Statistic summary cards
  - Data tables
  - Filter components
  - Detail drawer/modal overlays
```

The UI should follow the DocuMind AI design language:

* Glassmorphism layout cards.
* Clean blue/sky theme.
* Color-coded severity tags.
* Legible tables.
* Intuitive filtering options.
* Responsive compatibility.

---

#### Verification Steps

Manual verification steps:

1. Log in with a standard user account.
2. Attempt to navigate to `/admin`.
3. Verify that a `403 Forbidden` error page or redirect blocks access.
4. Log in with an Admin account.
5. Access the Admin Panel successfully.
6. Verify the Overview stats cards display correct numbers.
7. Open the Users page.
8. Open the Documents page.
9. Trigger a document failure and check the Processing Errors log.
10. Check if Admin Notifications update.
11. Check if Audit Logs log the logins and document activities.
12. Verify pagination and sorting filters.

---

#### Troubleshoot Scenarios

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| Non-admin users access Admin Panel   | Guard checks or middleware are missing   | Verify client router and backend auth rules|
| Admins are incorrectly blocked       | User role is not set to `ADMIN` in DB    | Check user record role value              |
| Stats count is incorrect             | Incorrect SQL aggregation query          | Check Prisma aggregate/count queries      |
| Document owner name is missing       | Query did not include user association   | Add `include: { user: true }` in Prisma   |
| Audit log table is empty             | Audit log service was not called         | Check backend controller event listeners  |
| Notification filtering fails         | API endpoint misses query parameter mapping| Verify query parameters parsing in controller|
| Slow table page load speeds          | Large database data without pagination   | Implement server-side pagination          |

---

#### Completion Checklist

You have completed this step when:

* The Admin Panel restricts access to `ADMIN` users only.
* The Overview screen displays accurate statistics.
* Administrators can inspect the users list.
* Administrators can inspect all uploaded documents.
* Administrators can view document processing errors.
* Administrators can view Admin notifications.
* Administrators can view system audit logs.
* Filters and paginations are operational.
* Administrator operations generate audit logs.
* Non-admin users are blocked from admin resources.

---

#### Expected Outcome

Following this step, DocuMind AI has a fully functional Admin Panel. Administrators can monitor users, documents, processing errors, alerts, and audit logs, making it easier to control and monitor the platform.
