---
title : "User Notifications"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.10.1. </b> "
---

#### Overview

In this step, we will build the **User Notifications** feature for the **DocuMind AI** system.

User Notifications keep users updated on events related to their documents, such as successful uploads, documents queued for processing, OCR completions, AI analysis completions, or processing failures.

This feature eliminates the need for users to manually check their dashboard. Once document processing is completed, the frontend displays a notification, allowing users to view the results on the Dashboard.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create user notifications.
* Store notifications in PostgreSQL via Prisma.
* Render the list of notifications on the frontend.
* Allow users to mark notifications as read.
* Map notifications to specific document processing stages.
* Integrate notifications with file upload, OCR, and AI Analysis tasks.
* Lay the foundation for real-time alerts if required.

---

#### The Role of User Notifications in DocuMind AI

In the **DocuMind AI** system, User Notifications are used to alert users when:

* A document is uploaded successfully.
* A document is queued for processing.
* Amazon Textract OCR extraction completes.
* Gemini/OpenAI AI Analysis completes.
* Document processing fails.
* Results are ready for display on the Dashboard.
* Processing errors require the user to re-upload the document.

Example notifications:

```text id="c9fc9m"
Document uploaded successfully.
OCR processing completed.
AI analysis completed.
Document processing failed.
```

---

#### User Notifications Architecture

Notification flow:

```text id="7o115l"
Backend API / Worker
  |
  v
Create Notification
  |
  v
PostgreSQL + Prisma
  |
  v
Frontend Notification Center
  |
  v
User reads notification
```

---

#### Notification Model

In Prisma, notifications can be stored using the `Notification` model.

Example schema:

```prisma id="e6ep4x"
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

Fields description:

| Field        | Description                                  |
| ------------ | -------------------------------------------- |
| `userId`     | Target user recipient of the notification    |
| `roleTarget` | Target role recipient, e.g., `ADMIN`         |
| `type`       | Classification type of the notification      |
| `severity`   | Alert level: info, success, warning, error   |
| `title`      | Short summary heading                        |
| `message`    | Descriptive text body                        |
| `metadata`   | Optional JSON payload (e.g., `documentId`)   |
| `isRead`     | Indicates whether the user read the alert    |
| `readAt`     | Timestamp indicating when it was read        |

---

#### User Notification Types

Recommended notification types:

```text id="d6mwt1"
DOCUMENT_UPLOADED
DOCUMENT_QUEUED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
```

Example payload:

```json id="5ur88b"
{
  "userId": "user_001",
  "type": "AI_ANALYSIS_COMPLETED",
  "severity": "success",
  "title": "AI analysis completed",
  "message": "Your document invoice-demo.pdf has been analyzed successfully.",
  "metadata": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf"
  },
  "isRead": false
}
```

---

#### Notification Service

Create the file:

```text id="60p0jg"
src/services/notification.service.ts
```

Example implementation to create notifications:

```ts id="kpj8k1"
import { prisma } from "../prisma/client";
 
export async function createUserNotification(params: {
  userId: string;
  type: string;
  severity: string;
  title: string;
  message: string;
  metadata?: any;
}) {
  return prisma.notification.create({
    data: {
      userId: params.userId,
      type: params.type,
      severity: params.severity,
      title: params.title,
      message: params.message,
      metadata: params.metadata || {},
      isRead: false
    }
  });
}
```

---

#### Generate Notification on Upload

Upon successfully uploading a document, invoke the notification service:

```ts id="t198ll"
await createUserNotification({
  userId,
  type: "DOCUMENT_UPLOADED",
  severity: "success",
  title: "Document uploaded",
  message: `${file.originalname} has been uploaded successfully.`,
  metadata: {
    documentId: document.id,
    fileName: file.originalname
  }
});
```

---

#### Generate Notification on AI Analysis Completion

Inside the background worker, after updating the status to `COMPLETED`, trigger the notification:

```ts id="v87y7h"
await createUserNotification({
  userId,
  type: "AI_ANALYSIS_COMPLETED",
  severity: "success",
  title: "AI analysis completed",
  message: "Your document has been analyzed successfully.",
  metadata: {
    documentId,
    provider: aiResult.provider,
    model: aiResult.model
  }
});
```

---

#### Generate Notification on Processing Failure

If the worker fails to process a document:

```ts id="r8mxfw"
await createUserNotification({
  userId,
  type: "DOCUMENT_PROCESSING_FAILED",
  severity: "error",
  title: "Document processing failed",
  message: "Your document could not be processed. Please check the file format or try again.",
  metadata: {
    documentId,
    errorCode: "PROCESSING_FAILED"
  }
});
```

---

#### Recommended Notification APIs

Endpoints exposed to users:

| API                                   | Purpose                             |
| ------------------------------------- | ----------------------------------- |
| `GET /api/notifications`              | Fetch all user notifications        |
| `GET /api/notifications/unread-count` | Get the count of unread notifications|
| `PUT /api/notifications/:id/read`     | Mark a specific notification as read|
| `PUT /api/notifications/read-all`     | Mark all user notifications as read |
| `DELETE /api/notifications/:id`       | Delete a notification               |

---

#### Notification List Response

Example payload:

```json id="nsctf0"
{
  "success": true,
  "data": [
    {
      "id": "noti_001",
      "type": "AI_ANALYSIS_COMPLETED",
      "severity": "success",
      "title": "AI analysis completed",
      "message": "Your document invoice-demo.pdf has been analyzed successfully.",
      "isRead": false,
      "metadata": {
        "documentId": "doc_001"
      },
      "createdAt": "2026-06-10T10:00:00.000Z"
    }
  ]
}
```

---

#### Notification Center UI Layout

The frontend should incorporate:

* A bell notification icon.
* An unread count badge overlay.
* A dropdown list displaying notifications.
* Read vs. unread state styles.
* A button to mark individual items as read.
* A button to mark all alerts as read.
* Navigation links to the Document Details screen if a `documentId` exists in metadata.

Example layout:

```text id="ie2cih"
🔔 3
 
AI analysis completed
Your document invoice-demo.pdf has been analyzed successfully.
View document
```

---

#### Real-time Notifications (Optional)

If implementing real-time functionality, Socket.IO can be utilized.

Workflow:

```text id="mjdb1u"
Worker creates notification
  |
  v
Emit socket event to user room
  |
  v
Frontend receives notification
  |
  v
Show toast + update notification badge
```

Proposed client room:

```text id="1wvf58"
user:{userId}
```

Proposed socket event name:

```text id="hfy3v8"
notification:new
```

---

#### Verification Steps

Manual verification steps:

1. Log in with a standard user account.
2. Upload a sample document.
3. Check for a successful upload notification.
4. Run the worker process to trigger OCR extraction and AI Analysis.
5. Check if the AI Analysis completed notification appears.
6. Open the Notification Center.
7. Click the notification card to verify it navigates to Document Details.
8. Click "mark as read".
9. Verify that the unread badge count decreases.

---

#### Troubleshoot Scenarios

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| Notifications are not appearing      | Service not called by backend/worker     | Verify controller and worker execution logs|
| Unread badge count is incorrect      | Query did not filter by `isRead=false`   | Verify unread API query parameters        |
| Notifications do not open documents  | `documentId` missing in notification metadata| Check S3 upload metadata payload      |
| Users see other users' alerts        | Query filter missing the `userId` key    | Validate auth filters in database queries |
| Mark as read endpoint fails          | Incorrect notification ID or ownership error| Check notification ownership validation   |
| Toast alerts do not sync in real-time | Socket client failed to join user room   | Verify Socket.IO server room membership   |

---

#### Completion Checklist

You have completed this step when:

* Notifications are successfully saved to PostgreSQL.
* Users receive notifications on file upload success.
* Users receive notifications on OCR/AI completion.
* Users receive notifications on processing failures.
* The fetch notifications API is operational.
* The mark as read API is operational.
* The frontend Notification Center is functional.
* Unread badges display accurate counts.
* Users can only see their own notifications.

---

#### Expected Outcome

Following this step, DocuMind AI has a user notification system. Users are notified when their documents are queued, processed, or fail, removing the need for manual dashboard monitoring.
