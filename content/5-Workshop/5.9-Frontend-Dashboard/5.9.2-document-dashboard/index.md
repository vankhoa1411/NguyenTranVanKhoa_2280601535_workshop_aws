---
title : "Document Dashboard"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.9.2. </b> "
---

#### Overview

In this step, we will build the **Document Dashboard** for the **DocuMind AI** system.

The Document Dashboard is where users manage their uploaded documents, track processing states, view OCR results, inspect AI Analysis summaries, and access the AI Chat/RAG features. It serves as the primary user interface enabling users to monitor the entire document processing pipeline from upload to completion.

The Dashboard will connect to the Backend API to retrieve the list of documents, processing statuses, OCR results, and AI Analysis results stored in PostgreSQL.

---

#### Objectives of This Step

Upon completing this step, you will:

* Build the document list interface.
* Display the processing status of each document.
* Display the progress of document processing corresponding to its status.
* Call the API to fetch the list of documents.
* Call the API to fetch details of a specific document.
* Display the OCR result and AI Analysis.
* Integrate a button to upload new documents.
* Allow users to access and view their own documents.
* Allow Admins to see an overview of multiple documents if authorized.

---

#### The Role of the Document Dashboard

In DocuMind AI, the Document Dashboard helps users to:

* View a table of all uploaded documents.
* Know which stage a document is currently in within the pipeline.
* Inspect document metadata (e.g., file name, file type, file size, upload timestamp).
* Monitor processing statuses: `QUEUED`, `PROCESSING`, `OCR_COMPLETED`, `AI_ANALYZING`, `COMPLETED`, `FAILED`.
* Open document details to view OCR text and AI Analysis.
* Quickly access the AI Chat/RAG interface.
* Receive notifications when document processing completes.

---

#### Document Dashboard Architecture

Data flow:

```text
React Dashboard
  |
  v
GET /api/documents
  |
  v
Backend API
  |
  v
PostgreSQL + Prisma
  |
  v
Return documents + status + analysis summary
```
