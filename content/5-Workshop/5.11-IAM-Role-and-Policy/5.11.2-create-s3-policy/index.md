---
title : "Create S3 Policy"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.11.2. </b> "
---

#### Overview

In this step, we will create the **IAM Policy for Amazon S3** so that the **DocuMind AI** backend and worker have permissions to upload, read, and manage documents in the S3 bucket.

Amazon S3 is the storage repository for the original documents uploaded by users. Therefore, the backend requires `PutObject` permissions to upload files, while the worker needs `GetObject` permissions to read files and forward them to Amazon Textract for OCR.

This policy should restrict access to the project's specific bucket, rather than granting permissions on all S3 resources globally.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an IAM Policy for Amazon S3.
* Authorize the backend to upload files to S3.
* Authorize the worker to read files from S3.
* Restrict permissions to the DocuMind AI project bucket.
* Prepare the policy to be attached to the IAM Role in subsequent steps.

---

#### The Role of the S3 Policy in DocuMind AI

The S3 Policy enables the backend/worker to:

* Upload document payloads to S3.
* Read documents from S3 for OCR execution.
* Delete objects when users remove documents.
* List the bucket contents to validate objects.
* Block public internet access to documents.

Permission flow:

```text id="rx4dh0"
IAM Role
  |
  v
S3 Policy
  |
  v
Amazon S3 Bucket
  |
  v
documents stored as s3Key
```

![s3-policy](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.2-create-s3-policy/s3-policy.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the S3 IAM Policy details or JSON definition page inside the AWS Console.
> Save the image at the path:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.2-create-s3-policy/s3-policy.png`

---

#### S3 Permissions Required

Minimal required actions:

```text id="kjqbh7"
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

Action descriptions:

| Action            | Purpose                                  |
| ----------------- | ---------------------------------------- |
| `s3:PutObject`    | Backend uploads document payloads to S3  |
| `s3:GetObject`    | Worker reads document payloads from S3   |
| `s3:DeleteObject` | Deletes files when user/admin deletes them|
| `s3:ListBucket`   | Validates objects inside the bucket      |

---

#### Step 1: Open the IAM Policies Console

Log in to the AWS Console, search for:

```text id="6xlwbs"
IAM
```

Select:

```text id="67uxve"
Policies
```

Then click:

```text id="ouiznb"
Create policy
```

---

#### Step 2: Choose JSON Editor

On the policy creation page, choose the tab:

```text id="ul7gey"
JSON
```

Then paste the policy JSON defined below.

---

#### Step 3: S3 Policy JSON Document

Replace `documind-document-storage` with your actual bucket name.

```json id="kw34gm"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindS3ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::documind-document-storage/*"
    },
    {
      "Sid": "DocuMindS3ListBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::documind-document-storage"
    }
  ]
}
```

---

#### Step 4: Name the Policy

Specify a clear name for the policy:

```text id="x7h0dj"
DocuMindS3AccessPolicy
```

Or:

```text id="v2cx91"
DocMind-S3-Document-Storage-Policy
```

Proposed description:

```text id="95cz91"
Allows DocuMind AI backend and worker to upload, read, delete and list documents in the project S3 bucket.
```

---

#### Step 5: Create the Policy

Review the settings:

| Component      | Value                              |
| -------------- | ---------------------------------- |
| Service        | Amazon S3                          |
| Bucket         | DocuMind AI Project Bucket         |
| Object actions | PutObject, GetObject, DeleteObject |
| Bucket action  | ListBucket                         |

Then click:

```text id="5jacb1"
Create policy
```

---

#### Step 6: Verify the Created Policy

After creation, inspect the policy:

* Confirm the policy name matches.
* Verify the resource ARN targets the correct bucket.
* Ensure no global wildcards (like `arn:aws:s3:::*`) are used.
* Ensure no public read/write configurations are allowed.
* Limit actions to the scope of this workshop.

Do not use overly broad policies like:

```json id="23pp3h"
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

Broad policies violate least privilege guidelines.

---

#### Step 7: Update Bucket Name in `.env`

Verify that the backend's `.env` references the correct bucket name:

```env id="cm9k82"
AWS_S3_BUCKET_NAME="documind-document-storage"
```

If your bucket has a different name, align both the policy and the `.env` settings.

---

#### Quick Test of S3 Permissions

After attaching the policy to the IAM Role, you can test access:

```bash id="x4b5nu"
aws s3 ls s3://documind-document-storage
```

Test file upload:

```bash id="rxi3t7"
aws s3 cp ./invoice-demo.pdf s3://documind-document-storage/test/invoice-demo.pdf
```

Test file download:

```bash id="sotbka"
aws s3 cp s3://documind-document-storage/test/invoice-demo.pdf ./downloaded-invoice-demo.pdf
```

---

#### Troubleshoot Scenarios

| Error                     | Cause                                   | Solution                                      |
| ------------------------- | --------------------------------------- | --------------------------------------------- |
| `AccessDenied`            | S3 permissions are missing              | Verify the policy is attached to the role     |
| `NoSuchBucket`            | Incorrect bucket name                   | Align the bucket name in the policy and `.env`|
| `PermanentRedirect`       | Region mismatch                         | Verify the `AWS_REGION` setting               |
| Upload failures           | Missing `s3:PutObject` action           | Add PutObject action in S3 policy             |
| Worker failed to read     | Missing `s3:GetObject` action           | Add GetObject action in S3 policy             |
| Cannot list bucket        | Missing `s3:ListBucket` action          | Add ListBucket action in S3 policy            |

---

#### Completion Checklist

You have completed this step when:

* The S3 IAM Policy is created.
* Access is restricted to the DocuMind AI project bucket.
* The policy grants `s3:PutObject`.
* The policy grants `s3:GetObject`.
* The policy grants `s3:DeleteObject` for cleanup.
* The policy grants `s3:ListBucket`.
* Wildcard privileges are restricted.
* The policy is ready to be attached to the IAM Role.

---

#### Expected Outcome

Following this step, DocuMind AI has a configured IAM Policy for Amazon S3. The backend can upload payloads to S3 and the worker can fetch those documents to execute OCR extractions using Amazon Textract.
