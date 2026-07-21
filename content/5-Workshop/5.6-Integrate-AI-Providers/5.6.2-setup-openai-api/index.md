---
title : "Configure OpenAI ChatGPT API"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.6.2. </b> "
---

#### Overview

In this step, we will configure the **OpenAI ChatGPT API** to integrate it into the **DocuMind AI** system.

The OpenAI ChatGPT API is the second primary AI provider for the project, operating in parallel with the Google Gemini API. After Amazon Textract extracts the text content from a document, the backend can send the OCR text to OpenAI to perform summarization, classification, key entity extraction, generating structured JSON, and supporting AI Chat/RAG.

Integrating OpenAI provides the system with a powerful AI model option, while increasing system reliability via fallback capabilities between OpenAI and Gemini.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create or prepare an OpenAI API Key.
* Add the OpenAI API Key to the `.env` file.
* Install the OpenAI SDK for the backend.
* Create an OpenAI Service.
* Test that OpenAI can successfully analyze OCR text.
* Prepare OpenAI for integration into the AI Gateway.

---

#### The Role of OpenAI in DocuMind AI

In the **DocuMind AI** system, the OpenAI ChatGPT API is used to:

* Analyze OCR text extracted from documents.
* Summarize documents.
* Classify documents.
* Extract key entities and important information.
* Support AI Chat/RAG with documents.
* Support semantic search if using embeddings.
* Act as the primary or fallback provider within the AI Gateway.

OpenAI processing flow in the pipeline:

```text id="8uejwk"
OCR Text from Amazon Textract
  |
  v
OpenAI Service
  |
  v
OpenAI ChatGPT API
  |
  v
AI Analysis Result
  |
  v
PostgreSQL + Prisma
```

![setup-openai-api](/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.2-setup-openai-api/setup-openai-api.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the OpenAI API Key creation page, or draw a diagram representing OCR Text → OpenAI Service → OpenAI ChatGPT API → AI Analysis Result.
> Save the image at the path:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.2-setup-openai-api/setup-openai-api.png`

---

#### Step 1: Prepare OpenAI API Key

Access the OpenAI API key management page and generate a new API key for the project.

After creating it, copy the API key and save it securely.

The API key typically matches the following format:

```text id="qkhzji"
sk-xxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ **Security Warning:**
> Do not share your OpenAI API Key. Do not expose the API key to the frontend. Do not commit your API key to GitHub. Only store it in your local `.env` file, or in AWS Secrets Manager for production deployments.

---

#### Step 2: Configure OpenAI in the `.env` File

Add the following environment variables to the backend:

```env id="hq21nr"
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
```

If you want OpenAI as the default AI provider, set:

```env id="ea6p0j"
AI_PROVIDER="openai"
```

If you prefer Gemini as the default and OpenAI as the fallback, you can set:

```env id="7xtu9q"
AI_PROVIDER="gemini"
```

---

#### Step 3: Install OpenAI SDK

In the backend directory, run:

```bash id="fbg9c9"
npm install openai
```

---

#### Step 4: Create OpenAI Service

Create the file:

```text id="l54hpm"
src/services/ai/openai.service.ts
```

Example basic service implementation:

```ts id="46t3xi"
import OpenAI from "openai";
 
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});
 
export async function analyzeWithOpenAI(ocrText: string) {
  const response = await openai.chat.completions.create({
    model: process.env.OPENAI_MODEL || "gpt-4.1-mini",
    messages: [
      {
        role: "system",
        content: "You are DocuMind AI, a document intelligence assistant. Return JSON only."
      },
      {
        role: "user",
        content: `
Analyze the OCR text below.
 
Rules:
- Return JSON only.
- Do not use markdown.
- If a field is not found, return null.
- Do not invent information.
 
OCR Text:
${ocrText}
 
Return JSON format:
{
  "documentType": "",
  "summary": "",
  "entities": [],
  "keywords": [],
  "importantInformation": [],
  "confidence": 0
}
`
      }
    ],
    response_format: { type: "json_object" }
  });
 
  const content = response.choices[0]?.message?.content || "{}";
  return JSON.parse(content);
}
```

---

#### Step 5: Standardize Output Format

The OpenAI Service should return data in the same format as Gemini to ensure consistency in the AI Gateway.

Example output:

```json id="b9jl3t"
{
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "documentType": "Invoice",
  "summary": "This document is an invoice for cloud usage.",
  "entities": [
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

---

#### Step 6: Create a Test API Endpoint for OpenAI

Implement a temporary test route:

```text id="2ues21"
POST /api/ai/test-openai
```

Request body:

```json id="efpaxv"
{
  "text": "Invoice No: INV-001. Vendor: AWS. Total: 120 USD."
}
```

Expected response:

```json id="0d2dqz"
{
  "success": true,
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "result": {
    "documentType": "Invoice",
    "summary": "The document is an invoice from AWS.",
    "confidence": 0.9
  }
}
```

---

#### Step 7: Verify Using Postman

Open Postman and create a request:

```text id="wihj7l"
Method: POST
URL: http://localhost:3000/api/ai/test-openai
```

JSON body:

```json id="gkmdac"
{
  "text": "Contract Agreement between Company A and Company B. Effective date: 2026-06-10."
}
```

If the API key is correct and the model is functional, the backend will return the structured JSON analysis result.

---

#### Step 8: Integrate Embeddings for AI Chat/RAG

If the system supports **AI Chat/RAG** or **Semantic Search**, OpenAI can generate vectors using an embedding model.

Example environment variable:

```env id="f2h39u"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
```

Example embedding generation function:

```ts id="0qbnb5"
export async function generateOpenAIEmbedding(text: string) {
  const response = await openai.embeddings.create({
    model: process.env.OPENAI_EMBEDDING_MODEL || "text-embedding-3-small",
    input: text
  });
 
  return response.data[0].embedding;
}
```

These embeddings can be used to:

* Save vector embeddings for document chunks.
* Retrieve the most relevant document sections based on a user query.
* Power AI Chat/RAG and Semantic Search capabilities.

---

#### Step 9: Troubleshoot Common Errors

| Error                | Cause                                    | Solution                                          |
| -------------------- | ---------------------------------------- | ------------------------------------------------- |
| `401 Unauthorized`   | Incorrect or expired API key             | Regenerate the OpenAI API Key                     |
| `429 Rate limit`     | Quota exceeded or high request frequency  | Reduce request frequency, retry, or fallback to Gemini|
| `model_not_found`    | Incorrect model name                     | Verify the `OPENAI_MODEL` configuration           |
| `insufficient_quota` | Account out of credit/billing inactive   | Check OpenAI billing settings                     |
| `JSON.parse error`   | Response is not valid JSON               | Use `response_format` and tighten prompt rules    |
| Timeout              | Response too slow                        | Add timeouts and fallback to Gemini               |

---

#### Completion Checklist

You have completed this step when:

* You have obtained an OpenAI API Key.
* The API key has been added to `.env`.
* The `openai` package is installed in the backend.
* The OpenAI Service is created.
* The test endpoint for OpenAI is functional.
* OpenAI returns structured JSON outputs.
* The backend does not expose the API key to the frontend.
* OpenAI is ready for integration into the AI Gateway.

---

#### Expected Outcome

Following this step, DocuMind AI has successfully integrated the OpenAI ChatGPT API as a backend service. The system can send OCR text to OpenAI for document analysis, and is prepared to combine OpenAI with Gemini within the AI Gateway to create a flexible, multi-model architecture.
