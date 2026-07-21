---
title : "Set Up PostgreSQL Database"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.7.1. </b> "
---

#### Overview

In this step, we will set up the **PostgreSQL Database** for the **DocuMind AI** system.

PostgreSQL is used to store core application data such as user profiles, uploaded documents, OCR text, AI analysis results, chat history, notifications, and audit logs. This is a critical component that enables the system to manage the document processing state from upload to the completion of AI analysis.

In this workshop, the system uses **PostgreSQL combined with Prisma ORM**, without using Amazon RDS. The database can run locally on your personal machine or on a dedicated PostgreSQL server depending on your deployment environment.

---

#### Objectives of This Step

Upon completing this step, you will:

* Install or prepare a PostgreSQL Database.
* Create a dedicated database for DocuMind AI.
* Configure the connection string `DATABASE_URL`.
* Verify that the backend can connect to PostgreSQL.
* Prepare the database for Prisma to create tables in the next step.

---

#### The Role of PostgreSQL in DocuMind AI

In the **DocuMind AI** system, PostgreSQL is used to store:

* User profiles and User/Admin permissions.
* Uploaded document metadata.
* File paths on Amazon S3 via `s3Bucket` and `s3Key`.
* Document processing statuses: `PENDING`, `UPLOADED`, `QUEUED`, `PROCESSING`, `OCR_COMPLETED`, `AI_ANALYZING`, `COMPLETED`, `FAILED`.
* OCR results from Amazon Textract.
* AI Analysis results from Gemini/OpenAI.
* AI Chat/RAG history.
* Notifications for Users/Admins.
* Audit Logs for critical operations.

Database-related data flow:

```text
Backend API / Worker
  |
  v
Prisma ORM
  |
  v
PostgreSQL Database
```

![postgresql-database](/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.1-setup-postgresql-database/postgresql-database.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the PostgreSQL database created in pgAdmin, DBeaver, TablePlus, or the terminal. Optionally, capture a diagram representing Backend API → Prisma ORM → PostgreSQL Database.
> Save the image at the path:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.1-setup-postgresql-database/postgresql-database.png`

---

#### Step 1: Install PostgreSQL

If running locally, you can install PostgreSQL according to your operating system.

For macOS, you can use Homebrew:

```bash
brew install postgresql
brew services start postgresql
```

For Ubuntu:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

For Windows, you can download PostgreSQL from the official website or run it via Docker.

If using Docker:

```bash
docker run --name docmind-postgres \
  -e POSTGRES_USER=docmind \
  -e POSTGRES_PASSWORD=docmind_password \
  -e POSTGRES_DB=docmind \
  -p 5432:5432 \
  -d postgres:16
```

---

#### Step 2: Create a Database for DocuMind AI

If using the PostgreSQL terminal:

```bash
psql -U postgres
```

Create the user and database:

```sql
CREATE USER docmind WITH PASSWORD 'docmind_password';
CREATE DATABASE docmind OWNER docmind;
GRANT ALL PRIVILEGES ON DATABASE docmind TO docmind;
```

Exit the PostgreSQL CLI:

```sql
\q
```

If you are using pgAdmin, DBeaver, or TablePlus, you can create the database using the GUI.

Recommended database name:

```text
docmind
```

---

#### Step 3: Configure the `DATABASE_URL` Environment Variable

Add the connection string to the backend `.env` file:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Explanation:

| Component          | Description                  |
| ------------------ | ---------------------------- |
| `postgresql://`    | PostgreSQL connection protocol|
| `docmind`          | Username                     |
| `docmind_password` | Password                     |
| `localhost`        | Database Host                |
| `5432`             | Default PostgreSQL port      |
| `docmind`          | Database Name                |

If the database is running on a different server, replace `localhost` with the IP address or domain of that server.

---

#### Step 4: Verify Database Connectivity

You can test connectivity by running:

```bash
psql "postgresql://docmind:docmind_password@localhost:5432/docmind"
```

If successful, the command will open the PostgreSQL interactive shell.

You can check the list of tables:

```sql
\dt
```

Initially, no tables will be displayed since Prisma migrations have not run yet.

---

#### Step 5: Verify Backend Reads `DATABASE_URL`

In the backend, ensure that the `.env` file is loaded correctly.

If using Node.js, ensure you have the `dotenv` package installed:

```bash
npm install dotenv
```

In the entry file (e.g., `src/server.ts` or `src/app.ts`), load environment variables:

```ts
import dotenv from "dotenv";
 
dotenv.config();
```

Prisma will then automatically utilize `DATABASE_URL` defined in the `.env` file.

---

#### Step 6: Database Security Best Practices

When deploying PostgreSQL for DocuMind AI, keep the following in mind:

* Do not commit your actual `DATABASE_URL` to GitHub.
* Do not use weak passwords in production environments.
* Only allow connections to the database from the backend.
* If hosting the database on a separate server, restrict access by IP.
* Do not expose the database port `5432` to the public internet unless required.
* Use `.env.example` to provide template environment variables.
* For production deployments, store `DATABASE_URL` in AWS Secrets Manager.

---

#### Step 7: Verify Database Status

If running PostgreSQL via Docker:

```bash
docker ps
```

Check the container logs:

```bash
docker logs docmind-postgres
```

If running the PostgreSQL service locally on your host machine:

```bash
pg_isready
```

Expected output:

```text
accepting connections
```

---

#### Common Errors

| Error                            | Cause                        | Solution                             |
| -------------------------------- | ---------------------------- | ------------------------------------ |
| `password authentication failed` | Incorrect username/password  | Verify the `DATABASE_URL` configuration |
| `database does not exist`        | Database has not been created| Create the `docmind` database        |
| `ECONNREFUSED`                   | PostgreSQL is not running    | Start the PostgreSQL service or Docker container|
| `port 5432 already in use`       | Port conflict                | Stop other conflicting services      |
| `role does not exist`            | User has not been created    | Create the PostgreSQL user role      |
| Backend fails to read `.env`     | dotenv package not loaded    | Check backend config file loading    |

---

#### Completion Checklist

You have completed this step when:

* PostgreSQL is installed or running via Docker.
* The database for DocuMind AI has been created.
* The database user is created.
* The `.env` file contains a valid `DATABASE_URL`.
* The backend can read the environment variables.
* The connection to PostgreSQL is successful.
* The system is ready to configure the Prisma schema in step 5.7.2.

---

#### Expected Outcome

Following this step, DocuMind AI has a PostgreSQL database configured to store application data. This serves as the foundation for the Prisma ORM to create the User, Document, OCRResult, AIAnalysis, ChatHistory, Notification, and AuditLog tables in subsequent steps.
