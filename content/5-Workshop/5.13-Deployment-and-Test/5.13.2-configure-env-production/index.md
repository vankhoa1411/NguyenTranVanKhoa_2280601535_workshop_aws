---
title : "Cấu hình môi trường Production"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.13.2. </b> "
---

#### Overview

In this step, we will configure the **production environment** for the **DocuMind AI** system.

The production environment must be configured carefully to ensure the backend can connect to PostgreSQL, Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, CloudWatch Logs, and AWS Secrets Manager. Concurrently, sensitive configurations such as API keys, JWT secrets, and database URLs must be kept secure.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create a production configuration file for the backend.
* Configure required environment variables.
* Distinguish between local `.env` and production `.env` files.
* Configure AWS regions, S3 buckets, and SQS queues.
* Configure Gemini/OpenAI providers.
* Configure PostgreSQL database connections.
* Configure Secrets Manager (if used).
* Verify the backend reads the correct environment variables.

---

#### The Role of Production Configuration

The production configuration controls how the backend connects to external services.

Key configuration areas include:

* App configuration.
* Database configuration.
* AWS configuration.
* AI Provider configuration.
* Authentication configuration.
* CloudWatch configuration.
* Secrets Manager configuration.
* CORS configuration.

If configured incorrectly, the system will trigger errors, such as database connection failures, S3 upload issues, SQS queueing errors, or Gemini/OpenAI API failures.

---

#### Environment Configuration Flow

```text id="env-flow"
.env.production / Secrets Manager
  |
  v
Backend Config Loader
  |
  v
Backend API + Worker
  |
  v
External Services
```

![configure-env-production](/images/5-Workshop/5.13-Deployment-and-Test/5.13.2-configure-env-production/configure-env-production.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the `.env.production` file layout, ensuring that sensitive parameters like API keys, JWT secrets, and database passwords are redacted or masked.
> Save the image at the path:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.2-configure-env-production/configure-env-production.png`

---

#### Local vs. Production Environment Files

Local development typically uses:

```text id="local-env"
.env
```

Production environments use:

```text id="prod-env"
.env.production
```

Do not commit these files to GitHub repository trees.

Only commit template files:

```text id="example-env"
.env.example
```

---

#### App Configuration

```env id="app-config"
NODE_ENV="production"
PORT=3000
APP_NAME="DocuMind AI"
APP_URL="https://your-domain.com"
FRONTEND_URL="https://your-frontend-domain.com"
```

If testing via the public IP of the EC2 instance:

```env id="app-ip"
APP_URL="http://your-ec2-public-ip:3000"
FRONTEND_URL="http://your-frontend-domain-or-localhost"
```

---

#### Database Configuration

DocuMind AI utilizes **PostgreSQL + Prisma**, rather than Amazon RDS in this workshop.

```env id="database-config"
DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"
```

If PostgreSQL is running on the local EC2 instance:

```env id="database-localhost"
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

If PostgreSQL is hosted on a separate database server:

```env id="database-remote"
DATABASE_URL="postgresql://docmind:docmind_password@your-postgres-server-ip:5432/docmind"
```

---

#### AWS Configuration

```env id="aws-config"
AWS_REGION="ap-southeast-1"
 
AWS_S3_BUCKET_NAME="documind-document-storage"
 
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
```

If the backend runs on an EC2 instance associated with an IAM Role, omit:

```env id="no-access-key"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

The AWS SDK resolves credentials dynamically from the IAM Role metadata.

---

#### AI Provider Configuration

If using OpenAI as the primary AI provider:

```env id="openai-config"
AI_PROVIDER="openai"
 
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
```

If using Gemini as the primary AI provider:

```env id="gemini-config"
AI_PROVIDER="gemini"
 
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

If integrating both providers to support fallbacks, define variables for both:

```env id="multi-ai-config"
AI_PROVIDER="openai"
 
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
 
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

---

#### Authentication Configuration

```env id="auth-config"
JWT_SECRET="your-strong-jwt-secret"
JWT_EXPIRES_IN="1d"
REFRESH_TOKEN_EXPIRES_IN="7d"
```

If Google OAuth is integrated:

```env id="oauth-config"
GOOGLE_CLIENT_ID="your-google-client-id"
GOOGLE_CLIENT_SECRET="your-google-client-secret"
GOOGLE_CALLBACK_URL="https://your-domain.com/api/auth/google/callback"
```

---

#### CORS Configuration

```env id="cors-config"
CORS_ORIGIN="https://your-frontend-domain.com"
```

If testing locally:

```env id="cors-local"
CORS_ORIGIN="http://localhost:5173"
```

Avoid wildcard rules in production environments:

```env id="bad-cors"
CORS_ORIGIN="*"
```

---

#### CloudWatch Configuration

```env id="cloudwatch-config"
CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-production"
CLOUDWATCH_WORKER_LOG_STREAM="worker-production"
```

---

#### Secrets Manager Configuration

If fetching secrets dynamically via AWS Secrets Manager:

```env id="secrets-config"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

This overrides local environment parameters with keys (`DATABASE_URL`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `JWT_SECRET`) stored inside AWS Secrets Manager.

---

#### Complete `.env.production` Template

```env id="full-env-production"
NODE_ENV="production"
PORT=3000
APP_NAME="DocuMind AI"
APP_URL="https://your-domain.com"
FRONTEND_URL="https://your-frontend-domain.com"
 
DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"
 
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
 
AI_PROVIDER="openai"
 
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
 
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
 
JWT_SECRET="your-strong-jwt-secret"
JWT_EXPIRES_IN="1d"
REFRESH_TOKEN_EXPIRES_IN="7d"
 
CORS_ORIGIN="https://your-frontend-domain.com"
 
CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-production"
CLOUDWATCH_WORKER_LOG_STREAM="worker-production"
```

---

#### `.env.example` Template File

The `.env.example` template exposes key fields without storing active secrets:

```env id="env-example"
NODE_ENV=
PORT=
 
DATABASE_URL=
 
AWS_REGION=
AWS_S3_BUCKET_NAME=
AWS_SQS_QUEUE_URL=
AWS_SQS_DLQ_URL=
 
AI_PROVIDER=
 
OPENAI_API_KEY=
OPENAI_MODEL=
OPENAI_EMBEDDING_MODEL=
 
GEMINI_API_KEY=
GEMINI_MODEL=
GEMINI_EMBEDDING_MODEL=
 
JWT_SECRET=
JWT_EXPIRES_IN=
REFRESH_TOKEN_EXPIRES_IN=
 
CORS_ORIGIN=
 
USE_SECRETS_MANAGER=
AWS_SECRET_NAME=
```

---

#### Environment Verification Logging

Configure logging to report config loading status safely:

```text id="safe-config-log"
[CONFIG_LOADED]
NODE_ENV=production
AWS_REGION=ap-southeast-1
AI_PROVIDER=openai
```

Do not log sensitive parameters:

```text id="no-log-secret"
DATABASE_URL
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
```

---

#### Verification Steps

Manual verification steps:

1. Create the `.env.production` file.
2. Compile project files (`npm run build`).
3. Deploy database schemas (`npx prisma migrate deploy`).
4. Launch the API server via PM2.
5. Query health check APIs.
6. Upload a document payload.
7. Verify storage objects in S3.
8. Verify queue messages in SQS.
9. Launch the worker process.
10. Verify successful execution of OCR and AI Analysis tasks.

---

#### Troubleshoot Scenarios

| Error                         | Cause                                    | Solution                                  |
| ----------------------------- | ---------------------------------------- | ----------------------------------------- |
| Env variables not resolved    | Incorrect file name or dotenv missing    | Verify config loader imports              |
| Database connection fails     | Incorrect `DATABASE_URL` settings        | Test DB connections via `psql`            |
| S3 operations return 403      | IAM Role lacks S3 permissions            | Inspect attached IAM policies             |
| SQS queue is not found        | Incorrect Queue URL or Region            | Check queue URL values                    |
| AI models trigger failures    | Incorrect API key or model versions      | Verify OpenAI/Gemini credentials          |
| Requests blocked by CORS      | Incorrect `CORS_ORIGIN` setup            | Map frontend domains accurately           |
| Secret parameters leak        | Logger outputs configuration block       | Mask secret variables in logs             |

---

#### Completion Checklist

You have completed this step when:

* `.env.production` is defined on the server.
* App, database, AWS, AI, and auth configuration blocks are complete.
* AWS access keys are excluded from configurations.
* No credentials are committed to Git repositories.
* The backend resolves production settings correctly.
* Prisma connects to the PostgreSQL database.
* Backend calls S3, SQS, Textract, and Gemini/OpenAI services.
* CORS headers permit frontend client communication.
* AWS Secrets Manager integration is configured (if used).

---

#### Expected Outcome

Following this step, DocuMind AI has a complete production configuration setup. The backend and worker resolve system parameters, communicate with AWS and database dependencies, and are prepared to run end-to-end testing cycles.
