---

title : "Cấu hình môi trường Production"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.13.2. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ cấu hình **môi trường production** cho hệ thống **DocuMind AI**.

Môi trường production cần được cấu hình cẩn thận để backend có thể kết nối PostgreSQL, Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, CloudWatch và Secrets Manager. Đồng thời, các thông tin nhạy cảm như API key, JWT secret và database URL cần được bảo vệ an toàn.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo file cấu hình production cho backend.
* Cấu hình biến môi trường cần thiết.
* Phân biệt local `.env` và production `.env`.
* Cấu hình AWS region, S3 bucket và SQS queue.
* Cấu hình Gemini/OpenAI provider.
* Cấu hình PostgreSQL connection.
* Cấu hình Secrets Manager nếu dùng.
* Kiểm tra backend đọc đúng biến môi trường.

---

#### Vai trò của cấu hình production

Production config quyết định backend sẽ kết nối đến dịch vụ nào.

Các nhóm cấu hình chính:

* App config.
* Database config.
* AWS config.
* AI Provider config.
* Auth config.
* CloudWatch config.
* Secrets Manager config.
* CORS config.

Nếu cấu hình sai, hệ thống có thể gặp lỗi như không kết nối được database, không upload được S3, không gửi được SQS hoặc không gọi được Gemini/OpenAI.

---

#### Kiến trúc cấu hình môi trường

```text id="env-flow"
.env.production / Secrets Manager
  |
  v
Backend Config Loader
  |
  v
Backend API + Worker
  |
  v
External Services
```

![configure-env-production](/images/5-Workshop/5.13-Deployment-and-Test/5.13.2-configure-env-production/configure-env-production.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình file `.env.production` nhưng phải che các giá trị nhạy cảm như API key, JWT secret và database password.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.2-configure-env-production/configure-env-production.png`

---

#### File môi trường local và production

Local development có thể dùng:

```text id="local-env"
.env
```

Production có thể dùng:

```text id="prod-env"
.env.production
```

Không nên commit các file này lên GitHub.

Chỉ nên commit file mẫu:

```text id="example-env"
.env.example
```

---

#### Cấu hình App

```env id="app-config"
NODE_ENV="production"
PORT=3000
APP_NAME="DocuMind AI"
APP_URL="https://your-domain.com"
FRONTEND_URL="https://your-frontend-domain.com"
```

Nếu chưa có domain, có thể dùng IP EC2 để test:

```env id="app-ip"
APP_URL="http://your-ec2-public-ip:3000"
FRONTEND_URL="http://your-frontend-domain-or-localhost"
```

---

#### Cấu hình Database

DocuMind AI sử dụng **PostgreSQL + Prisma**, không sử dụng Amazon RDS trong workshop này.

```env id="database-config"
DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"
```

Nếu PostgreSQL chạy cùng EC2:

```env id="database-localhost"
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Nếu PostgreSQL chạy trên server riêng:

```env id="database-remote"
DATABASE_URL="postgresql://docmind:docmind_password@your-postgres-server-ip:5432/docmind"
```

---

#### Cấu hình AWS

```env id="aws-config"
AWS_REGION="ap-southeast-1"

AWS_S3_BUCKET_NAME="documind-document-storage"

AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
```

Nếu backend chạy trên EC2 có IAM Role, không cần:

```env id="no-access-key"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

AWS SDK sẽ tự lấy quyền từ IAM Role.

---

#### Cấu hình AI Provider

Nếu dùng OpenAI làm provider mặc định:

```env id="openai-config"
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
```

Nếu dùng Gemini làm provider mặc định:

```env id="gemini-config"
AI_PROVIDER="gemini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

Nếu dùng cả hai provider để fallback, nên cấu hình cả hai key:

```env id="multi-ai-config"
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

---

#### Cấu hình Authentication

```env id="auth-config"
JWT_SECRET="your-strong-jwt-secret"
JWT_EXPIRES_IN="1d"
REFRESH_TOKEN_EXPIRES_IN="7d"
```

Nếu có Google OAuth:

```env id="oauth-config"
GOOGLE_CLIENT_ID="your-google-client-id"
GOOGLE_CLIENT_SECRET="your-google-client-secret"
GOOGLE_CALLBACK_URL="https://your-domain.com/api/auth/google/callback"
```

---

#### Cấu hình CORS

```env id="cors-config"
CORS_ORIGIN="https://your-frontend-domain.com"
```

Nếu test tạm thời từ local:

```env id="cors-local"
CORS_ORIGIN="http://localhost:5173"
```

Không nên để production quá rộng:

```env id="bad-cors"
CORS_ORIGIN="*"
```

nếu hệ thống xử lý tài liệu riêng tư.

---

#### Cấu hình CloudWatch

```env id="cloudwatch-config"
CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-production"
CLOUDWATCH_WORKER_LOG_STREAM="worker-production"
```

---

#### Cấu hình Secrets Manager

Nếu dùng AWS Secrets Manager:

```env id="secrets-config"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

Khi đó, các secret như `DATABASE_URL`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `JWT_SECRET` có thể được load từ Secrets Manager.

---

#### File `.env.production` mẫu đầy đủ

```env id="full-env-production"
NODE_ENV="production"
PORT=3000
APP_NAME="DocuMind AI"
APP_URL="https://your-domain.com"
FRONTEND_URL="https://your-frontend-domain.com"

DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"

AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"

AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"

JWT_SECRET="your-strong-jwt-secret"
JWT_EXPIRES_IN="1d"
REFRESH_TOKEN_EXPIRES_IN="7d"

CORS_ORIGIN="https://your-frontend-domain.com"

CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-production"
CLOUDWATCH_WORKER_LOG_STREAM="worker-production"
```

---

#### File `.env.example`

File `.env.example` nên có key nhưng không có giá trị thật:

```env id="env-example"
NODE_ENV=
PORT=

DATABASE_URL=

AWS_REGION=
AWS_S3_BUCKET_NAME=
AWS_SQS_QUEUE_URL=
AWS_SQS_DLQ_URL=

AI_PROVIDER=

OPENAI_API_KEY=
OPENAI_MODEL=
OPENAI_EMBEDDING_MODEL=

GEMINI_API_KEY=
GEMINI_MODEL=
GEMINI_EMBEDDING_MODEL=

JWT_SECRET=
JWT_EXPIRES_IN=
REFRESH_TOKEN_EXPIRES_IN=

CORS_ORIGIN=

USE_SECRETS_MANAGER=
AWS_SECRET_NAME=
```

---

#### Kiểm tra backend đọc đúng env

Tạo log kiểm tra an toàn:

```text id="safe-config-log"
[CONFIG_LOADED]
NODE_ENV=production
AWS_REGION=ap-southeast-1
AI_PROVIDER=openai
```

Không log:

```text id="no-log-secret"
DATABASE_URL
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
```

---

#### Test cấu hình production

Các bước test:

1. Tạo `.env.production`.
2. Chạy `npm run build`.
3. Chạy `npx prisma migrate deploy`.
4. Start backend bằng PM2.
5. Kiểm tra backend health check.
6. Upload thử tài liệu.
7. Kiểm tra S3 object.
8. Kiểm tra message SQS.
9. Chạy worker.
10. Kiểm tra OCR và AI Analysis.

---

#### Các lỗi thường gặp

| Lỗi                    | Nguyên nhân                        | Cách xử lý                    |
| ---------------------- | ---------------------------------- | ----------------------------- |
| Backend không đọc env  | File sai tên hoặc chưa load dotenv | Kiểm tra config loader        |
| Database không kết nối | Sai `DATABASE_URL`                 | Test bằng `psql`              |
| S3 AccessDenied        | IAM Role thiếu quyền               | Kiểm tra role/policy          |
| QueueDoesNotExist      | Sai Queue URL hoặc region          | Kiểm tra SQS URL              |
| AI provider lỗi        | Sai API key/model                  | Kiểm tra OpenAI/Gemini config |
| CORS lỗi               | Sai `CORS_ORIGIN`                  | Thêm đúng frontend domain     |
| Lộ secret trong log    | Log config quá chi tiết            | Mask secret                   |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* `.env.production` đã được tạo.
* Các biến app/database/AWS/AI/auth đã được cấu hình.
* Không hardcode AWS Access Key trong production.
* Không commit secret lên GitHub.
* Backend đọc được production env.
* Prisma kết nối được PostgreSQL.
* Backend gọi được S3, SQS, Textract, Gemini/OpenAI.
* CORS hoạt động với frontend.
* Secrets Manager được cấu hình nếu dùng.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có môi trường production rõ ràng và an toàn hơn. Backend và worker có thể đọc đúng cấu hình, kết nối các dịch vụ cần thiết và sẵn sàng chạy kiểm thử full workflow.
