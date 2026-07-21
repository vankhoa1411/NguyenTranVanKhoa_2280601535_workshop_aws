---
title : "Configure Amazon Textract Permissions"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.5.1. </b> "
---

#### Overview

In this step, we will configure the necessary permissions to enable the **DocuMind AI** backend or worker to call **Amazon Textract** to extract text from documents.

Amazon Textract is AWS's OCR service, which analyzes PDF documents or images and returns the extracted textual content. In the DocuMind AI pipeline, Textract is used after a document is uploaded to **Amazon S3** and its processing message is sent to **Amazon SQS**.

The worker will retrieve the message from SQS, extract the `s3Bucket` and `s3Key` information, and then call Textract to perform OCR.

---

#### Objectives of This Step

Upon completing this step, you will:

* Understand the role of Amazon Textract in DocuMind AI.
* Identify the necessary IAM permissions to call Textract.
* Configure permissions for the backend/worker to call Textract.
* Verify region availability for Amazon Textract.
* Prepare the environment for the OCR Worker in the next step.

---

#### The Role of Textract in DocuMind AI

In the **DocuMind AI** system, Amazon Textract is used to:

* Extract text from PDF documents or images.
* Convert unstructured documents into text content.
* Provide input data for the Gemini/OpenAI AI Gateway.
* Support summarization, classification, entity extraction, and AI Chat/RAG features.
* Reduce manual reading and data entry operations.

The processing flow in this step:

```text
Amazon SQS
  |
  v
Worker
  |
  v
Amazon S3 retrieves document
  |
  v
Amazon Textract OCR
  |
  v
PostgreSQL + Prisma stores OCR Result
```

![textract-permission](/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.1-configure-textract-permission/textract-permission.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the IAM Policy or IAM Role configured with permissions to invoke Amazon Textract, or draw a diagram representing Worker → S3 → Textract → PostgreSQL.
> Save the image at the path:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.1-configure-textract-permission/textract-permission.png`

---

#### Supported Document Formats

In this workshop, documents should be in formats supported by Amazon Textract:

```text
PDF
PNG
JPG
JPEG
```

Do not upload the following formats directly:

```text
DOCX
XLSX
PPTX
TXT
```

If you need to process DOCX, XLSX, or PPTX files, convert them to PDF before sending them to the OCR pipeline.

---

#### Verify Amazon Textract Region Support

Before configuring, make sure you are using a region that supports Amazon Textract.

Example region:

```env
AWS_REGION="ap-southeast-1"
```

If Textract is not available in your selected region, you may encounter errors like:

```text
UnknownEndpoint
```

or:

```text
Textract is not available in this region
```

Therefore, services in this workshop should be placed in the same region if possible:

* Amazon S3
* Amazon SQS
* Amazon Textract
* CloudWatch
* Secrets Manager
* Backend/Worker

---

#### Necessary IAM Permissions for Textract

The worker needs permissions to call Amazon Textract. The basic permissions are:

```text
textract:DetectDocumentText
textract:AnalyzeDocument
```

If the system processes documents asynchronously using Textract jobs, you might also need:

```text
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

For simple demos or small files in this workshop, `DetectDocumentText` or `AnalyzeDocument` is sufficient.

---

#### Necessary IAM Permissions for S3

Since Textract needs to read files from S3, the worker also needs access permissions to objects inside the S3 bucket:

```text
s3:GetObject
s3:ListBucket
```

If the backend also uploads files to S3, the backend requires additional permissions:

```text
s3:PutObject
```

---

#### Sample IAM Policy for the Textract OCR Worker

Reference policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindTextractAccess",
      "Effect": "Allow",
      "Action": [
        "textract:DetectDocumentText",
        "textract:AnalyzeDocument",
        "textract:StartDocumentTextDetection",
        "textract:GetDocumentTextDetection",
        "textract:StartDocumentAnalysis",
        "textract:GetDocumentAnalysis"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DocuMindS3ReadAccessForTextract",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::documind-document-storage",
        "arn:aws:s3:::documind-document-storage/*"
      ]
    }
  ]
}
```

> ⚠️ **Note:**
> If your bucket has a different name, replace `documind-document-storage` with your actual S3 bucket name.

---

#### Configure IAM Roles for Backend/Worker

In production or when deploying to EC2, use **IAM Roles** instead of hardcoding AWS credentials.

Recommended model:

```text
EC2 / Backend Server
  |
  v
IAM Role: DocMind-EC2-Role
  |
  v
Policy: S3 + SQS + Textract + CloudWatch + Secrets Manager
```

Do not use in production:

```env
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

The AWS SDK will automatically retrieve permissions from the IAM Role attached to the EC2 instance.

---

#### Related Environment Variables

The backend/worker requires the following environment variables:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
```

If a separate region is configured for Textract:

```env
AWS_TEXTRACT_REGION="ap-southeast-1"
```

If no separate variable is configured, `AWS_REGION` will be used by default.

---

#### Verify Permissions Using the AWS CLI

You can verify the current AWS identity by running:

```bash
aws sts get-caller-identity
```

If running on EC2 via an IAM Role, the output should resemble:

```text
arn:aws:sts::123456789012:assumed-role/DocMind-EC2-Role/...
```

You can quickly test S3 permissions:

```bash
aws s3 ls s3://documind-document-storage
```

If this command executes successfully, the worker has permissions to read the bucket.

---

#### Common Errors

| Error                           | Cause                                       | Solution                                                                 |
| ------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| `AccessDeniedException`         | Missing Textract permissions                | Add `textract:DetectDocumentText` or `textract:AnalyzeDocument` permissions|
| `AccessDenied` (S3)             | Worker lacks S3 read permissions            | Add `s3:GetObject` permissions for the bucket                             |
| `NoSuchBucket`                  | Incorrect bucket name                       | Check `AWS_S3_BUCKET_NAME`                                               |
| `InvalidS3ObjectException`      | Incorrect `s3Key` or file does not exist    | Verify the object key in S3                                               |
| `UnsupportedDocumentException`  | Invalid file format                         | Use PDF, PNG, JPG, or JPEG                                               |
| `UnknownEndpoint`               | Incorrect or unsupported region             | Check `AWS_REGION`                                                       |

---

#### Completion Checklist

You have completed this step when:

* The worker/backend has permissions to invoke Amazon Textract.
* The worker/backend has permissions to read S3 files.
* The AWS region is correctly configured.
* The S3 bucket name and `s3Key` are validated.
* AWS access keys are not hardcoded in production.
* The system is ready to create the OCR Worker in step 5.5.2.

---

#### Expected Outcome

Following this step, the DocuMind AI system has the necessary permissions for the worker to download documents from S3 and invoke Amazon Textract OCR. This lays the groundwork for converting original documents into text to support Gemini/OpenAI AI analysis.
