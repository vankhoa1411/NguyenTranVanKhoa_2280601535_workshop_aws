---
title : "Test Database Connection"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.7.4. </b> "
---

#### Overview

In this step, we will verify the connection between the **DocuMind AI** backend, **Prisma ORM**, and the **PostgreSQL Database**.

After setting up PostgreSQL, configuring the Prisma Schema, and running the Prisma Migration, the backend needs to be validated to ensure it can successfully perform read and write operations. This is a crucial step before integrating the database with components such as document uploads, OCR Result storage, AI Analysis results, Notifications, and Audit Logs.

---

#### Objectives of This Step

Upon completing this step, you will:

* Verify the correctness of the `DATABASE_URL` connection string.
* Ensure the Prisma Client has been generated.
* Confirm that the backend can successfully connect to PostgreSQL.
* Test creating sample database records.
* Test querying data using Prisma.
* Confirm the database is ready for integration with the S3, SQS, Textract, and AI Analysis modules.

---

#### Connection Verification Flow

The database verification flow in DocuMind AI:

```text
Backend API
  |
  v
Prisma Client
  |
  v
PostgreSQL Database
  |
  v
Return test result
```

![test-database-connection](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.4-test-database-connection/test-database-connection.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the terminal or Postman screen showing a successful database health check API call, or open Prisma Studio to show the mock test data.
> Save the image at the path:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.4-test-database-connection/test-database-connection.png`

---

#### Prerequisites Before Testing

Before testing, make sure that:

* PostgreSQL is running.
* The `docmind` database is created.
* The `.env` file contains the correct `DATABASE_URL`.
* The Prisma Schema has been validated.
* The Prisma Migration has run successfully.
* The Prisma Client is generated.
* The backend is configured to load the `.env` file.

Verify `.env`:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

---

#### Step 1: Verify PostgreSQL is Running

If running PostgreSQL locally:

```bash
pg_isready
```

Expected output:

```text
accepting connections
```

If using Docker:

```bash
docker ps
```

Check the PostgreSQL container logs:

```bash
docker logs docmind-postgres
```

If the container runs normally, proceed to the next step.

---

#### Step 2: Verify Connection via psql

Execute the command:

```bash
psql "postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Upon a successful connection, you will access the PostgreSQL interactive shell.

List the database tables:

```sql
\dt
```

Expected tables:

```text
User
Document
OCRResult
AIAnalysis
ChatHistory
Notification
AuditLog
```

Actual table names might differ depending on how Prisma mapped the models.

---

#### Step 3: Verify Prisma Client

Generate the client:

```bash
npx prisma generate
```

Then launch Prisma Studio:

```bash
npx prisma studio
```

Prisma Studio typically opens at:

```text
http://localhost:5555
```

If tables are displayed correctly, Prisma is successfully connected to the database.

---

#### Step 4: Create Database Health Check API

Create a temporary health check endpoint in the backend:

```text
GET /api/health/database
```

Expected response:

```json
{
  "success": true,
  "message": "Database connection is healthy",
  "database": "PostgreSQL",
  "orm": "Prisma"
}
```

Example verification logic:

```ts
import { prisma } from "../prisma/client";
 
export async function checkDatabaseHealth() {
  await prisma.$queryRaw`SELECT 1`;
  return {
    success: true,
    message: "Database connection is healthy",
    database: "PostgreSQL",
    orm: "Prisma"
  };
}
```

---

#### Step 5: Verify Using Postman

Open Postman and trigger:

```text
GET http://localhost:3000/api/health/database
```

Expected response:

```json
{
  "success": true,
  "message": "Database connection is healthy",
  "database": "PostgreSQL",
  "orm": "Prisma"
}
```

If an error occurs, the backend should return a descriptive error response instead of crashing.

---

#### Step 6: Create a Test User

Create a test user using Prisma Studio or a seed script.

Example seed script:

```ts
await prisma.user.create({
  data: {
    email: "demo@docmind.ai",
    fullName: "Demo User",
    role: "USER"
  }
});
```

Verify that the user record appears in Prisma Studio.

If the schema requires a `passwordHash` field, provide a placeholder value:

```ts
passwordHash: "hashed_password_demo"
```

---

#### Step 7: Create a Test Document

Create a test document to verify the User → Document relationship.

Example:

```ts
await prisma.document.create({
  data: {
    userId: "user_id_here",
    fileName: "invoice-demo.pdf",
    fileType: "application/pdf",
    fileSize: 204800,
    s3Bucket: "documind-document-storage",
    s3Key: "uploads/user_001/doc_001/invoice-demo.pdf",
    status: "UPLOADED"
  }
});
```

Verify that the record is created in the `Document` table.

---

#### Step 8: Test Querying Relationships

Use Prisma to retrieve the user along with their associated documents:

```ts
const user = await prisma.user.findFirst({
  include: {
    documents: true
  }
});
 
console.log(user);
```

If the result includes the document list, the relationship between `User` and `Document` is functional.

---

#### Step 9: Verify OCR Result Storage

Once the test document is created, test saving an OCR result:

```ts
await prisma.ocrResult.create({
  data: {
    documentId: "document_id_here",
    provider: "AWS_TEXTRACT",
    rawText: "Invoice No: INV-001\nTotal: 120 USD"
  }
});
```

Make sure the model name matches the case generated by Prisma (e.g., `ocrResult` or `OCRResult`). Adjust according to your project settings.

---

#### Step 10: Verify AI Analysis Storage

Test creating an AI analysis record:

```ts
await prisma.aiAnalysis.create({
  data: {
    documentId: "document_id_here",
    provider: "openai",
    model: "gpt-4.1-mini",
    documentType: "Invoice",
    summary: "This document is an invoice.",
    entities: [
      {
        type: "totalAmount",
        value: "120 USD"
      }
    ],
    keywords: ["invoice", "payment"],
    importantInformation: ["Total amount: 120 USD"],
    confidence: 0.9
  }
});
```

If Prisma throws an error, verify the model name and JSON data types.

---

#### Step 11: Verify Document Status Updates

After testing OCR or AI Analysis insertions, verify that the document status can be updated:

```ts
await prisma.document.update({
  where: {
    id: "document_id_here"
  },
  data: {
    status: "COMPLETED"
  }
});
```

Expected status in the database:

```text
COMPLETED
```

---

#### Step 12: Check Backend Logs

When querying the database, the backend should output logs:

```text
[DATABASE_HEALTH_CHECK_STARTED]
[DATABASE_CONNECTION_SUCCESS]
[DATABASE_TEST_USER_CREATED]
[DATABASE_TEST_DOCUMENT_CREATED]
```

If it fails:

```text
[DATABASE_CONNECTION_FAILED]
```

Do not log passwords or the complete `DATABASE_URL` string.

---

#### Common Errors

| Error                                          | Cause                                 | Solution                               |
| ---------------------------------------------- | ------------------------------------- | -------------------------------------- |
| `P1001`                                        | Cannot connect to the database        | Ensure the PostgreSQL service is active|
| `P1000`                                        | Incorrect username/password           | Verify the `DATABASE_URL` values       |
| `ECONNREFUSED`                                 | Database not active or wrong host/port| Start PostgreSQL and verify connection port|
| `relation does not exist`                      | Migration not executed                | Run `npx prisma migrate dev`           |
| `Environment variable not found: DATABASE_URL` | `.env` file missing or misplaced      | Ensure `.env` is loaded using dotenv   |
| `Unique constraint failed`                     | Attempting to create duplicate unique records| Change test data values               |
| `Invalid enum value`                           | Status value does not match enum list | Verify valid `DocumentStatus` values   |

---

#### Completion Checklist

You have completed this step when:

* PostgreSQL is running.
* The backend reads `DATABASE_URL` successfully.
* The Prisma Client is generated successfully.
* Prisma Studio runs without errors.
* The database health check API returns a successful response.
* Creating test User records succeeds.
* Creating test Document records succeeds.
* Saving an `OCRResult` succeeds.
* Saving an `AIAnalysis` succeeds.
* Updating the `Document` status succeeds.
* No database connection errors remain.

---

#### Expected Outcome

Following this step, the DocuMind AI system has a verified, stable database connection. The backend can use Prisma to store and retrieve data, preparing the system for integration with document uploads, OCR processing with Amazon Textract, AI Analysis with Gemini/OpenAI, notifications, and the Admin Panel.
