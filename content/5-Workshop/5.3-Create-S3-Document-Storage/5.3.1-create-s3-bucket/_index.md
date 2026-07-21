---
title : "Create S3 Bucket"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.3.1. </b> "
---

#### Overview

In this step, we will create an **Amazon S3 Bucket** to store the original documents for the **DocuMind AI** system.

Amazon S3 serves as the storage location for user-uploaded input files, such as invoices, forms, contracts, or internal documents. Once a user uploads a document from the web interface, the backend will upload the file to S3 and save the metadata to **PostgreSQL + Prisma**.

Files stored on S3 will be used in subsequent steps, especially when the worker retrieves the document to perform OCR via **Amazon Textract**.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create an S3 bucket dedicated to DocuMind AI.
* Configure the bucket to be private.
* Enable Block Public Access to protect documents.
* Enable encryption to enhance security.
* Understand the role of `bucketName` and `s3Key` in the system.
* Prepare the bucket for the backend to upload documents to S3.

---

#### The Role of S3 Bucket in DocuMind AI

In the DocuMind AI system, the S3 Bucket is used to:

* Store original document files uploaded by users.
* Provide the `s3Key` path so the worker can retrieve files for processing.
* Decouple file storage from the backend server.
* Help the system scale easily as the document count grows.
* Ensure documents are not directly exposed to the public internet.

The workflow in this step:

```text
User
  |
  v
React Frontend
  |
  v
Backend API
  |
  v
Amazon S3 Bucket
  |
  v
PostgreSQL + Prisma storing metadata
```

![create-s3-bucket](/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.1-create-s3-bucket/create-s3-bucket.png)


> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.1-create-s3-bucket/create-s3-bucket.png`

---

#### Step 1: Access the Amazon S3 Service

Log in to the AWS Console, then search for the service:

```text
S3
```

Select:

```text
Amazon S3
```

Then click:

```text
Create bucket
```

---

#### Step 2: Name the Bucket

In the **Bucket name** section, enter a bucket name for the project.

For example:

```text
documind-document-storage
```

If the bucket name is already taken, you can append your name or a project code:

```text
documind-document-storage-tuan
documind-document-storage-2026
documind-ai-documents-tuan
```

> ⚠️ **Note:**
> S3 bucket names must be globally unique across all of AWS, not just within your own account.

---

#### Step 3: Choose AWS Region

Select the region you are using for the entire workshop.

For example:

```text
Asia Pacific (Singapore) - ap-southeast-1
```

Or the region you configured in your backend `.env` file:

```env
AWS_REGION="ap-southeast-1"
```

It is recommended to use the same region for:

* S3
* SQS
* Textract
* CloudWatch
* Secrets Manager
* Backend/EC2 (if applicable)

Using the same region reduces configuration errors and makes the system easier to manage.

---

#### Step 4: Configure Object Ownership

In the **Object Ownership** section, select:

```text
ACLs disabled
```

This configuration ensures that object ownership belongs to the bucket owner and restricts the use of manual ACLs.

Recommended settings:

```text
Object Ownership: Bucket owner enforced
ACLs: Disabled
```

---

#### Step 5: Enable Block Public Access

In the **Block Public Access settings for this bucket** section, keep the default configuration:

```text
Block all public access: Enabled
```

This is a critical configuration because user documents should not be publicly accessible directly from the internet.

DocuMind AI should control document viewing permissions via the backend API rather than letting users access S3 public URLs directly.

---

#### Step 6: Configure Bucket Versioning

In the **Bucket Versioning** section, you can select:

```text
Disable
```

or:

```text
Enable
```

For the workshop/demo, you can choose:

```text
Disable
```

If you want to keep multiple versions of the same document, you can select:

```text
Enable
```

However, keeping versioning enabled may increase storage costs when uploading many files or overwriting files multiple times.

---

#### Step 7: Configure Encryption

In the **Default encryption** section, enable default encryption.

Select:

```text
Server-side encryption with Amazon S3 managed keys (SSE-S3)
```

Recommended configuration:

```text
Default encryption: Enabled
Encryption type: SSE-S3
Bucket key: Enable
```

Enabling encryption ensures that documents are encrypted when stored on S3.

---

#### Step 8: Create the Bucket

After verifying all information, click:

```text
Create bucket
```

Upon successful creation, the bucket will appear in your S3 buckets list.

---

#### Step 9: Verify the Bucket Configuration

Select the newly created bucket and verify the following:

| Component        | Expected State             |
| ---------------- | -------------------------- |
| Bucket name      | Correct project bucket name |
| Region           | Correct region being used  |
| Public access    | Blocked                    |
| Encryption       | Enabled                    |
| Object ownership | ACLs disabled              |

---

#### Step 10: Update Backend Environment Variables

Once the bucket is ready, add the bucket information to the backend `.env` file:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
```

If your bucket has a different name, replace it accordingly.

For example:

```env
AWS_S3_BUCKET_NAME="documind-document-storage-tuan"
```

---

#### Guidelines for Structuring `s3Key` in the Project

The backend should store files using a clear directory structure for easy management:

```text
uploads/{userId}/{documentId}/{fileName}
```

For example:

```text
uploads/user_001/doc_001/invoice-demo.pdf
```

This approach makes it easy to locate documents by user and document ID.

Example metadata stored in PostgreSQL:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "fileName": "invoice-demo.pdf",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "status": "UPLOADED"
}
```

---

#### Completion Checklist

You have completed this step when:

* The S3 bucket is successfully created.
* Public access to the bucket is blocked.
* Default encryption is enabled on the bucket.
* The bucket name has been added to the backend `.env` file.
* The backend can reference `AWS_S3_BUCKET_NAME`.
* You understand the role of `s3Bucket` and `s3Key` in the system.

---

#### Expected Outcome

Following this step, the DocuMind AI system has a secure place to store original documents using Amazon S3. This provides the foundation for subsequent steps, including CORS configuration, bucket security, file uploading, sending SQS messages, and performing Textract OCR processing.
