---
title : "PostgreSQL and Prisma Database"
date : 2026-06-10
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

#### Configure PostgreSQL and Prisma Database

In this section, we will configure the **PostgreSQL Database** and the **Prisma ORM** for the **DocuMind AI** system.

PostgreSQL is used to store all core application data such as users, documents, OCR results, AI analysis, chat history, notifications, and audit logs. Prisma ORM helps the Node.js/TypeScript backend work with PostgreSQL easily through schemas, migrations, and the Prisma Client.

In this workshop, the system uses **PostgreSQL + Prisma** without using Amazon RDS. The database can be run locally using a local PostgreSQL service, Docker, or a dedicated PostgreSQL server.

---

#### The Role of PostgreSQL in DocuMind AI

In the **DocuMind AI** project, PostgreSQL is used to store:

* User account information.
* User/Admin role permissions.
* Uploaded document metadata.
* File paths on Amazon S3 via `s3Bucket` and `s3Key`.
* Document processing status.
* OCR results from Amazon Textract.
* AI Analysis results from Gemini/OpenAI.
* AI Chat/RAG history.
* Notifications for Users/Admins.
* Audit Logs for critical actions.

---

#### Database Architecture

The database data flow in DocuMind AI:

```text
Backend API / Worker
  |
  v
Prisma ORM
  |
  v
PostgreSQL Database
```

![postgresql-prisma-flow](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/postgresql-prisma-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the Backend API and Worker using Prisma ORM to read/write data into the PostgreSQL Database.
> Save the image at the path:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/postgresql-prisma-flow.png`

---

#### Core Database Tables

The DocuMind AI database should consist of the following main tables:

| Table          | Purpose                                      |
| -------------- | -------------------------------------------- |
| `User`         | Stores user accounts and User/Admin roles    |
| `Document`     | Stores metadata of uploaded documents        |
| `OCRResult`    | Stores OCR text extracted from Amazon Textract |
| `AIAnalysis`   | Stores analysis results from Gemini/OpenAI   |
| `ChatHistory`  | Stores document Q&A history                  |
| `Notification` | Stores notifications for Users/Admins        |
| `AuditLog`     | Logs critical administrative/user actions    |

---

#### Document Processing Statuses

In the document processing pipeline, the `Document` table must track the processing status so the frontend can display progress to the user.

Available statuses:

```text
PENDING
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Definitions:

| Status          | Description                                  |
| --------------- | -------------------------------------------- |
| `PENDING`       | Document metadata has just been initialized  |
| `UPLOADED`      | The file has been uploaded to S3             |
| `QUEUED`        | The processing job has been sent to SQS      |
| `PROCESSING`    | The worker is currently processing the document|
| `OCR_COMPLETED` | Textract OCR is finished                     |
| `AI_ANALYZING`  | Gemini/OpenAI is analyzing the content       |
| `COMPLETED`     | The document has finished processing         |
| `FAILED`        | An error occurred during processing          |

---

#### Detailed Content

In section **5.7 - PostgreSQL Prisma Database**, we will perform the following steps:

* [Set Up PostgreSQL Database](5.7.1-setup-postgresql-database/)
* [Configure Prisma Schema](5.7.2-configure-prisma-schema/)
* [Run Prisma Migrations](5.7.3-run-prisma-migration/)
* [Test Database Connection](5.7.4-test-database-connection/)

---

#### Related Environment Variables

The backend requires the following environment variable:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

If using a local Docker PostgreSQL:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

If the database is hosted on a remote server, replace `localhost` with the server's IP address or domain name.

> ⚠️ **Security Warning:**
> Do not commit your actual `DATABASE_URL` to GitHub. Only commit the `.env.example` file. When deploying to production, store `DATABASE_URL` in AWS Secrets Manager.

---

#### Integration with Other Components

PostgreSQL is utilized throughout the pipeline:

```text
Upload Document
  |
  v
Save Document Metadata
  |
  v
SQS Worker updates status
  |
  v
Textract saves OCRResult
  |
  v
Gemini/OpenAI saves AIAnalysis
  |
  v
Dashboard displays result
```

This integration allows the frontend to display the document processing status, OCR text, AI analysis, chat history, and notifications.

---

#### Expected Outcomes

Upon completing this section, the system should achieve:

* The PostgreSQL Database is successfully created.
* The backend has a valid `DATABASE_URL` configuration.
* The Prisma Schema is configured.
* Core database models are defined.
* Prisma Migrations have successfully created the tables.
* The Prisma Client is generated.
* The backend can read and write data to PostgreSQL.
* The database is ready to support file uploads, OCR, AI Analysis, Notifications, and the Admin Panel.

---

#### Expected Outcome

Following this section, DocuMind AI has a stable database layer to manage the document processing lifecycle. PostgreSQL and Prisma enable structured, queryable, and scalable data storage for the Node.js/TypeScript backend.
