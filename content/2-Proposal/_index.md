---
title: "Proposal"
date: 2026-06-10
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

In this section, you need to summarize the workshop contents that you **plan** to carry out.

# DocuMind AI – Cloud-Native AI Document Extractor & Analytics
## An AWS Cloud Solution Integrated with AI for Automated Document Extraction, Analysis, and Q&A

### 1. Executive Summary
DocuMind AI is an automated document extraction and analysis platform designed to help users process documents such as invoices, forms, contracts, reports, or internal files. The system allows users to upload documents, automatically extract content using OCR, analyze data with AI, classify documents, summarize content, extract important information, and ask questions directly about the document through AI Chat/RAG functionality.

The platform is built with a cloud-native architecture and uses AWS services such as Amazon S3, Amazon SQS, Amazon Textract, PostgreSQL, CloudWatch, Secrets Manager, IAM Role, and AWS WAF. The AI layer is integrated with two primary models, Google Gemini API and OpenAI ChatGPT API, to improve flexibility and provide fallback capability when one AI provider fails or hits quota limits.

The goal of the project is to build a modern document processing system that is scalable, secure, easy to monitor, and suitable for real-world enterprise or organizational document digitization and analysis needs.

---

### 2. Problem Statement

#### Current Problem
In many businesses and organizations, document processing still depends heavily on manual work. Users must read each document, re-enter important data, manually classify documents, and summarize the needed information. This process is time-consuming, error-prone, and difficult to scale as the document volume increases.

In addition, many existing systems only provide document storage and do not have the ability to understand document content, summarize information, extract structured data, or support direct question answering over document content. The lack of monitoring, error alerts, and processing-state management also makes real-world operations difficult.

#### Proposed Solution
DocuMind AI provides an automated document processing platform built on AWS Cloud and AI. Users can upload documents to the system, and the documents are stored in Amazon S3. The system then sends processing jobs to Amazon SQS to ensure asynchronous handling and avoid overload. Amazon Textract is used to extract text from the documents. The OCR output is then sent to Gemini API or OpenAI ChatGPT API to perform analysis tasks such as summarization, classification, entity extraction, structured data generation, and document Q&A.

The processing results are stored in PostgreSQL through Prisma ORM. Users can view processing status, OCR results, AI analysis results, chat history, and notifications in the web interface. The system also includes an Admin area for tracking users, documents, processing errors, notifications, audit logs, and system status.

#### Benefits and Value
The solution reduces manual document-processing time, improves the accuracy of extraction and information synthesis, and provides an intelligent experience through AI Chat. The system can be applied to many document types such as invoices, contracts, records, reports, or enterprise documents.

From a technical perspective, the project demonstrates the ability to design and implement a cloud-native system that integrates multiple AWS services, uses queue-based asynchronous processing, strengthens security with IAM Role, Secrets Manager, and AWS WAF, monitors operations with CloudWatch, and integrates modern AI models such as Gemini and ChatGPT.

---

### 3. Solution Architecture
DocuMind AI applies a cloud-native architecture combined with asynchronous processing. Users interact with a React web interface. When a document is uploaded, the Node.js/Express backend stores the file in Amazon S3 and creates metadata in PostgreSQL. The system then sends a message to Amazon SQS for the worker to process the document.

The worker consumes the SQS message, calls Amazon Textract to extract text from the document, and then sends the OCR output to the AI Provider Gateway. The AI Provider Gateway enables selection or fallback between Google Gemini API and OpenAI ChatGPT API. The analysis result is stored in PostgreSQL and displayed again on the Dashboard.

Overall architecture:

User
→ React Frontend
→ Backend API Node.js/Express
→ Amazon S3 for document storage
→ Amazon SQS for queue processing
→ Amazon Textract OCR
→ AI Provider Gateway
→ Gemini API / OpenAI ChatGPT API
→ PostgreSQL Database
→ Dashboard / AI Chat / Analytics

#### AWS Services Used

- Amazon S3: Stores original documents and supports document file management.
- Amazon SQS: Manages asynchronous document-processing queues.
- Amazon Textract: Extracts text from PDF, PNG, JPG, or JPEG files.
- PostgreSQL: Stores user data, documents, OCR, AI analysis, chat history, notifications, and audit logs.
- Amazon CloudWatch: Logs and monitors backend errors, AI errors, OCR errors, and processing status.
- AWS Secrets Manager: Stores sensitive information such as database URL, JWT secret, OpenAI API key, and Gemini API key.
- IAM Role: Grants secure access for the backend/EC2 to AWS services without hardcoding access keys.
- AWS WAF: Protects the application from malicious requests such as SQL injection, XSS, brute force, and abnormal traffic.

#### System Components

- Frontend: User interface including Home Page, Dashboard, Document Management, AI Models, Enterprise Sandbox, Notification Center, and Admin Panel.
- Backend API: Handles authentication, document uploads, document management, AI service calls, and exposes APIs for the frontend.
- Document Processing Worker: Receives jobs from SQS, performs OCR, and calls AI.
- AI Gateway Service: Manages the two main AI providers, Gemini and OpenAI.
- Admin Module: Manages users, documents, notifications, audit logs, and system status.
- Notification System: Sends notifications to User and Admin when upload, OCR, AI analysis, or system errors occur.

---

### 4. Technical Implementation

#### Implementation Phases
The project is divided into several implementation stages:

1. Requirements analysis and architecture design
    Identify the main system functions, design the cloud-native architecture, choose suitable AWS services, and build the overall diagram.
2. Backend and database development
    Develop the backend with Node.js/Express, design the PostgreSQL database with Prisma ORM, and build APIs for authentication, documents, OCR, AI analysis, and notifications.
3. AWS Storage and Queue integration
    Integrate Amazon S3 for document storage and Amazon SQS for asynchronous document queues.
4. OCR and AI integration
    Use Amazon Textract to extract text from documents. Integrate Gemini API and OpenAI ChatGPT API for document analysis, summarization, classification, and document Q&A.
5. User interface development
    Build the React/TypeScript interface with TailwindCSS, including Home Page, Dashboard, Upload Page, Document Detail, AI Chat, Enterprise Sandbox, Notification Center, and Admin Dashboard.
6. Security and monitoring
    Integrate JWT authentication, role-based access control, IAM Role, Secrets Manager, CloudWatch, AWS WAF, and Audit Log.
7. Testing and deployment
    Test document upload, OCR, AI analysis, AI Chat, notifications, admin module, logging, and security. Then deploy the backend to EC2 and connect it with PostgreSQL/S3/SQS.

#### Technical Requirements

- Backend uses Node.js, Express.js, and TypeScript.
- Frontend uses React, TypeScript, TailwindCSS, and Framer Motion.
- Database uses PostgreSQL.
- Prisma ORM is used to manage schema and migrations.
- JWT is used for authentication and User/Admin authorization.
- Amazon S3 is used to store document files.
- Amazon SQS is used to process asynchronous jobs.
- Amazon Textract is used for OCR.
- Gemini API and OpenAI ChatGPT API are used for document analysis and AI Chat.
- CloudWatch is used to track logs and system errors.
- Secrets Manager is used to manage secrets.
- AWS WAF is used to strengthen request security.

---

### 5. Timeline and Milestones

- Stage 1: Requirements analysis and system architecture design.
- Stage 2: Backend, database schema, and authentication development.
- Stage 3: Document upload to S3 and queue processing with SQS integration.
- Stage 4: Textract OCR and AI Provider Gateway integration.
- Stage 5: Gemini API and OpenAI ChatGPT API integration.
- Stage 6: Dashboard, AI Chat, Semantic Search, and Enterprise Sandbox development.
- Stage 7: Notification System for User and Admin.
- Stage 8: Admin Panel, Audit Log, CloudWatch, and AWS WAF integration.
- Stage 9: System testing, UI refinement, and AWS deployment.

---

### 6. Budget Estimation
Infrastructure cost depends on the number of documents, storage usage, OCR calls, and AI API calls. For the prototype and thesis scope, the system can run at low cost by using small resources and limiting requests.

Possible cost components:

- Amazon S3: Stores original documents.
- Amazon SQS: Processes the message queue.
- Amazon Textract: Extracts text from documents.
- PostgreSQL: Stores application data.
- Amazon EC2: Runs the backend API.
- Amazon CloudWatch: Stores logs and monitors the system.
- AWS Secrets Manager: Stores secrets.
- AWS WAF: Protects the web/API application.
- OpenAI API: Handles AI analysis, AI Chat, and embeddings.
- Gemini API: Handles AI analysis, AI Chat, and fallback provider logic.

For demo environments, costs can be optimized by:

- Limiting the number of test documents.
- Using a small EC2 instance.
- Using a low-cost database configuration.
- Removing unnecessary test data.
- Tracking billing with AWS Budgets.
- Enabling WAF/CloudWatch logging only at the level needed for the demo.

---

### 7. Risk Assessment

#### Risk Matrix

- OCR errors due to unsupported document formats: Medium impact, medium probability.
- AI API quota or rate-limit issues: High impact, medium probability.
- Large file uploads slowing the system: Medium impact, medium probability.
- Database errors or migration failures: High impact, low probability.
- Exposure of API keys or sensitive information: High impact, low probability.
- AWS/API costs increasing beyond expectations: Medium impact, medium probability.
- Backend worker errors or failed messages: Medium impact, medium probability.

#### Mitigation Strategies

- Allow only supported formats such as PDF, PNG, JPG, and JPEG.
- Use SQS and retry mechanisms for more stable document processing.
- Add fallback between Gemini and OpenAI when one provider fails.
- Use Secrets Manager to store API keys and secrets.
- Use IAM Role instead of hardcoding AWS access keys.
- Use CloudWatch to monitor logs, errors, and processing status.
- Use AWS WAF to reduce risk from malicious requests.
- Use AWS Budgets to control costs.

#### Contingency Plans

- If Gemini API fails, switch to OpenAI API.
- If OpenAI API fails, switch to Gemini API.
- If OCR fails, update the status to FAILED and notify the user.
- If the worker fails, the message can be retried or moved to a dead-letter queue.
- If the cloud system is temporarily unstable, the documents remain stored in S3 for later reprocessing.

---

### 8. Expected Outcomes
After completion, DocuMind AI can help users upload documents, extract content with OCR, analyze documents with AI, ask questions in document context, and track processing status on the dashboard.

From a technical perspective, the project demonstrates how to build a cloud-native system with practical components such as storage, queueing, OCR, AI providers, database, monitoring, security, and admin management. This becomes a foundation that can be expanded for enterprise use cases such as invoice management, contract analysis, form processing, internal document automation, or knowledge management systems.

Expected results include:

- Reduced manual reading and data-entry work.
- Faster document processing.
- Structured data extracted from unstructured documents.
- Direct Q&A support for documents.
- Admin monitoring of users, documents, processing errors, and system status.
- Improved security and operability through WAF, CloudWatch, Secrets Manager, and IAM Role.