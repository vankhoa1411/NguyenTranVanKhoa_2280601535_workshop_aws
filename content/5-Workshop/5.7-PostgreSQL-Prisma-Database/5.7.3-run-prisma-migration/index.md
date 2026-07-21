---
title : "Run Prisma Migrations"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.7. </b> "
---

#### Overview

In this step, we will execute the **Prisma Migration** to generate the required tables in the **PostgreSQL Database** for the **DocuMind AI** system.

After defining the `schema.prisma` file, Prisma needs to run a migration to convert models like `User`, `Document`, `OCRResult`, `AIAnalysis`, `ChatHistory`, `Notification`, and `AuditLog` into actual tables within PostgreSQL.

Migrations help manage database schema changes in a controlled, trackable manner suitable for backend development environments.

---

#### Objectives of This Step

Upon completing this step, you will:

* Execute the initial migration for DocuMind AI.
* Create the tables in PostgreSQL.
* Generate the Prisma Client.
* Verify the created tables using Prisma Studio.
* Test that the backend can perform data operations.
* Prepare the database to store upload, OCR, and AI Analysis records.

---

#### What is Prisma Migration?

**Prisma Migration** is the mechanism used to map the schema defined in `schema.prisma` into table structures inside the database.

For example, when you define a model:

```prisma
model Document {
  id       String @id @default(uuid())
  fileName String
  s3Key    String
}
```

Prisma migration translates it into the corresponding table in PostgreSQL:

```text
Document
```

with columns representing the model attributes:

```text
id
fileName
s3Key
```

Workflow:

```text
schema.prisma
  |
  v
Prisma Migration
  |
  v
PostgreSQL Tables
  |
  v
Prisma Client
```

![prisma-migration](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.3-run-prisma-migration/prisma-migration.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the terminal screen displaying a successful run of `npx prisma migrate dev`, or open Prisma Studio to show the created tables.
> Save the image at the path:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.3-run-prisma-migration/prisma-migration.png`

---

#### Prerequisites Before Running Migrations

Before starting, ensure that:

* PostgreSQL is active.
* The `docmind` database is created.
* The `.env` file contains the `DATABASE_URL`.
* The `schema.prisma` file is configured with the PostgreSQL datasource.
* Prisma models are defined.
* `npx prisma validate` executes successfully.

Double-check the `DATABASE_URL`:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

---

#### Step 1: Format and Validate Schema

Format the schema file:

```bash
npx prisma format
```

Then validate the schema structure:

```bash
npx prisma validate
```

If the schema is valid, Prisma will report that no errors were found.

---

#### Step 2: Run the Initial Migration

Execute the command:

```bash
npx prisma migrate dev --name init
```

This command performs the following actions:

* Generates a migrations directory under `prisma/migrations`.
* Creates the required tables in PostgreSQL.
* Generates the Prisma Client.
* Synchronizes the database with your current schema state.

Once finished, a migration folder is created:

```text
prisma/
└── migrations/
    └── 20260610120000_init/
        └── migration.sql
```

---

#### Step 3: Generate Prisma Client

Although the `migrate dev` command automatically generates the Prisma Client, you can trigger it manually using:

```bash
npx prisma generate
```

The Prisma Client enables database queries using TypeScript.

Example:

```ts
const documents = await prisma.document.findMany();
```

---

#### Step 4: Verify Tables in Prisma Studio

Run:

```bash
npx prisma studio
```

Prisma Studio will launch a web interface, typically at:

```text
http://localhost:5555
```

Verify that the following tables exist:

```text
User
Document
OCRResult
AIAnalysis
ChatHistory
Notification
AuditLog
```

If the tables appear, the migration succeeded.

---

#### Step 5: Seed Optional Test Data

You can insert test users or documents via Prisma Studio or by running a seed script.

Example seed function:

```ts
await prisma.user.create({
  data: {
    email: "demo@docmind.ai",
    fullName: "Demo User",
    role: "USER"
  }
});
```

If the `passwordHash` field is required, provide a suitable hash value.

---

#### Step 6: Verify Backend Integration with Prisma

Create or check the Prisma client instantiation file:

```text
src/prisma/client.ts
```

Example implementation:

```ts
import { PrismaClient } from "@prisma/client";
 
export const prisma = new PrismaClient();
```

Test database connectivity in your backend:

```ts
const users = await prisma.user.findMany();
console.log(users);
```

If no errors are thrown, the backend connection is successful.

---

#### Step 7: Migration When Modifying Schema

If you modify or add models to `schema.prisma`, generate a new migration:

```bash
npx prisma migrate dev --name add_notification_table
```

Avoid manually updating PostgreSQL tables when using Prisma migrations, as it can cause synchronization issues between the schema and the database.

---

#### Step 8: Production Environments

In production, do not run `migrate dev`. Use the deployment command:

```bash
npx prisma migrate deploy
```

This command applies existing migrations directly to the production database without creating new migration files.

For local development, use:

```bash
npx prisma migrate dev
```

---

#### Troubleshoot Common Errors

| Error                    | Cause                       | Solution                             |
| ------------------------ | --------------------------- | ------------------------------------ |
| `P1001`                  | Database connection failed  | Verify that PostgreSQL is running    |
| `P1000`                  | Authentication failure      | Check username/password in `DATABASE_URL`|
| `P1012`                  | Invalid Prisma schema       | Validate using `npx prisma validate` |
| `database does not exist`| Database not created        | Create the `docmind` database        |
| `shadow database` error  | Insufficient DB permissions | Check PostgreSQL user privileges     |
| Prisma Studio fails      | Port conflict               | Restart or configure a different port|

---

#### Completion Checklist

You have completed this step when:

* `npx prisma format` executes successfully.
* `npx prisma validate` finishes without errors.
* `npx prisma migrate dev --name init` completes successfully.
* The Prisma Client is generated.
* Prisma Studio opens correctly.
* Core tables are created in PostgreSQL.
* The backend can query the database using the Prisma Client.
* The database is ready to support file uploads, OCR, AI Analysis, Notifications, and Administration.

---

#### Expected Outcome

Following this step, DocuMind AI has a working database schema deployed on PostgreSQL. The database tables are created, and the backend is configured to read/write document metadata, OCR results, AI Analysis, chat logs, notifications, and audit records via the Prisma Client.
