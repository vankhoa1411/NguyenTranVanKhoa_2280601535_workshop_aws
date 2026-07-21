---
title : "Deploy Backend to EC2"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.13.1. </b> "
---

#### Overview

In this step, we will deploy the **Backend API and Worker** components of the **DocuMind AI** system to **Amazon EC2**.

The Backend API accepts requests from the frontend, handles user authentication, uploads document payloads to Amazon S3, queues processing tasks in Amazon SQS, and exposes API endpoints for the document Dashboard. The SQS Worker fetches queued messages, calls Amazon Textract for OCR text extraction, sends parsed context to Gemini/OpenAI, and saves results in PostgreSQL.

Deploying backend services to EC2 runs the platform on a stable cloud server rather than a local workstation.

---

#### Objectives of This Step

Upon completing this step, you will:

* Provision or prepare an EC2 instance for backend hosting.
* Install Node.js, Git, and other required utilities.
* Clone the backend source code to the EC2 instance.
* Configure production environment settings.
* Install application dependencies.
* Build the TypeScript backend code.
* Execute the Backend API and Worker processes using PM2.
* Verify public backend API accessibility.

---

#### Backend Deployment Architecture

Deployment pipeline:

```text id="deploy-flow"
Developer
  |
  v
Git Repository
  |
  v
Amazon EC2
  |
  |-- Backend API
  |-- SQS Worker
  |
  v
AWS Services + PostgreSQL + AI Providers
```

![deploy-backend-ec2](/images/5-Workshop/5.13-Deployment-and-Test/5.13.1-deploy-backend-ec2/deploy-backend-ec2.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the running EC2 instance screen in AWS Console, the active SSH session terminal, and PM2 displaying backend and worker processes online.
> Save the image at the path:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.1-deploy-backend-ec2/deploy-backend-ec2.png`

---

#### Step 1: Provision EC2 Instance

In the AWS Console, create an EC2 instance with the following settings:

| Component      | Recommended Value                      |
| -------------- | -------------------------------------- |
| AMI            | Ubuntu Server 22.04 LTS or 24.04 LTS   |
| Instance type  | t2.micro or t3.micro for demo          |
| Storage        | 20GB or larger                         |
| Security Group | Open SSH port and backend API port     |
| IAM Role       | `DocuMindBackendWorkerRole`            |

Required Security Group Ports:

```text id="ports"
22    SSH
3000  Backend API
80    HTTP (if configuring Nginx proxy)
443   HTTPS (if configuring SSL)
```

If only running a demo test, port `3000` is sufficient. For production setups, configure Nginx as a reverse proxy listening on port `80/443`.

---

#### Step 2: SSH Connect to EC2

From your local machine, connect to the EC2 instance via SSH:

```bash id="ssh-ec2"
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

If key permission errors occur, adjust parameters:

```bash id="chmod-key"
chmod 400 your-key.pem
```

---

#### Step 3: Update System Packages

Once SSH access is established:

```bash id="update-server"
sudo apt update
sudo apt upgrade -y
```

Install core dependencies:

```bash id="install-basic"
sudo apt install git curl unzip build-essential -y
```

---

#### Step 4: Install Node.js runtime

Install the LTS version of Node.js:

```bash id="install-node"
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

Verify the installation:

```bash id="check-node"
node -v
npm -v
```

---

#### Step 5: Install PM2 Process Manager

PM2 manages backend API and worker processes in the background:

```bash id="install-pm2"
sudo npm install -g pm2
```

Verify PM2:

```bash id="check-pm2"
pm2 -v
```

---

#### Step 6: Clone Backend Source Code

Clone the repository using Git:

```bash id="clone-project"
git clone https://github.com/your-username/documind-ai.git
```

Change directory into the backend project folder:

```bash id="cd-backend"
cd documind-ai/backend
```

If directory names vary, list directory structures:

```bash id="ls-folder"
ls
```

*Note: If the directory change fails, confirm repository structure folder paths using `ls` before executing `cd`.*

---

#### Step 7: Install Dependencies

In the backend directory:

```bash id="npm-install"
npm install
```

If dependency version conflicts arise, clean install:

```bash id="npm-ci"
npm ci
```

---

#### Step 8: Create `.env.production` Configuration

Create the production environment file:

```bash id="create-env"
nano .env.production
```

Example content properties:

```env id="env-production"
NODE_ENV="production"
PORT=3000
 
DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"
 
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
 
AI_PROVIDER="openai"
 
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
 
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
 
JWT_SECRET="your-jwt-secret"
```

If using AWS Secrets Manager:

```env id="env-secrets"
NODE_ENV="production"
PORT=3000
 
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

---

#### Step 9: Build the Backend

Compile the TypeScript project files:

```bash id="build-backend"
npm run build
```

Upon a successful build, the target directory is compiled:

```text id="dist-folder"
dist/
```

---

#### Step 10: Run Prisma Migrations

Apply Prisma schema migrations to the production database:

```bash id="prisma-deploy"
npx prisma migrate deploy
```

Generate the database client:

```bash id="prisma-generate"
npx prisma generate
```

Avoid running development migration commands (`prisma migrate dev`) in production.

---

#### Step 11: Launch Backend API via PM2

If compiled server entrypoint resides at `dist/server.js`:

```bash id="pm2-backend"
pm2 start dist/server.js --name documind-backend
```

If executing scripts configured inside `package.json`:

```bash id="pm2-npm"
pm2 start npm --name documind-backend -- start
```

Inspect active processes:

```bash id="pm2-list"
pm2 list
```

---

#### Step 12: Launch SQS Worker via PM2

If the worker compiled code resides at `dist/workers/document.worker.js`:

```bash id="pm2-worker"
pm2 start dist/workers/document.worker.js --name documind-worker
```

Or execute worker script alias:

```bash id="pm2-worker-npm"
pm2 start npm --name documind-worker -- run worker
```

Review outputs:

```bash id="pm2-logs"
pm2 logs documind-backend
pm2 logs documind-worker
```

---

#### Step 13: Configure PM2 Autostart

Enable PM2 processes to restart upon server reboots:

```bash id="pm2-startup"
pm2 startup
```

Follow the CLI instructions output by PM2.

Save configuration snapshots:

```bash id="pm2-save"
pm2 save
```

---

#### Step 14: Verify Health Endpoint

Trigger a local request:

```bash id="curl-health"
curl http://localhost:3000/api/health
```

Test public reachability:

```text id="public-health"
http://your-ec2-public-ip:3000/api/health
```

Expected response payload:

```json id="health-response"
{
  "success": true,
  "message": "DocuMind AI backend is running"
}
```

---

#### Step 15: Verify Active IAM Role

Confirm the EC2 metadata IAM bindings:

```bash id="sts-check"
aws sts get-caller-identity
```

Expected assumed role output format:

```text id="assumed-role"
assumed-role/DocuMindBackendWorkerRole
```

If missing, verify IAM role attachment details.

---

#### Troubleshoot Scenarios

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| Failed to SSH                        | Wrong key file or blocked port 22        | Check security group inbound rules        |
| Directory path missing               | Typo in repo or checkout directory path  | List folder contents using `ls`           |
| `npm install` failures               | Node version conflicts or compiler missing| Confirm Node runtime matches requirements |
| Compilation failures                 | Typo inside source code or configuration | Review compiler errors output             |
| Port 3000 unreachable                | Security group blocks port 3000          | Add inbound rules for port 3000           |
| `DATABASE_URL` connectivity fails    | Relational database routing error        | Check network routes and DB credentials   |
| AWS operations return `AccessDenied` | IAM role policies missing configuration  | Review IAM policies in step 5.11          |
| Worker process fails to start        | Typo in build script path                | Verify script mapping in `package.json`   |

---

#### Completion Checklist

You have completed this step when:

* The EC2 instance is running.
* SSH connection is established.
* Node.js, Git, and PM2 are installed on the server.
* Backend source code is cloned.
* The `.env.production` file is configured.
* TypeScript code builds successfully.
* Production Prisma migrations are applied.
* Backend API process is managed by PM2.
* SQS Worker process is managed by PM2.
* The health check endpoint responds successfully.
* The EC2 instance assumes the correct IAM Role.

---

#### Expected Outcome

Following this step, the DocuMind AI Backend API and SQS Worker are running on the EC2 server, prepared to integrate with the frontend UI, S3 storage, SQS messaging queues, Textract OCR, PostgreSQL databases, and Gemini/OpenAI services.
