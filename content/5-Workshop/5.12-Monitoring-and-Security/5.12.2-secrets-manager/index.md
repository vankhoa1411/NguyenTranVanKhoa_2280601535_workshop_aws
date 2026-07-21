---
title : "Secrets Manager"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.12.2. </b> "
---

#### Overview

In this step, we will configure **AWS Secrets Manager** for the **DocuMind AI** system.

Secrets Manager is utilized to store sensitive credentials such as `DATABASE_URL`, `JWT_SECRET`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, and other production configurations. Instead of saving these secrets directly in the production `.env` file or hardcoding them in source files, the backend can fetch secrets from AWS Secrets Manager during application initialization.

This configuration enhances security, simplifies secret lifecycle management, and minimizes credential leakage risks.

---

#### Objectives of This Step

Upon completing this step, you will:

* Store credentials in AWS Secrets Manager for DocuMind AI.
* Save sensitive environment variables inside the secret key.
* Configure the backend to read secrets at startup in production.
* Verify the IAM Role possesses secret reading permissions.
* Omit committing real `.env` settings to GitHub repositories.
* Establish a production-ready secret management setup.

---

#### The Role of Secrets Manager in DocuMind AI

Secrets Manager stores:

```text id="8nmy4f"
DATABASE_URL
JWT_SECRET
OPENAI_API_KEY
GEMINI_API_KEY
AI_PROVIDER
AWS_SQS_QUEUE_URL
```

Optional configurations:

```text id="yg9tns"
SMTP_HOST
SMTP_USER
SMTP_PASSWORD
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
```

If email verification or OAuth logins are integrated.

---

#### Secrets Manager Architecture

Credential retrieval flow:

```text id="ug46x8"
Backend / Worker
  |
  v
IAM Role
  |
  v
AWS Secrets Manager
  |
  v
Load secrets into application config
```

![secrets-manager](/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.2-secrets-manager/secrets-manager.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the secret details page for `docmind/backend` inside the AWS Secrets Manager Console, ensuring sensitive values are masked before inclusion.
> Save the image at the path:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.2-secrets-manager/secrets-manager.png`

---

#### Step 1: Open the Secrets Manager Console

Log in to the AWS Console, search for:

```text id="q3wkia"
Secrets Manager
```

Select:

```text id="27uynp"
AWS Secrets Manager
```

Click:

```text id="g7epx2"
Store a new secret
```

---

#### Step 2: Select Secret Type

Choose:

```text id="gwtfhj"
Other type of secret
```

Then choose the key/value format to add key parameters.

---

#### Step 3: Input Secret Key/Value Parameters

Input key credentials:

```json id="kd3zu5"
{
  "DATABASE_URL": "postgresql://username:password@host:5432/docmind",
  "JWT_SECRET": "your-jwt-secret",
  "OPENAI_API_KEY": "your-openai-api-key",
  "GEMINI_API_KEY": "your-gemini-api-key",
  "AI_PROVIDER": "openai"
}
```

*Note: Ensure values are masked when capturing screens for reports.*

---

#### Step 4: Name the Secret

Proposed secret name:

```text id="l9gzz0"
docmind/backend
```

Or:

```text id="m3be15"
docmind/production
docmind/ai-keys
```

Proposed description:

```text id="igjyxg"
Secrets for DocuMind AI backend production environment.
```

---

#### Step 5: Configure Rotation Options

For development/workshop setups, disable automatic rotation:

```text id="va2djl"
Disable automatic rotation
```

In production, you can configure rotation triggers based on compliance requirements.

---

#### Step 6: Store the Secret

Review the settings, then click:

```text id="o4v3ik"
Store
```

Once saved, copy the Secret name:

```text id="r9g4zk"
docmind/backend
```

---

#### Step 7: Configure Environment Variables

In production `.env` files, retain only:

```env id="pycfw1"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

Remove plaintext parameters:

```env id="ulbuxw"
OPENAI_API_KEY=""
GEMINI_API_KEY=""
JWT_SECRET=""
DATABASE_URL=""
```

---

#### Step 8: Install Secrets Manager SDK

If deploying a Node.js backend, install:

```bash id="h6sqyh"
npm install @aws-sdk/client-secrets-manager
```

---

#### Step 9: Create Secrets Service

Create the file:

```text id="xdoil8"
src/services/secrets.service.ts
```

Example service:

```ts id="3kuv7z"
import {
  SecretsManagerClient,
  GetSecretValueCommand
} from "@aws-sdk/client-secrets-manager";
 
const client = new SecretsManagerClient({
  region: process.env.AWS_REGION
});
 
export async function loadSecretsFromManager() {
  if (process.env.USE_SECRETS_MANAGER !== "true") {
    return {};
  }
 
  const secretName = process.env.AWS_SECRET_NAME;
 
  if (!secretName) {
    throw new Error("AWS_SECRET_NAME is required when USE_SECRETS_MANAGER=true");
  }
 
  const command = new GetSecretValueCommand({
    SecretId: secretName
  });
 
  const response = await client.send(command);
 
  if (!response.SecretString) {
    throw new Error("SecretString is empty");
  }
 
  return JSON.parse(response.SecretString);
}
```

---

#### Step 10: Load Secrets on Application Start

In the startup module (e.g., `server.ts`):

```ts id="d3yjnr"
import dotenv from "dotenv";
import { loadSecretsFromManager } from "./services/secrets.service";
 
dotenv.config();
 
async function bootstrap() {
  const secrets = await loadSecretsFromManager();
 
  for (const [key, value] of Object.entries(secrets)) {
    if (!process.env[key]) {
      process.env[key] = String(value);
    }
  }
 
  console.log("[SECRETS_LOADED]");
 
  // start app here
}
 
bootstrap();
```

Do not output secret values in logs. Simply log that secrets loaded successfully.

---

#### Step 11: Verify IAM Permissions

The backend/worker role requires:

```text id="jdpqb9"
secretsmanager:GetSecretValue
```

This was configured in step **5.11 - IAM Role and Policy**.

Test using the AWS CLI:

```bash id="dpy08m"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

If successful, the IAM Role possesses secret reading permissions.

---

#### Security Best Practices for Secrets

Adhere to the following guidelines:

* Do not commit production `.env` files to git repositories.
* Do not hardcode API keys in source files.
* Do not log secret values to standard output or CloudWatch logs.
* Restrict `GetSecretValue` permissions to authorized IAM Roles only.
* Name secrets using project prefix paths (e.g., `docmind/`).
* Mask secret values in screenshot artifacts.
* Rotate credentials immediately if a key leakage is suspected.

---

#### Verification Steps

Manual verification steps:

1. Create a secret named `docmind/backend`.
2. Add `OPENAI_API_KEY`, `GEMINI_API_KEY`, and `JWT_SECRET`.
3. Set `USE_SECRETS_MANAGER=true` in the production `.env`.
4. Ensure the server's IAM Role allows secret reading.
5. Launch the backend application.
6. Verify log output displays `[SECRETS_LOADED]`.
7. Trigger an AI Analysis task.
8. Confirm that the application correctly authenticates API calls using the loaded secrets.

---

#### Troubleshoot Scenarios

| Error                       | Cause                                   | Solution                                      |
| --------------------------- | --------------------------------------- | --------------------------------------------- |
| `AccessDeniedException`     | Missing `GetSecretValue` permission     | Check the role's IAM policy                   |
| `ResourceNotFoundException` | Secret name typo                        | Verify `AWS_SECRET_NAME` spelling             |
| `SecretString is empty`     | Secret holds empty payload              | Check values inside Secrets Manager           |
| JSON parse error            | Secret format is not valid JSON         | Re-configure secret as key/value JSON pairs   |
| Secrets do not load         | `USE_SECRETS_MANAGER` flag is false     | Verify `.env` properties                      |
| Secrets leak in logs        | Logger output prints secret parameters  | Remove raw configurations from logging modules|

---

#### Completion Checklist

You have completed this step when:

* The secret is created in AWS Secrets Manager.
* The secret contains all required environment variables.
* The backend fetches secrets dynamically during start.
* The IAM Role possesses `secretsmanager:GetSecretValue` permissions.
* Real keys are omitted from the production `.env` configuration.
* Log files do not contain plaintext secrets.
* AI operations authenticate successfully using loaded secrets.

---

#### Expected Outcome

Following this step, DocuMind AI has a secure secret management layer configured via AWS Secrets Manager. The backend resolves configuration parameters dynamically in production, eliminating plaintext credentials from environment configuration files.
