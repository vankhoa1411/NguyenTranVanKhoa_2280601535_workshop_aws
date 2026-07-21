---
title : "Test OCR Processing"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.5.3. </b> "
---

#### Overview

After configuring Textract permissions and creating the OCR Worker, the next step is to test the entire **OCR Processing** workflow in the **DocuMind AI** system.

In this step, we will verify the end-to-end flow: pushing a message to **Amazon SQS**, the worker retrieving the document from **Amazon S3**, invoking **Amazon Textract**, saving the OCR results in **PostgreSQL + Prisma**, and updating the document status.

This is a critical step to confirm that the system is capable of converting raw documents into text. This OCR text will serve as the input data for the **Gemini/OpenAI AI Gateway** in the next section.

---

#### Objectives of This Step

Upon completing this step, you will:

* Verify that the worker retrieves messages from SQS.
* Confirm that the worker downloads the correct file from S3.
* Ensure Amazon Textract OCR functions correctly.
* Verify that the OCR text is stored in PostgreSQL.
* Confirm that the document status is updated accordingly.
* Test error handling for unsupported formats or missing files.
* Prepare the OCR data for the AI Analysis section.

---

#### OCR Verification Flow

The OCR flow in DocuMind AI:

```text
Amazon SQS
  |
  v
OCR Worker
  |
  v
Amazon S3
  |
  v
Amazon Textract
  |
  v
PostgreSQL + Prisma
  |
  v
Document status = OCR_COMPLETED
```

![test-ocr-processing](/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.3-test-ocr-processing/test-ocr-processing.png)

> ⚠️ **Screenshot Suggestion:**
> Capture screenshots of the worker logs displaying successful OCR execution, the OCR results saved in the database, or the document status transitioning to `OCR_COMPLETED`.
> Save the image at the path:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.3-test-ocr-processing/test-ocr-processing.png`

---

#### Prerequisites Before Testing

Before testing OCR, ensure that:

* The S3 bucket is created.
* A PDF/PNG/JPG/JPEG file has been uploaded to S3.
* The SQS processing queue is created.
* A message containing `documentId`, `s3Bucket`, and `s3Key` has been sent to SQS.
* The worker has S3 read permissions.
* The worker has permissions to invoke Amazon Textract.
* PostgreSQL and Prisma are active and running.
* The worker is configured with the correct `.env` variables.

Example `.env`:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"

DATABASE_URL="postgresql://username:password@localhost:5432/docmind"
```

---

#### Step 1: Prepare a Test Document

Use a valid document format:

```text
invoice-demo.pdf
receipt-demo.jpg
contract-demo.pdf
```

Supported formats:

```text
PDF
PNG
JPG
JPEG
```

Do not use directly:

```text
DOCX
XLSX
PPTX
```

as Amazon Textract may throw format-not-supported errors.

---

#### Step 2: Upload the Document to S3

Upload the document using the API verified in section 5.3.3:

```text
POST http://localhost:3000/api/documents/upload
```

Expected output:

```json
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED"
  }
}
```

---

#### Step 3: Verify the SQS Message

Go to the AWS Console:

```text
Amazon SQS
```

Select the queue:

```text
docmind-document-processing-queue
```

Check if `Available messages` indicates that a new message has arrived.

Expected message payload:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

---

#### Step 4: Start the OCR Worker

Open a terminal in your backend directory and run the worker.

For example:

```bash
npm run worker
```

Or if the worker has a dedicated script:

```bash
npm run worker:document
```

Expected logs:

```text
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
```

---

#### Step 5: Verify Successful Textract OCR

When Textract completes processing successfully, the worker should log:

```text
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[DOCUMENT_STATUS_UPDATED_TO_OCR_COMPLETED]
```

Upon successful execution, the message must be deleted from SQS:

```text
[SQS_MESSAGE_DELETED]
```

---

#### Step 6: Verify the OCR Result in PostgreSQL

Using Prisma Studio:

```bash
npx prisma studio
```

Open the table containing OCR results, for example:

```text
OCRResult
```

Verify that the new record exists:

```text
documentId: doc_001
provider: AWS_TEXTRACT
rawText: Extracted OCR content from document
createdAt: 2026-06-10
```

If the OCR text is empty, check the document file quality or format.

---

#### Step 7: Check Document Status

In the `Document` table, the document status after OCR should be:

```text
OCR_COMPLETED
```

Or if the system immediately forwards the data to AI analysis:

```text
AI_ANALYZING
```

If an error occurred, the status should be updated to:

```text
FAILED
```

---

#### Step 8: Test Missing File Error Handling

Send an SQS message with a non-existent `s3Key`:

```json
{
  "documentId": "doc_error_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_error_001/not-found.pdf",
  "action": "PROCESS_DOCUMENT"
}
```

The worker should log the failure:

```text
[DOCUMENT_PROCESSING_FAILED]
InvalidS3ObjectException
```

or:

```text
NoSuchKey
```

If a matching document record exists in the database, its status should update to `FAILED`.

---

#### Step 9: Test Unsupported File Type Handling

Try uploading an unsupported file format such as:

```text
test.docx
```

Expected outcomes:

* The backend blocks the file immediately at the upload stage.

Or if the file bypasses upload validation, Textract will report:

```text
UnsupportedDocumentException
```

The best practice is for the backend to block unsupported formats before uploading to S3 or before queueing SQS messages.

---

#### Step 10: Verify CloudWatch Logs (If Integrated)

If the worker sends logs to CloudWatch, check the console:

```text
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Look for the logs:

```text
TEXTRACT_OCR_STARTED
TEXTRACT_OCR_COMPLETED
TEXTRACT_OCR_FAILED
```

Avoid logging the full document content if it contains sensitive information.

---

#### Common Errors

| Error                           | Cause                                       | Solution                                     |
| ------------------------------- | ------------------------------------------- | -------------------------------------------- |
| `AccessDeniedException`         | Missing Textract permissions                | Add Textract permissions to the IAM Policy    |
| `InvalidS3ObjectException`      | Textract cannot read the S3 object          | Verify bucket, key, region, and S3 permissions|
| `NoSuchKey`                     | Incorrect `s3Key`                           | Verify the object exists in S3               |
| `UnsupportedDocumentException`  | Unsupported file type                       | Use PDF, PNG, JPG, or JPEG                   |
| Duplicate message retrieval     | Message not deleted upon success            | Call `DeleteMessage` when processing succeeds|
| OCR text is empty               | Document scan is blurry or contains no text | Use a higher quality file                    |
| `QueueDoesNotExist`             | Incorrect Queue URL or region               | Check `AWS_SQS_QUEUE_URL` and `AWS_REGION`   |

---

#### Completion Checklist

You have completed this step when:

* A valid document upload is successful.
* The processing message is queued in SQS.
* The worker retrieves the message from SQS.
* The worker successfully invokes Textract.
* OCR results are saved in PostgreSQL.
* The document status is updated to `OCR_COMPLETED`.
* The message is successfully deleted from SQS.
* Invalid file types or incorrect S3 keys are handled gracefully.
* The system is ready to proceed to section 5.6 - Integrate AI Providers.

---

#### Expected Outcome

Following this step, DocuMind AI has completed the OCR processing component. The system can successfully download documents from S3, extract their content using Amazon Textract, store the OCR results in PostgreSQL, and prepare the text data to be sent to Gemini/OpenAI for AI Analysis in the next section.
