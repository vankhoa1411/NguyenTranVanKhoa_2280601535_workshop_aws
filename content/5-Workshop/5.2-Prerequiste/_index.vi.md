---
title : "Chuẩn bị"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

#### Tổng quan phần chuẩn bị

Trước khi triển khai workshop **DocuMind AI**, bạn cần chuẩn bị tài khoản AWS, môi trường phát triển local, API key cho Gemini/OpenAI và một số dịch vụ nền tảng như S3, SQS, Textract, RDS PostgreSQL, CloudWatch, Secrets Manager, IAM Role và AWS WAF.

Phần này giúp đảm bảo hệ thống có đủ điều kiện để chạy luồng xử lý:

**Upload tài liệu → S3 → SQS → Worker → Textract OCR → Gemini/OpenAI AI Gateway → PostgreSQL → Dashboard/Notification**

---

### 1. Tài khoản và quyền truy cập AWS

Bạn cần có một tài khoản AWS có quyền tạo và cấu hình các dịch vụ sau:

- Amazon S3
- Amazon SQS
- Amazon Textract
- Amazon RDS PostgreSQL
- Amazon CloudWatch
- AWS Secrets Manager
- AWS IAM
- AWS WAF
- Amazon EC2 nếu triển khai backend lên server

Đối với môi trường học tập hoặc demo, có thể sử dụng tài khoản AWS Free Tier. Tuy nhiên, một số dịch vụ như Textract, RDS, WAF, Secrets Manager hoặc CloudWatch Logs vẫn có thể phát sinh chi phí tùy mức sử dụng.

> ⚠️ **Lưu ý chi phí:**
> Hãy bật AWS Budgets hoặc thường xuyên kiểm tra Billing Dashboard để tránh phát sinh chi phí ngoài dự kiến trong quá trình test OCR, AI API, RDS, WAF và CloudWatch logging.

---

### 2. Khu vực triển khai AWS

Trong workshop, nên chọn một region cố định để tránh lỗi sai region giữa các dịch vụ.

Ví dụ:

```bash
AWS_REGION=ap-southeast-1