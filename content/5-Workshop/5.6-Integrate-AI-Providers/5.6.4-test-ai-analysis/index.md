---
title : "Test AI Analysis"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.6.4. </b> "
---

#### Overview

After configuring the Gemini API, OpenAI ChatGPT API, and the AI Gateway, the next step is to test the **AI Analysis** functionality in the **DocuMind AI** system.

In this step, we will verify sending OCR text to the AI Gateway, selecting either Gemini or OpenAI as the provider, receiving the structured document analysis, saving the results in **PostgreSQL + Prisma**, and updating the document status.

This step confirms that the system is capable of converting raw documents into OCR text, and then into structured AI analysis results.

---

#### Objectives of This Step

Upon completing this step, you will:

* Verify that the Gemini API is operational in the backend.
* Verify that the OpenAI ChatGPT API is operational in the backend.
* Confirm that the AI Gateway selects the correct provider.
* Test the fallback mechanism between OpenAI and Gemini.
* Ensure the AI response returns valid JSON.
* Verify that the AI Analysis is successfully saved in PostgreSQL.
* Confirm that the document status is updated to `COMPLETED`.

---

#### AI Analysis Verification Flow

The verification flow in DocuMind AI:

```text
OCR Text
  |
  v
AI Gateway
  |
  |----------------------|
  |                      |
  v                      v
OpenAI Service       Gemini Service
  |                      |
  |----------------------|
           |
           v
AI Analysis Result
           |
           v
PostgreSQL + Prisma
           |
           v
Document status = COMPLETED
```

![test-ai-analysis](/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.4-test-ai-analysis/test-ai-analysis.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the Postman screen when testing AI Analysis successfully, the backend logs displaying the selected provider, or the AIAnalysis data in PostgreSQL.
> Save the image at the path:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.4-test-ai-analysis/test-ai-analysis.png`

---

#### Prerequisites Before Testing

Before testing AI Analysis, make sure that:

* You have a Gemini API Key.
* You have an OpenAI API Key.
* The Gemini Service is created in the backend.
* The OpenAI Service is created in the backend.
* The AI Gateway Service is created in the backend.
* OCR text exists in the database.
* PostgreSQL and Prisma are active.
* The `.env` file is configured with the correct AI provider variables.

Example `.env`:

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

---

#### Step 1: Prepare Test OCR Text

You can use OCR text retrieved from Amazon Textract or enter sample text.

Example OCR text:

```text
Invoice No: INV-2026-001
Vendor: AWS
Customer: Tuan Quang
Invoice Date: 2026-06-10
Total Amount: 120 USD
Payment Status: Unpaid
```

---

#### Step 2: Test AI Analysis with OpenAI

Create a request in Postman:

```text
Method: POST
URL: http://localhost:3000/api/ai/analyze
```

JSON Body:

```json
{
  "text": "Invoice No: INV-2026-001. Vendor: AWS. Customer: Tuan Quang. Total Amount: 120 USD.",
  "provider": "openai"
}
```

Expected response:

```json
{
  "success": true,
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "result": {
    "documentType": "Invoice",
    "summary": "This document is an invoice from AWS for Tuan Quang.",
    "entities": [
      {
        "type": "invoiceNumber",
        "value": "INV-2026-001"
      },
      {
        "type": "vendor",
        "value": "AWS"
      },
      {
        "type": "customer",
        "value": "Tuan Quang"
      },
      {
        "type": "totalAmount",
        "value": "120 USD"
      }
    ],
    "keywords": ["invoice", "AWS", "payment"],
    "importantInformation": [
      "Invoice number: INV-2026-001",
      "Total amount: 120 USD"
    ],
    "confidence": 0.9
  }
}
```

---

#### Step 3: Test AI Analysis with Gemini

Create a similar request but change the provider:

```json
{
  "text": "Contract Agreement between Company A and Company B. Effective date: 2026-06-10. Term: 12 months.",
  "provider": "gemini"
}
```

Expected response:

```json
{
  "success": true,
  "provider": "gemini",
  "model": "gemini-1.5-flash",
  "result": {
    "documentType": "Contract",
    "summary": "This document is a contract agreement between Company A and Company B.",
    "entities": [
      {
        "type": "effectiveDate",
        "value": "2026-06-10"
      },
      {
        "type": "term",
        "value": "12 months"
      }
    ],
    "keywords": ["contract", "agreement", "effective date"],
    "importantInformation": [
      "Effective date: 2026-06-10",
      "Term: 12 months"
    ],
    "confidence": 0.85
  }
}
```

---

#### Step 4: Test Analysis Using `documentId`

If the system has already stored the OCR text in the database, you can test by `documentId`.

Example endpoint:

```text
POST /api/documents/:id/analyze
```

Request:

```json
{
  "provider": "openai"
}
```

The backend will:

1. Find the document by `id`.
2. Retrieve the OCR text from the `OCRResult` table.
3. Send the OCR text to the AI Gateway.
4. Save the results in `AIAnalysis`.
5. Update the document status to `COMPLETED`.

Expected response:

```json
{
  "success": true,
  "message": "AI analysis completed successfully",
  "documentId": "doc_001",
  "provider": "openai",
  "status": "COMPLETED"
}
```

---

#### Step 5: Verify Data in PostgreSQL

Using Prisma Studio:

```bash
npx prisma studio
```

Verify the table:

```text
AIAnalysis
```

Expected data:

```text
documentId: doc_001
provider: openai
model: gpt-4.1-mini
summary: This document is an invoice...
documentType: Invoice
confidence: 0.9
createdAt: 2026-06-10
```

Verify the table:

```text
Document
```

Expected status:

```text
COMPLETED
```

---

#### Step 6: Test Provider Fallback

To test fallback capabilities, temporarily input an incorrect API key for the primary provider.

For example:

```env
AI_PROVIDER="openai"
OPENAI_API_KEY="wrong-key"
```

Then invoke the analyze API again.

Expected results:

* OpenAI fails.
* AI Gateway logs primary provider failure.
* The system switches to Gemini.
* Response returns `gemini` as the provider.

Expected logs:

```text
[AI_PROVIDER_SELECTED] openai
[AI_PROVIDER_PRIMARY_FAILED] openai
[AI_PROVIDER_FALLBACK_USED] gemini
[AI_ANALYSIS_COMPLETED]
```

---

#### Step 7: Test Empty OCR Text Handling

Send a request with empty text:

```json
{
  "text": "",
  "provider": "openai"
}
```

Expected response:

```json
{
  "success": false,
  "message": "OCR text is empty. Cannot analyze document."
}
```

Do not call Gemini/OpenAI if the OCR text is empty, as it wastes quota and returns meaningless results.

---

#### Step 8: Test Non-JSON Response Handling

In some cases, AI providers may return markdown or plain text instead of valid JSON. The system must handle these exceptions gracefully.

Expected error response:

```json
{
  "success": false,
  "message": "AI response is not valid JSON."
}
```

Mitigation strategies:

* Tighten prompts requesting JSON only.
* Use `response_format` with OpenAI when possible.
* Clean markdown syntax before calling `JSON.parse`.
* Implement a single retry if the response is invalid.

---

#### Step 9: Check Backend Logs

Expected logs:

```text
[AI_ANALYSIS_STARTED]
[AI_PROVIDER_SELECTED]
[OPENAI_REQUEST_STARTED]
[OPENAI_RESPONSE_RECEIVED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[DOCUMENT_STATUS_UPDATED_TO_COMPLETED]
```

Or if Gemini is called:

```text
[GEMINI_REQUEST_STARTED]
[GEMINI_RESPONSE_RECEIVED]
```

Do not log API keys or sensitive document details.

---

#### Common Errors

| Error                    | Cause                      | Solution                        |
| ------------------------ | -------------------------- | ------------------------------- |
| `401 Unauthorized`       | Incorrect OpenAI API key   | Verify `OPENAI_API_KEY`         |
| `API key not valid`      | Incorrect Gemini API key   | Verify `GEMINI_API_KEY`         |
| `429 Rate limit`         | Quota exceeded             | Implement retries or fallback   |
| `JSON.parse error`       | Response not valid JSON    | Tighten prompt rules, clean text|
| `OCR text is empty`      | Missing OCR results        | Check the Textract OCR step     |
| `Provider not supported` | Unsupported provider name  | Use only `openai` or `gemini`   |
| Timeout                  | AI response too slow       | Configure timeouts and fallback |

---

#### Completion Checklist

You have completed this step when:

* OpenAI Analysis is successfully tested.
* Gemini Analysis is successfully tested.
* The AI Gateway selects the correct provider.
* Fallback functions correctly when the primary provider fails.
* The AI response is valid JSON.
* AIAnalysis results are stored in PostgreSQL.
* The document status is updated to `COMPLETED`.
* Backend logs clearly indicate the selected provider, model, and status.
* The system is ready to connect the AI results to the Dashboard.

---

#### Expected Outcome

Following this step, DocuMind AI has completed the basic AI Analysis component. The system can retrieve OCR text, analyze it via OpenAI or Gemini, fall back when necessary, store results in PostgreSQL, and make the results available for display on the Dashboard, AI Chat/RAG, and Enterprise Sandbox.
