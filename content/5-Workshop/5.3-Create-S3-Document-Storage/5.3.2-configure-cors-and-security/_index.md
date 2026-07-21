---
title : "Configure CORS and S3 Security"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.3.2. </b> "
---

#### Overview

After creating the S3 bucket, the next step is to configure **CORS** and the necessary security settings for the bucket.

In the **DocuMind AI** project, user documents can be uploaded either via the backend or via pre-signed URLs. Therefore, S3 needs to be configured correctly so that the frontend/backend can upload valid files, while still ensuring that documents are not accidentally exposed to the public.

CORS helps the browser allow the frontend to make calls to S3 in case of direct uploads from the web. However, even with CORS configured, the bucket must remain private, and access permissions should be controlled via the backend or IAM Roles.

---

#### Objectives of This Step

Upon completing this step, you will:

* Configure CORS for the S3 bucket.
* Keep the bucket in a private state.
* Enable Block Public Access.
* Understand how to protect user documents.
* Prepare the IAM Policy for backend/worker access to S3.
* Determine allowed file formats for uploading.
* Avoid CORS errors when the frontend invokes file uploads.

---

#### What is CORS?

**CORS** stands for **Cross-Origin Resource Sharing**. It is a mechanism that allows a website on one domain to access resources from another domain.

For example:

```text
Frontend: http://localhost:5173
S3 Bucket: https://documind-document-storage.s3.ap-southeast-1.amazonaws.com
```

If the frontend uploads files directly to S3 without configuring CORS, the browser may report an error:

```text
Access to fetch at ... from origin ... has been blocked by CORS policy
```

If the backend receives the file and uploads it to S3, CORS errors with S3 are less likely to occur. However, CORS configuration remains useful if the system uses pre-signed URLs.

---

#### CORS and Security Architecture

Secure upload flow in DocuMind AI:

```text
React Frontend
  |
  v
Backend API checking user and file
  |
  v
Private Amazon S3 Bucket
  |
  v
PostgreSQL storing metadata
```

Or if using pre-signed URLs:

```text
React Frontend
  |
  v
Backend API generating pre-signed URL
  |
  v
Frontend uploading directly to S3
  |
  v
Backend storing metadata and sending SQS job
```

![s3-cors-security](/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.2-configure-cors-and-security/s3-cors-security.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the CORS configuration section in your S3 bucket or draw a diagram illustrating the private bucket, the backend controlling upload permissions, and the frontend calling the API.
> Save the image at the path:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.2-configure-cors-and-security/s3-cors-security.png`

---

#### Step 1: Open S3 Bucket

In the AWS Console:

```text
Amazon S3
```

Select the project bucket, for example:

```text
documind-document-storage
```

Then select the tab:

```text
Permissions
```

---

#### Step 2: Check Block Public Access

In the **Permissions** tab, find the section:

```text
Block public access
```

Ensure the status is:

```text
On
```

Or:

```text
Block all public access: Enabled
```

If it is disabled, turn on all Block Public Access options again.

Correct configuration:

```text
Block public access to buckets and objects granted through new access control lists (ACLs): On
Block public access to buckets and objects granted through any access control lists (ACLs): On
Block public access to buckets and objects granted through new public bucket policies: On
Block public and cross-account access to buckets and objects through any public bucket policies: On
```

---

#### Step 3: Check Bucket Policy

For DocuMind AI, the bucket should not have a public policy.

There should not be a policy like:

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::documind-document-storage/*"
}
```

The policy above would put files at risk of being made public.

Instead, S3 access permissions should be granted to the backend/worker via IAM Roles, or IAM Users for the local environment.

---

#### Step 4: Configure CORS

In the **Permissions** tab, scroll down to the section:

```text
Cross-origin resource sharing (CORS)
```

Click:

```text
Edit
```

Add the sample CORS configuration for the development environment:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": [
      "http://localhost:5173",
      "http://localhost:3000"
    ],
    "ExposeHeaders": ["ETag"]
  }
]
```

Then click:

```text
Save changes
```

---

#### CORS for Production Environment

When deploying to production, do not configure the origin too broadly.

Example for production:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": [
      "https://your-documind-domain.com"
    ],
    "ExposeHeaders": ["ETag"]
  }
]
```

Do not use:

```json
"AllowedOrigins": ["*"]
```

for the production environment if the system processes private documents.

---

#### Step 5: Configure File Upload Rules in Backend

In addition to S3 CORS, the backend still needs to validate file uploads.

Allowed formats should include:

```text
.pdf
.png
.jpg
.jpeg
```

Corresponding MIME types:

```text
application/pdf
image/png
image/jpeg
```

Do not allow:

```text
.exe
.sh
.bat
.js
.docx
.xlsx
.pptx
```

as Amazon Textract does not directly support office formats like DOCX, XLSX, or PPTX.

---

#### Step 6: Limit File Size

The backend should limit the upload file size to prevent users from uploading excessively large files that consume resources.

For example:

```text
Max file size: 10MB
```

Or if you need to OCR larger documents:

```text
Max file size: 20MB
```

When a file exceeds the limit, the backend should return a clear error:

```json
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

---

#### Step 7: Prepare IAM Policy for S3

The backend or worker requires access permissions to the bucket.

Minimum permissions usually required:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

Sample policy for reference:

```json
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

> ⚠️ **Note:**
> If your bucket has a different name, replace `documind-document-storage` with your actual bucket name.

---

#### Step 8: Do Not Hardcode AWS Credentials

In the local environment, you can use:

```bash
aws configure
```

for quick testing.

However, when deploying to production on EC2/backend server, you should use:

```text
IAM Role
```

Do not store the following variables in production `.env`:

```env
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Instead, the backend uses the AWS SDK default credentials chain to retrieve credentials automatically from the IAM Role attached to the EC2 instance.

---

#### Step 9: Verify CORS

After configuring CORS, test the upload from the frontend or Postman.

If a CORS error occurs, the browser usually reports:

```text
CORS policy: No 'Access-Control-Allow-Origin' header is present
```

Verification steps:

* Re-check `AllowedOrigins`.
* Ensure the frontend is running on the correct port.
* Ensure the upload method `PUT` or `POST` is listed in `AllowedMethods`.
* Ensure the request does not use blocked headers.
* If uploading via backend, check the backend Express CORS configuration instead of the S3 CORS configuration.

---

#### Completion Checklist

You have completed this step when:

* The bucket remains private.
* Block Public Access is enabled.
* The bucket has no public policy.
* CORS has allowed local origins or production domains.
* The backend has rules to validate file types.
* The backend has rules to limit file sizes.
* The IAM Policy for S3 has been prepared.
* AWS credentials are not hardcoded in production.

---

#### Expected Outcome

Following this step, the S3 bucket of DocuMind AI has been configured more securely and is ready for the document upload process. The system can avoid CORS errors when the frontend uploads files, while still ensuring that user documents are not publicly exposed.
