---
title : "Các lỗi thường gặp"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.13.4. </b> "
---

#### Overview

In this step, we will summarize **common errors** encountered during the deployment and verification of the **DocuMind AI** system.

Errors can occur across various layers, including EC2, the Node.js backend, PostgreSQL, Prisma, Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, IAM Roles, CloudWatch Logs, CORS headers, or the frontend dashboard.

Having a consolidated error reference guides troubleshooting steps and identifies which component should be verified first.

---

#### Objectives of This Step

Upon completing this step, you will:

* Identify common errors during backend deployment.
* Learn how to inspect EC2 and PM2 process errors.
* Learn how to troubleshoot database and Prisma errors.
* Learn how to troubleshoot AWS permission and IAM role errors.
* Learn how to troubleshoot S3, SQS, and Textract execution errors.
* Learn how to resolve Gemini/OpenAI integration errors.
* Learn how to verify frontend connection and CORS errors.
* Build a quick debugging checklist for the full workflow.

---

#### Debugging Flowchart

When encountering an error, follow the system workflow sequentially:

```text id="debug-flow"
Frontend
  |
  v
Backend API
  |
  v
PostgreSQL + Prisma
  |
  v
Amazon S3
  |
  v
Amazon SQS
  |
  v
Worker
  |
  v
Amazon Textract
  |
  v
Gemini / OpenAI
  |
  v
Dashboard Result
```

![common-errors](/images/5-Workshop/5.13-Deployment-and-Test/5.13.4-common-errors/common-errors.png)

> ⚠️ **Screenshot Suggestion:**
> Capture terminal windows or CloudWatch Logs displaying actual error outputs along with their corresponding solutions.
> Save the image at the path:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.4-common-errors/common-errors.png`

---

#### EC2 and Server Troubleshooting

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| Failed to SSH connect to EC2         | Incorrect key, wrong user, port 22 closed | Check key pair, use `ubuntu` user, check SG|
| Backend API unreachable externally   | Port 3000 is closed in Security Groups   | Open inbound TCP port 3000 or use Nginx   |
| Server runs out of memory (RAM)      | Instance type too small, heavy workload  | Configure swap files, upgrade instance type|
| `node` command not found             | Node.js runtime not installed            | Install Node.js LTS                       |
| `git clone` fails                    | Private repository or missing credentials | Configure GitHub tokens or SSH keys       |

System inspection commands:

```bash id="ec2-check"
df -h
free -m
pm2 list
pm2 logs
```

---

#### PM2 Process Manager Troubleshooting

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| PM2 process reports stopped status   | Application crashed during runtime       | Run `pm2 logs` to view stack traces       |
| Processes do not resume after reboot | PM2 startup configuration not saved      | Re-run PM2 startup and save processes     |
| Worker process fails to start        | Incorrect file path or startup script    | Verify scripts in `package.json`          |
| Log files consume excessive storage  | Log rotation not configured              | Install `pm2-logrotate` utility           |

Debugging commands:

```bash id="pm2-debug"
pm2 list
pm2 logs documind-backend
pm2 logs documind-worker
pm2 restart documind-backend
pm2 restart documind-worker
```

---

#### Node.js and Compilation Troubleshooting

| Error                                | Cause                                    | Solution                                  |
| ------------------------------------ | ---------------------------------------- | ----------------------------------------- |
| `npm install` failures               | Dependency versions mismatch             | Use `npm ci` or check Node version        |
| `npm run build` failures             | TypeScript syntax or configuration errors | Review the compiler logs                  |
| `Cannot find module`                 | Build missing files or incorrect imports  | Verify file path casing and imports       |
| Environment variables are empty      | Env files not loaded or misnamed         | Check environment variable loader logic   |
| `PORT already in use`                | Another process is bound to the port     | Terminate the conflicting process         |

Inspect port utilization:

```bash id="check-port"
sudo lsof -i :3000
```

Terminate conflicting process:

```bash id="kill-port"
sudo kill -9 process_id
```

---

#### PostgreSQL and Prisma Troubleshooting

| Error                     | Cause                                    | Solution                                  |
| ------------------------- | ---------------------------------------- | ----------------------------------------- |
| `P1001`                   | Cannot reach database server             | Check PostgreSQL service status           |
| `P1000`                   | Database credentials invalid             | Verify `DATABASE_URL` settings            |
| `database does not exist` | Target database has not been created     | Create the `docmind` database             |
| `relation does not exist` | Database tables missing                  | Execute `npx prisma migrate deploy`       |
| Prisma Client fails       | Prisma client types not generated        | Run `npx prisma generate`                 |
| Migration fail in prod    | Dev commands executed on production      | Run `npx prisma migrate deploy`           |

Inspect PostgreSQL connection:

```bash id="db-check"
psql "postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Verify Prisma:

```bash id="prisma-debug"
npx prisma validate
npx prisma generate
npx prisma migrate deploy
```

---

#### Amazon S3 Troubleshooting

| Error                      | Cause                                    | Solution                                  |
| -------------------------- | ---------------------------------------- | ----------------------------------------- |
| `AccessDenied`             | IAM Role lacks S3 permissions            | Inspect S3 IAM policies                   |
| `NoSuchBucket`             | Incorrect bucket name configured         | Verify `AWS_S3_BUCKET_NAME` env value     |
| `PermanentRedirect`        | Incorrect bucket region configured       | Verify `AWS_REGION` env value             |
| Files fail to upload       | Backend client triggers upload errors    | Inspect backend log outputs               |
| Object URL cannot be opened| Bucket configuration is private          | Correct behavior; access via SDK/presigned|

Verify S3 access:

```bash id="s3-debug"
aws s3 ls s3://documind-document-storage
aws s3 cp ./test.pdf s3://documind-document-storage/test/test.pdf
```

---

#### Amazon SQS Troubleshooting

| Error                         | Cause                                    | Solution                                  |
| ----------------------------- | ---------------------------------------- | ----------------------------------------- |
| `QueueDoesNotExist`           | Queue URL or Region is incorrect         | Verify `AWS_SQS_QUEUE_URL` env value      |
| `AccessDenied`                | IAM Role lacks SQS permissions           | Inspect SQS IAM policies                  |
| Worker does not poll messages | Worker process offline or queue is empty | Check PM2 status and SQS Console queues   |
| Duplicate message execution   | Worker fails to delete messages          | Confirm `DeleteMessage` is executed       |
| Messages routed to DLQ        | Processing fails repeatedly              | Inspect worker logs for exceptions        |

Verify SQS queuing:

```bash id="sqs-debug"
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue" \
  --message-body '{"documentId":"test","action":"PROCESS_DOCUMENT"}'
```

---

#### Amazon Textract Troubleshooting

| Error                          | Cause                                     | Solution                                  |
| ------------------------------ | ----------------------------------------- | ----------------------------------------- |
| `AccessDeniedException`        | IAM Role lacks Textract permissions       | Inspect Textract IAM policies             |
| `InvalidS3ObjectException`     | Object key is missing or unreachable      | Verify S3 bucket key and read permissions |
| `UnsupportedDocumentException` | File format is not supported by Textract  | Convert to PDF, PNG, JPG, or JPEG         |
| Extracted OCR text is empty    | Document image is blurry or has no text   | Use clear sample documents                |
| Region mismatch error          | Textract is not active in selected region | Align `AWS_REGION` settings               |

Supported file formats:

```text id="textract-format"
PDF
PNG
JPG
JPEG
```

*Note: Microsoft Office formats (DOCX, XLSX, PPTX) are not directly supported by Amazon Textract and must be converted to PDF before submission.*

---

#### Gemini and OpenAI Troubleshooting

| Error                | Cause                                     | Solution                                  |
| -------------------- | ----------------------------------------- | ----------------------------------------- |
| `401 Unauthorized`   | Invalid OpenAI API key                    | Verify `OPENAI_API_KEY` env value         |
| `API key not valid`  | Invalid Gemini API key                    | Verify `GEMINI_API_KEY` env value         |
| `429 Rate limit`     | API request quota limit reached           | Configure request retries or fallback provider|
| `insufficient_quota` | API balance exhausted                     | Add funds to developer billing account    |
| `model_not_found`    | Model name typo or model deprecated       | Verify model identifiers in env variables |
| JSON parsing failures| LLM output deviates from requested schema | Refine system prompts or enforce JSON mode|
| Execution Timeout    | Large payload or remote API delays        | Increase API client timeout settings      |

Monitor AI log statements:

```text id="ai-log"
[AI_PROVIDER_SELECTED]
[OPENAI_REQUEST_STARTED]
[GEMINI_REQUEST_STARTED]
[AI_PROVIDER_FALLBACK_USED]
[AI_ANALYSIS_FAILED]
```

---

#### IAM Roles and Permissions Troubleshooting

| Error                           | Cause                                    | Solution                                  |
| ------------------------------- | ---------------------------------------- | ----------------------------------------- |
| `Unable to locate credentials`  | EC2 instance lacks associated IAM Role   | Attach the IAM Role to the EC2 instance   |
| `AccessDenied` on API calls     | Attached policy lacks required privileges | Update policy JSON definitions            |
| Assumed role fails to resolve   | CLI overrides active role configuration  | Clear local access key configurations     |
| Secrets retrieval fails         | Missing `secretsmanager:GetSecretValue`  | Add secrets manager action to IAM policies|
| CloudWatch logging fails        | Missing `logs:PutLogEvents` permission   | Add CloudWatch actions to IAM policies    |

Verify instance identity:

```bash id="iam-debug"
aws sts get-caller-identity
```

---

#### CloudWatch Logs Troubleshooting

| Error                    | Cause                                    | Solution                                  |
| ------------------------ | ---------------------------------------- | ----------------------------------------- |
| Log group missing        | Not created or created in wrong region   | Create `/docmind/application` in correct region|
| Log stream missing       | Client failed stream initialization     | Check application logger startup code     |
| Logs fail to populate    | IAM role lacks writing permissions       | Inspect CloudWatch IAM policy             |
| Credentials leak in logs | Logging raw context or configurations    | Mask sensitive variables before logging   |

Ensure these variables are never written to logs:

```text id="no-secret-log"
password
JWT token
OPENAI_API_KEY
GEMINI_API_KEY
DATABASE_URL full password
AWS credentials
```

---

#### CORS and Frontend Troubleshooting

| Error                         | Cause                                    | Solution                                  |
| ----------------------------- | ---------------------------------------- | ----------------------------------------- |
| CORS block error              | Frontend domain not allowed in backend CORS| Update `CORS_ORIGIN` env variable         |
| API connection fails          | Client requests incorrect backend URL    | Update frontend API base URL env variable |
| `401 Unauthorized` response   | Missing or expired JWT credentials       | Request new session or re-authenticate    |
| `403 Forbidden` response      | Accessing resources owned by other users | Confirm database ownership rules          |
| Dashboard data fails to refresh| Status polling loop is inactive          | Confirm status query loops are running    |
| Upload fails via dashboard UI | Incorrect field name mapping             | Inspect requests in the browser Network tab|

Example frontend environment file (`.env`):

```env id="frontend-env"
VITE_API_BASE_URL="http://your-ec2-public-ip:3000"
```

---

#### AWS WAF Troubleshooting

| Error                          | Cause                                     | Solution                                  |
| ------------------------------ | ----------------------------------------- | ----------------------------------------- |
| WAF does not block requests    | Not Associated with ALB or CloudFront     | Link resources inside WAF Console         |
| Metrics are not visible        | CloudWatch metric publishing disabled     | Enable visibility configurations          |
| Legitimate requests blocked    | Managed rule configurations too strict    | Toggle rule to Count mode or add exception|
| Backend receives exploit traffic| Client bypassed WAF and called API directly| Restrict backend to ALB traffic only      |

---

#### Quick Debug Checklist

Verify in sequence:

1. Backend API health endpoints.
2. PM2 backend log outputs.
3. PM2 worker log outputs.
4. Database connectivity and Prisma status.
5. IAM Role assumed identity (`aws sts`).
6. S3 bucket connection and upload permissions.
7. SQS queue path and message consumption.
8. Textract OCR API calls and logs.
9. Gemini/OpenAI API credentials and budgets.
10. Document status values in the database.
11. Frontend API base URL settings.
12. CloudWatch log group events.

---

#### Common Debugging Commands

```bash id="quick-debug"
pm2 list
pm2 logs documind-backend
pm2 logs documind-worker
 
aws sts get-caller-identity
aws s3 ls s3://documind-document-storage
 
npx prisma validate
npx prisma generate
npx prisma migrate deploy
```

---

#### Completion Checklist

You have completed this step when you understand how to debug:

* EC2 and PM2 process crashes.
* Backend TypeScript compilation errors.
* PostgreSQL database connections and Prisma migrations.
* S3 storage, SQS messaging, and Textract OCR actions.
* Gemini/OpenAI API client integrations.
* IAM Role permissions.
* CloudWatch logs publishing.
* CORS headers and frontend connections.
* Full end-to-end processing workflows.

---

#### Expected Outcome

Following this step, you have a reference document compiling common deployment and verification errors for DocuMind AI. When errors arise, you can quickly locate and isolate the source across frontend, backend, database, AWS resources, AI gateway, or IAM permission scopes.
