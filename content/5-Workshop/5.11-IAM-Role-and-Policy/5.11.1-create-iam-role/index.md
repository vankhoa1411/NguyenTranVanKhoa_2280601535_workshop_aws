---
title : "Create IAM Role"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.11.1. </b> "
---

#### Overview

In this step, we will create the **IAM Role** for the **DocuMind AI** system.

The IAM Role is used to grant permissions to the backend or worker to access AWS services such as **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Amazon CloudWatch**, and **AWS Secrets Manager** without needing to hardcode `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in the source code.

This is a critical security step for deploying the backend to EC2 or any production server environment.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an IAM Role for the backend/worker.
* Understand the role of IAM Roles in DocuMind AI.
* Know how to select the appropriate trusted entity.
* Prepare the role to attach IAM Policies in subsequent steps.
* Avoid hardcoding AWS credentials in production.

---

#### The Role of the IAM Role in DocuMind AI

In the **DocuMind AI** system, the IAM Role is used to authorize the backend/worker to perform the following:

* Upload document payloads to Amazon S3.
* Retrieve documents from Amazon S3.
* Post messages to Amazon SQS.
* Retrieve and delete messages from Amazon SQS.
* Invoke Amazon Textract to perform OCR.
* Write application logs to Amazon CloudWatch.
* Fetch secrets from AWS Secrets Manager.

IAM Role execution flow:

```text id="6xkw9n"
Backend / Worker / EC2
  |
  v
IAM Role
  |
  v
AWS Services
  |
  |-- Amazon S3
  |-- Amazon SQS
  |-- Amazon Textract
  |-- CloudWatch
  |-- Secrets Manager
```

![create-iam-role](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.1-create-iam-role/create-iam-role.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the created IAM Role inside the AWS Console, or draw a diagram representing the Backend/Worker using the IAM Role to access S3, SQS, Textract, CloudWatch, and Secrets Manager.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.1-create-iam-role/create-iam-role.png`

---

#### Step 1: Open the IAM Console

Log in to the AWS Console, and search for the service:

```text id="yfg2at"
IAM
```

Select:

```text id="j7od6i"
Identity and Access Management
```

In the sidebar navigation, select:

```text id="a19n6p"
Roles
```

Then click:

```text id="zf6tww"
Create role
```

---

#### Step 2: Choose Trusted Entity

Under **Trusted entity type**, if the backend/worker runs on EC2, select:

```text id="e25cbh"
AWS service
```

Under use case, choose:

```text id="qnspgm"
EC2
```

This allows EC2 instances to assume the IAM Role to call AWS services.

*Note: If the backend runs in a different environment later (e.g., ECS or Lambda), select the corresponding trusted entity. For this workshop, EC2 is appropriate as the backend is deployed on an EC2 server.*

---

#### Step 3: Skip Policy Association for Now

In the permissions step, you can skip attaching policies for now.

The detailed policies will be created in the next steps:

* S3 Policy.
* SQS + Textract Policy.
* CloudWatch + Secrets Manager Policy.

Once those policies are created, we will attach them to this IAM Role.

---

#### Step 4: Name the IAM Role

Specify a clear name representing the project:

```text id="kkkdft"
DocuMindBackendWorkerRole
```

Or:

```text id="uhqx3i"
DocMind-Backend-Worker-Role
```

The name should clearly indicate that the role is used for the DocuMind AI backend and worker.

---

#### Step 5: Add Role Description

In the description field, input:

```text id="2jknbi"
IAM Role for DocuMind AI backend and worker to access S3, SQS, Textract, CloudWatch and Secrets Manager.
```

---

#### Step 6: Create the Role

Review the parameters:

| Parameter      | Proposed Value                     |
| -------------- | ---------------------------------- |
| Trusted entity | AWS Service                        |
| Use case       | EC2                                |
| Role name      | DocuMindBackendWorkerRole          |
| Purpose        | Backend/Worker access AWS services |

Then click:

```text id="zrvbhs"
Create role
```

---

#### Step 7: Inspect Trust Relationship Policy

After creating the role, open it and navigate to the tab:

```text id="6cj6tx"
Trust relationships
```

The trust policy for EC2 typically looks like:

```json id="mckoxy"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This policy allows the EC2 service to assume the role.

---

#### Step 8: Why Omit Hardcoded AWS Credentials?

Do not store these parameters directly in your production `.env` configuration:

```env id="mzpuwu"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Reasons:

* High risk of exposure if accidentally committed to GitHub.
* Hard to rotate keys.
* Unsafe if multiple environments share the same keys.
* Violates AWS deployment best practices.

Instead, when running the backend on an EC2 instance with the IAM Role attached, the AWS SDK automatically fetches temporary credentials.

---

#### Step 9: Required Configuration Settings

Although access keys are omitted, the backend still requires configuration variables such as:

```env id="i0xlcq"
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
```

There is no need to define access key variables in production.

---

#### Completion Checklist

You have completed this step when:

* The IAM Role is created for the backend/worker.
* The role is configured with the correct trusted entity.
* If deploying on EC2, the trust policy trusts `ec2.amazonaws.com`.
* The role is named clearly based on the project.
* The role is ready to accept S3, SQS, Textract, CloudWatch, and Secrets Manager policies.
* You understand why access keys are excluded in production environments.

---

#### Expected Outcome

Following this step, DocuMind AI has a foundation IAM Role to grant the backend and worker access to AWS services securely. The detailed policies will be created and attached in subsequent steps.
