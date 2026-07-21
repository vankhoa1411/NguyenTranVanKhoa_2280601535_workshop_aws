---
title : "Integrate AI Providers"
date : 2026-06-10
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

#### Integrate Gemini and OpenAI AI Providers

In this section, we will integrate two primary AI providers for the **DocuMind AI** system, namely **Google Gemini API** and **OpenAI ChatGPT API**.

After a document is processed by **Amazon Textract**, the system obtains the extracted text. This text is sent to the **AI Gateway Service** to perform analysis tasks such as document summarization, classification, key entity extraction, generating structured JSON, and supporting contextual Q&A.

Integrating two AI providers increases the system's flexibility. DocuMind AI can select either Gemini or OpenAI as the primary provider, and use the other as a fallback option when the primary provider encounters errors, timeouts, or rate limits.

---

#### The Role of AI Providers in DocuMind AI

In the **DocuMind AI** project, Gemini and OpenAI are used to:

* Point out and analyze OCR text extracted from documents.
* Summarize documents.
* Classify documents (e.g., invoices, contracts, forms, or reports).
* Extract key entities (e.g., customer names, vendor details, dates, total amounts, or key clauses).
* Generate structured JSON data.
* Support AI Chat/RAG features to let users query their documents.
* Support Semantic Search and Enterprise Sandbox features.
* Increase reliability through multi-provider support and automatic fallback.

---

#### AI Provider Architecture

The AI processing flow in this section:

```text
OCR Text from Amazon Textract
  |
  v
AI Gateway Service
  |
  |--------------------------|
  |                          |
  v                          v
Google Gemini API        OpenAI ChatGPT API
  |                          |
  |--------------------------|
              |
              v
        AI Analysis Result
              |
              v
PostgreSQL + Prisma stores result
```

![ai-provider-flow](/static/images/5-Workshop/5.6-Integrate-AI-Providers/ai-provider-flow.png)

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the flow of OCR Text from Amazon Textract into the AI Gateway, which then selects Gemini or OpenAI to analyze the document and store the results in PostgreSQL via Prisma.
> Save the image at the path:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/ai-provider-flow.png`

---

#### AI Analysis Workflow

The AI Analysis workflow in DocuMind AI operates as follows:

1. The document is uploaded to Amazon S3.
2. The backend sends a processing job to Amazon SQS.
3. The worker retrieves the message from SQS.
4. The worker invokes Amazon Textract to perform OCR.
5. The OCR results are saved in PostgreSQL.
6. The worker or backend sends the OCR text to the AI Gateway.
7. The AI Gateway selects the provider based on configuration or the user's request.
8. Gemini or OpenAI analyzes the document content.
9. The AI Analysis results are saved in PostgreSQL.
10. The document status is updated to `COMPLETED`.
11. The user receives a notification and can view the results on the Dashboard.

Document statuses used in this section:

```text
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

---

#### Detailed Content

In section **5.6 - Integrate AI Providers**, we will perform the following steps:

* [Configure Gemini API](5.6.1-setup-gemini-api/)
* [Configure OpenAI ChatGPT API](5.6.2-setup-openai-api/)
* [Create AI Gateway](5.6.3-create-ai-gateway/)
* [Test AI Analysis](5.6.4-test-ai-analysis/)

---

#### AI Analysis Result

After the AI provider processes the OCR text, the system should save the results in the `AIAnalysis` table.

Example AI output:

```json
{
  "documentId": "doc_001",
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "documentType": "Invoice",
  "summary": "This document is an invoice for cloud service usage.",
  "entities": [
    {
      "type": "vendor",
      "value": "AWS"
    },
    {
      "type": "totalAmount",
      "value": "120 USD"
    }
  ],
  "keywords": ["invoice", "payment", "cloud"],
  "importantInformation": [
    "Invoice number: INV-001",
    "Total amount: 120 USD"
  ],
  "confidence": 0.9
}
```

These results will be displayed on the Dashboard, Document Details page, Enterprise Sandbox, or used as reference data for the AI Chat/RAG component.

---

#### Related Environment Variables

The backend requires environment variables for Gemini and OpenAI:

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

To set Gemini as the default provider:

```env
AI_PROVIDER="gemini"
```

To set OpenAI as the default provider:

```env
AI_PROVIDER="openai"
```

---

#### API Key Security Rules

When integrating AI providers, ensure the following guidelines are followed:

* Do not expose Gemini/OpenAI API keys to the frontend.
* Do not commit your actual `.env` file to GitHub.
* Only the backend should call the Gemini/OpenAI APIs directly.
* In production, API keys should be stored in AWS Secrets Manager.
* Do not log API keys in the console or CloudWatch.
* Do not log the full document content if it contains sensitive information.

---

#### Logging During AI Analysis

The system should write logs at key milestones:

```text
[AI_ANALYSIS_STARTED]
[AI_PROVIDER_SELECTED]
[GEMINI_REQUEST_STARTED]
[GEMINI_RESPONSE_RECEIVED]
[OPENAI_REQUEST_STARTED]
[OPENAI_RESPONSE_RECEIVED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[AI_PROVIDER_FALLBACK_USED]
[AI_ANALYSIS_FAILED]
```

If CloudWatch is integrated, these logs should be forwarded to the log group:

```text
/docmind/application
```

---

#### Common Errors

| Error                   | Cause                                    | Solution                                          |
| ----------------------- | ---------------------------------------- | ------------------------------------------------- |
| `API key not valid`     | Incorrect Gemini/OpenAI API key          | Verify `.env` or AWS Secrets Manager              |
| `401 Unauthorized`      | Incorrect OpenAI API key                 | Recreate the OpenAI key                           |
| `403 Permission denied` | Gemini key has incorrect project/rights  | Verify Google AI Studio                           |
| `429 Rate limit`        | Quota exceeded                           | Implement retries or fallback provider            |
| `JSON.parse error`      | AI response is not valid JSON            | Adjust prompts, clean JSON, or use response format|
| Timeout                 | AI provider response too slow            | Add timeouts and implement fallback logic         |
| Empty OCR text          | Textract failed to extract text          | Verify OCR results before calling AI              |

---

#### Completion Checklist

You have completed this step when:

* The Gemini API is successfully configured and tested.
* The OpenAI ChatGPT API is successfully configured and tested.
* The AI Gateway can select the provider based on configuration.
* The system is capable of fallback between Gemini and OpenAI.
* OCR text is successfully analyzed into structured AI results.
* AI Analysis results are stored in PostgreSQL via Prisma.
* The document status is updated to `COMPLETED`.
* The Dashboard can display the analyzed document results.

---

#### Connection to the Next Step

Following the integration of AI Providers, the system is capable of processing documents from their raw format to structured AI analysis results. In the next section, we will configure the **PostgreSQL + Prisma Database** to ensure that database tables such as `User`, `Document`, `OCRResult`, `AIAnalysis`, `ChatHistory`, `Notification`, and `AuditLog` are correctly designed for the entire system.

Data flow after this section:

```text
S3
  |
  v
SQS
  |
  v
Textract OCR
  |
  v
Gemini / OpenAI
  |
  v
PostgreSQL + Prisma
  |
  v
Dashboard / AI Chat / Notification
```

Thus, section 5.6 is responsible for building the core artificial intelligence layer for DocuMind AI.
