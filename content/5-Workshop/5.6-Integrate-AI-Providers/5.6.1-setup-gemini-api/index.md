---
title : "Configure Gemini API"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.6.1. </b> "
---

#### Overview

In this step, we will configure the **Google Gemini API** to integrate it into the **DocuMind AI** system.

The Gemini API is one of the two primary AI providers for the project, used to analyze document content after OCR processing with Amazon Textract. Once the worker extracts text from a document, the OCR content is sent to Gemini to perform tasks such as document summarization, classification, key entity extraction, generating structured JSON, and supporting AI Chat/RAG.

Integrating Gemini gives the system a highly flexible AI provider that can act as either the primary provider or a backup provider when OpenAI encounters errors, timeouts, or quota limits.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create a Gemini API Key.
* Add the Gemini API Key to the `.env` file.
* Install the necessary SDK for the backend.
* Create a Gemini Service in the backend.
* Test that Gemini can successfully analyze OCR text.
* Prepare Gemini for integration into the AI Gateway in subsequent steps.

---

#### The Role of Gemini API in DocuMind AI

In the **DocuMind AI** system, the Gemini API is used to:

* Summarize document content.
* Classify documents (e.g., invoices, contracts, forms, or reports).
* Extract key information from OCR text.
* Generate structured JSON outputs.
* Support AI Chat with documents.
* Act as the primary or fallback provider within the AI Gateway.
* Increase flexibility by supporting multiple AI models.

Gemini processing flow in the pipeline:

```text id="9wlf4b"
OCR Text from Amazon Textract
  |
  v
Gemini Service
  |
  v
Gemini API
  |
  v
AI Analysis Result
  |
  v
PostgreSQL + Prisma
```

![setup-gemini-api](/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.1-setup-gemini-api/setup-gemini-api.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the API Key creation page in Google AI Studio, or draw a diagram representing OCR Text → Gemini Service → Gemini API → AI Analysis Result.
> Save the image at the path:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.1-setup-gemini-api/setup-gemini-api.png`

---

#### Step 1: Access Google AI Studio

Open your browser and navigate to:

```text id="3erwdv"
https://aistudio.google.com/app/apikey
```

Log in with your Google account.

Then select:

```text id="fk584w"
Create API Key
```

If you have multiple Google Cloud Projects, select the project dedicated to DocuMind AI.

---

#### Step 2: Create a Gemini API Key

After selecting the project, generate the key.

You will receive an API key in the following format:

```text id="y7o3ci"
AIzaSyxxxxxxxxxxxxxxxxxxxxxxxx
```

Copy this API key to add to the backend environment file.

> ⚠️ **Security Warning:**
> Do not share your Gemini API Key with anyone. Do not commit your API key to GitHub. Only store the key in your local `.env` file, or in AWS Secrets Manager for production deployments.

---

#### Step 3: Configure Gemini in the `.env` File

Add the following variables to the backend `.env` file:

```env id="fu30bb"
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

If the system uses Gemini as the default AI provider, add:

```env id="xqvccg"
AI_PROVIDER="gemini"
```

If Gemini is only used as a backup provider, you can set:

```env id="8302w9"
AI_PROVIDER="openai"
```

and the AI Gateway will fallback to Gemini when OpenAI fails.

---

#### Step 4: Install Gemini SDK

In the backend directory, install the package:

```bash id="h2y5jv"
npm install @google/generative-ai
```

If using TypeScript, ensure that the project is configured with suitable TypeScript settings.

---

#### Step 5: Create Gemini Service

Create the file:

```text id="m832re"
src/services/ai/gemini.service.ts
```

Example basic service implementation:

```ts id="v52pvd"
import { GoogleGenerativeAI } from "@google/generative-ai";
 
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || "");
 
const model = genAI.getGenerativeModel({
  model: process.env.GEMINI_MODEL || "gemini-1.5-flash"
});
 
export async function analyzeWithGemini(ocrText: string) {
  const prompt = `
You are DocuMind AI, a document intelligence assistant.
 
Analyze the OCR text below and return JSON only.
 
Rules:
- Do not return markdown.
- Do not wrap the response in code block.
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
`;
 
  const result = await model.generateContent(prompt);
  const text = result.response.text();
 
  return JSON.parse(text);
}
```

In a production project, you should add error handling for JSON parsing, timeouts, and rate limits.

---

#### Step 6: Standardize Output Format

The Gemini Service should return data in a standardized format so that the AI Gateway can consume it interchangeably with OpenAI.

Example output:

```json id="sda721"
{
  "provider": "gemini",
  "model": "gemini-1.5-flash",
  "documentType": "Invoice",
  "summary": "This document is an invoice for cloud service usage.",
  "entities": [
    {
      "type": "vendor",
      "value": "AWS"
    }
  ],
  "keywords": ["invoice", "cloud", "payment"],
  "importantInformation": [
    "Invoice date: 2026-06-10",
    "Total amount: 120 USD"
  ],
  "confidence": 0.86
}
```

---

#### Step 7: Create a Test API Endpoint for Gemini

Implement a temporary test route to verify that Gemini is functioning correctly.

Example endpoint:

```text id="bd5fkm"
POST /api/ai/test-gemini
```

Request body:

```json id="e983me"
{
  "text": "Invoice No: INV-001. Vendor: AWS. Total: 120 USD."
}
```

Expected response:

```json id="s2skhw"
{
  "success": true,
  "provider": "gemini",
  "model": "gemini-1.5-flash",
  "result": {
    "documentType": "Invoice",
    "summary": "The document is an invoice from AWS.",
    "confidence": 0.86
  }
}
```

---

#### Step 8: Verify Using Postman

Open Postman and create a request:

```text id="qcfmhh"
Method: POST
URL: http://localhost:3000/api/ai/test-gemini
```

JSON body:

```json id="br299p"
{
  "text": "Invoice No INV-2026-001. Customer: Tuan Quang. Total amount: 2,000,000 VND."
}
```

The expected outcome is that Gemini returns a JSON containing the document analysis.

---

#### Step 9: Troubleshoot Common Errors

| Error                    | Cause                                        | Solution                             |
| ------------------------ | -------------------------------------------- | ------------------------------------ |
| `API key not valid`      | Incorrect Gemini API Key                     | Regenerate key and update `.env`     |
| `403 Permission denied`  | API key has no permissions or wrong project | Check project in Google AI Studio    |
| `429 Resource exhausted` | Quota exceeded                               | Wait for quota reset or switch provider|
| `JSON.parse error`       | Gemini returned markdown or text outside JSON| Tighten prompts and add JSON cleaners|
| `model not found`        | Incorrect model name                         | Verify the `GEMINI_MODEL` variable   |
| Timeout                  | AI response too slow                         | Add timeouts and fallback to OpenAI  |

---

#### Completion Checklist

You have completed this step when:

* The Gemini API Key is created.
* The API key has been added to `.env`.
* The `@google/generative-ai` package is installed in the backend.
* The Gemini Service is created.
* The test endpoint for Gemini is functional.
* Gemini returns structured JSON outputs.
* The backend does not expose the API key to the frontend.
* Gemini is ready for integration into the AI Gateway.

---

#### Expected Outcome

Following this step, DocuMind AI has successfully integrated the Gemini API as a backend service. The system can send OCR text to Gemini for document analysis and is prepared to integrate Gemini alongside OpenAI into the AI Gateway in subsequent steps.
