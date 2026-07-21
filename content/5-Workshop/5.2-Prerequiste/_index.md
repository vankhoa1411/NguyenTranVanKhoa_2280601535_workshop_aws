---
title : "Prerequisites"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

#### Prerequisites Overview

Before deploying the **DocuMind AI** workshop, you need to prepare an AWS account, a local development environment, Gemini/OpenAI API keys, and several fundamental services such as S3, SQS, Textract, RDS PostgreSQL, CloudWatch, Secrets Manager, IAM Role, and AWS WAF.

This section ensures that the system is ready to execute the processing flow:

**Document Upload → S3 → SQS → Worker → Textract OCR → Gemini/OpenAI AI Gateway → PostgreSQL → Dashboard/Notification**

---

### 1. AWS Account and Permissions

You need an AWS account with permissions to create and configure the following services:

- Amazon S3
- Amazon SQS
- Amazon Textract
- Amazon RDS PostgreSQL
- Amazon CloudWatch
- AWS Secrets Manager
- AWS IAM
- AWS WAF
- Amazon EC2 if deploying the backend to a server

For learning or demo environments, you can use an AWS Free Tier account. However, some services like Textract, RDS, WAF, Secrets Manager, or CloudWatch Logs may still incur costs based on usage.

> ⚠️ **Cost Warning:**
> Enable AWS Budgets or check the Billing Dashboard regularly to avoid unexpected charges during OCR testing, AI API calls, RDS usage, WAF protection, and CloudWatch logging.

---

### 2. AWS Deployment Region

In this workshop, you should select a single fixed region to avoid region mismatch errors across services.

For example:

```bash
AWS_REGION=ap-southeast-1
```