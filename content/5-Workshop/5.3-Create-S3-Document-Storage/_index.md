---
title : "Create S3 Document Storage"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

#### Create Amazon S3 Document Storage

In this section, we will set up **Amazon S3** as the original document storage for the **DocuMind AI** system. This is the first step in the system's document processing pipeline.

When users upload documents such as PDF, PNG, JPG, or JPEG from the web interface, the backend will receive the file, check the format, and then upload the document to the **Amazon S3 Bucket**. Upon a successful upload, the backend will store the document's metadata in **PostgreSQL via Prisma**, including the file name, file type, size, `s3Bucket`, `s3Key`, and initial processing status.

Amazon S3 allows the system to decouple file storage from the backend server, ensuring that documents are stored stably, are easy to scale, and can be used by the worker in subsequent steps such as OCR with Amazon Textract and analysis with Gemini/OpenAI.

---

#### The Role of Amazon S3 in DocuMind AI

In the **DocuMind AI** project, Amazon S3 is used to:

* Store original documents uploaded by users.
* Support document formats for OCR like PDF, PNG, JPG, and JPEG.
* Provide the `s3Key` for the backend or worker to retrieve documents.
* Decouple file storage from the backend API.
* Help the system scale easily as the number of documents grows.
* Protect documents using a private bucket, Block Public Access, encryption, and IAM Policies.

Documents on S3 are not directly exposed to the public internet. Users should only access documents through the backend API after the system verifies valid permissions.

---

#### Document Storage Architecture

The document storage flow in this section:

```text
User
  |
  v
React Frontend
  |
  v
Backend API Node.js/Express
  |
  v
Amazon S3 Bucket
  |
  v
PostgreSQL + Prisma storing metadata
```

![s3-document-storage](/images/5-Workshop/5.3-Create-S3-Document-Storage/s3-document-storage.png)

> ⚠️ **Screenshot Suggestion:**
> Draw or capture a diagram illustrating the document upload flow containing these components: User, React Frontend, Node.js/Express Backend API, Amazon S3 Bucket, and PostgreSQL + Prisma.
> Save the image at the path:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/s3-document-storage.png`

---

#### Detailed Content

In section **5.3 - Create S3 Document Storage**, we will perform the following steps:

* [Create S3 Bucket](5.3.1-create-s3-bucket/)
* [Configure CORS and S3 Security](5.3.2-configure-cors-and-security/)
* [Test Document Upload to S3](5.3.3-test-s3-upload/)

---

#### Expected Outcomes

Upon completion of this section, the system should achieve the following results:

* An S3 bucket has been created to store DocuMind AI documents.
* The bucket is configured as private with Block Public Access enabled.
* The bucket has encryption enabled to protect documents.
* CORS is configured appropriately for development or production environments.
* The backend is capable of uploading files to S3.
* Uploaded files have clear `s3Key` values.
* Document metadata is saved into PostgreSQL via Prisma.
* The system is ready to move to the next part: sending document processing jobs to Amazon SQS.

---

#### Connection to the Next Step

After the document is successfully uploaded to Amazon S3, the backend will continue by sending a message containing document information to **Amazon SQS** in section **5.4 - Create SQS Processing Queue**. This message will inform the worker which document needs to be processed for OCR and AI analysis.

The subsequent processing flow:

```text
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
```

Thus, section 5.3 serves as the input storage layer for the entire DocuMind AI document processing pipeline.
