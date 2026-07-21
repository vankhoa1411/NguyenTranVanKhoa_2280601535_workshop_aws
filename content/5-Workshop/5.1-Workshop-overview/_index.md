---
title : "Introduction"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Introduction to DocuMind AI

**DocuMind AI** is an automated document extraction, analysis, and Q&A platform based on a Cloud-Native architecture. The system supports uploading documents such as invoices, forms, contracts, reports, or internal documents; then automatically extracts content using OCR, analyzes it using AI, and displays the results on a dashboard.

In this workshop, you will build a complete document processing workflow, combining AWS services such as **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Amazon RDS PostgreSQL**, **Amazon CloudWatch**, **AWS Secrets Manager**, **IAM Role**, and **AWS WAF**. The AI component of the system uses two main providers, **Google Gemini API** and **OpenAI ChatGPT API**, to analyze documents, summarize content, extract key information, and support AI Chat/RAG.

#### Objectives of the Workshop

After completing the workshop, you will be able to:

- Build a document upload and storage system on Amazon S3.
- Create an asynchronous processing queue using Amazon SQS.
- Use Amazon Textract for OCR on PDF documents or images.
- Integrate Gemini API and OpenAI ChatGPT API as the two main AI providers.
- Save metadata, OCR results, AI analysis, notifications, and audit logs to Amazon RDS PostgreSQL.
- Monitor logs and system errors using Amazon CloudWatch.
- Manage secrets using AWS Secrets Manager instead of hardcoding API keys.
- Enhance request security using IAM Roles and AWS WAF.
- Build a dashboard, AI Chat, Enterprise Sandbox, Notification Center, and Admin Panel.

#### Overall Architecture of the Workshop

The workshop is designed based on a **Cloud-Native Document Processing Pipeline** model combined with asynchronous processing using message queues. The Frontend communicates with the Backend API, the Backend stores documents in S3 and sends messages to SQS. The Worker receives messages, calls Textract to extract text, and then sends the content to the AI Gateway to be processed by Gemini or OpenAI. The final results are stored in PostgreSQL and displayed back to the user.

![overview](/images/5-Workshop/5.1-Workshop-overview/architecture.png)


#### System Workflow

1. **User uploads document to the system**  
   The user uploads PDF, PNG, JPG, or JPEG files through the React interface. The Backend checks the format, stores the file in **Amazon S3**, and creates initial metadata in **Amazon RDS PostgreSQL** with the status `PENDING`.

2. **Backend sends job to the queue**  
   After a successful upload, the Backend sends a message containing `documentId`, `s3Key`, `userId`, and processing information to **Amazon SQS**. This method helps the system process documents asynchronously, avoiding long wait times for users on the interface.

3. **Worker processes OCR using Amazon Textract**  
   Worker receives messages from SQS, retrieves documents from S3, and calls **Amazon Textract** to extract text. The OCR result is saved to the database, and the document status is updated to `OCR_COMPLETED` or `FAILED` in case of error.

4. **AI Gateway analyzes the document**  
   The text after OCR is sent to the **AI Gateway Service**. The Gateway selects the main provider based on configuration or user preference, including **OpenAI ChatGPT API** or **Google Gemini API**. The AI performs summarization, classification, entity extraction, generates structured JSON, and supports contextual document Q&A.

5. **Save results and send notifications**  
   AI Analysis results, Chat History, Notifications, and Audit Logs are saved to **Amazon RDS PostgreSQL**. Users receive notifications when document processing is complete, and the Admin can view system errors, AI provider failures, processing status, and audit logs.

6. **Monitoring and Security**  
   **CloudWatch** logs the backend, worker, OCR, and AI processing. **Secrets Manager** manages important API keys and secrets. **IAM Role** grants secure permissions for backend/EC2 to access AWS services. **AWS WAF** protects requests from common attacks such as SQL Injection, XSS, brute force, and anomalous traffic.

#### Key Components in the Workshop

- **Frontend Web App**: Home Page, Dashboard, Upload, Document Detail, AI Chat, Enterprise Sandbox, Notification Center, and Admin Panel.
- **Backend API**: Handles authentication, document uploads, document management, calls SQS, calls AI service, and provides data to the frontend.
- **Document Worker**: Receives jobs from SQS, calls Textract OCR, calls AI Gateway, and updates results.
- **AI Gateway**: Manages the two main providers, Gemini and OpenAI, supporting fallback when one provider fails or hits quota limits.
- **Database**: Stores users, roles, documents, OCR results, AI analysis, chat history, notifications, and audit logs.
- **Monitoring & Security**: CloudWatch, Secrets Manager, IAM Role, and AWS WAF.

#### Expected Outcomes After the Workshop

After the workshop, you will have a DocuMind AI system capable of document upload, OCR, AI analysis, document Q&A, dashboard visualization, sending notifications to Users/Admins, and monitoring system errors. This architecture can be scaled for practical problems such as invoice management, contract analysis, form processing, internal document management, or building an enterprise knowledge management system.
