---

title : "Tạo CloudWatch và Secrets Manager Policy"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.11.4. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **IAM Policy cho Amazon CloudWatch và AWS Secrets Manager** cho hệ thống **DocuMind AI**.

CloudWatch được sử dụng để lưu log backend và worker, giúp theo dõi lỗi upload, OCR, AI Analysis, SQS processing và security events. AWS Secrets Manager được sử dụng để lưu các thông tin nhạy cảm như `DATABASE_URL`, `OPENAI_API_KEY`, `GEMINI_API_KEY` hoặc các cấu hình production khác.

Policy này giúp backend/worker có quyền ghi log và đọc secret khi chạy trong môi trường production.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được IAM Policy cho CloudWatch Logs.
* Cho phép backend/worker ghi log lên CloudWatch.
* Tạo quyền đọc secret từ AWS Secrets Manager.
* Giới hạn quyền theo secret của DocuMind AI nếu có thể.
* Chuẩn bị policy để attach vào IAM Role.

---

#### Vai trò của CloudWatch trong DocuMind AI

CloudWatch giúp hệ thống:

* Ghi log backend.
* Ghi log worker.
* Theo dõi lỗi S3 upload.
* Theo dõi lỗi SQS worker.
* Theo dõi lỗi Textract OCR.
* Theo dõi lỗi Gemini/OpenAI.
* Theo dõi security event và forbidden access.
* Hỗ trợ Admin kiểm tra lỗi vận hành.

Log group đề xuất:

```text id="qga2aq"
/docmind/application
```

---

#### Vai trò của Secrets Manager

Secrets Manager giúp lưu trữ thông tin nhạy cảm như:

```text id="9bbrzb"
DATABASE_URL
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
AWS_SQS_QUEUE_URL
```

Production không nên hardcode secret trong source code hoặc commit `.env` thật lên GitHub.

---

#### Luồng CloudWatch và Secrets Manager

```text id="bksf96"
Backend / Worker
  |
  |-- Write logs --> CloudWatch Logs
  |
  |-- Read secrets --> AWS Secrets Manager
  |
  v
Run application securely
```

![cloudwatch-secrets-policy](/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.4-create-cloudwatch-secrets-policy/cloudwatch-secrets-policy.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình IAM Policy có quyền CloudWatch Logs và Secrets Manager, hoặc chụp CloudWatch Log Group `/docmind/application`.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.4-create-cloudwatch-secrets-policy/cloudwatch-secrets-policy.png`

---

#### Quyền CloudWatch Logs cần thiết

Các quyền thường cần:

```text id="ovw715"
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

Ý nghĩa:

| Quyền                     | Mục đích                          |
| ------------------------- | --------------------------------- |
| `logs:CreateLogGroup`     | Tạo log group nếu chưa có         |
| `logs:CreateLogStream`    | Tạo log stream cho backend/worker |
| `logs:PutLogEvents`       | Gửi log lên CloudWatch            |
| `logs:DescribeLogStreams` | Kiểm tra log stream hiện có       |

---

#### Quyền Secrets Manager cần thiết

Quyền đọc secret:

```text id="p3grf8"
secretsmanager:GetSecretValue
```

Nếu backend chỉ cần đọc secret, không nên cấp quyền ghi/xóa secret.

Không nên cấp quyền quá rộng như:

```text id="zc1lal"
secretsmanager:*
```

---

#### Bước 1: Tạo policy mới

Vào AWS Console:

```text id="bzy1nn"
IAM
```

Chọn:

```text id="l5sl6a"
Policies
```

Nhấn:

```text id="8qi01z"
Create policy
```

Chọn tab:

```text id="y70n3z"
JSON
```

---

#### Bước 2: Policy JSON cho CloudWatch và Secrets Manager

Policy mẫu:

```json id="xtn1n1"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindCloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": [
        "arn:aws:logs:ap-southeast-1:123456789012:log-group:/docmind/application",
        "arn:aws:logs:ap-southeast-1:123456789012:log-group:/docmind/application:*"
      ]
    },
    {
      "Sid": "DocuMindSecretsReadAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:docmind/*"
      ]
    }
  ]
}
```

> ⚠️ **Lưu ý:**
> Thay `ap-southeast-1`, `123456789012` và secret ARN theo AWS account thật của bạn.

---

#### Bước 3: Đặt tên policy

Đặt tên:

```text id="6f9byy"
DocuMindCloudWatchSecretsPolicy
```

Description:

```text id="j82slj"
Allows DocuMind AI backend and worker to write CloudWatch logs and read secrets from AWS Secrets Manager.
```

Sau đó nhấn:

```text id="xu2eh1"
Create policy
```

---

#### Bước 4: Chuẩn bị tên secret

Secret nên đặt theo prefix rõ ràng:

```text id="l6swpk"
docmind/backend
docmind/production
docmind/ai-keys
```

Ví dụ secret JSON:

```json id="sgajkz"
{
  "DATABASE_URL": "postgresql://username:password@host:5432/docmind",
  "OPENAI_API_KEY": "your-openai-api-key",
  "GEMINI_API_KEY": "your-gemini-api-key",
  "JWT_SECRET": "your-jwt-secret"
}
```

---

#### Bước 5: Cấu hình backend đọc secret

Backend có thể dùng biến môi trường để bật/tắt Secrets Manager:

```env id="jh61ny"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
```

Khi chạy local, có thể dùng `.env`.

Khi chạy production, backend đọc secret từ AWS Secrets Manager.

---

#### Bước 6: Logging cần có

Backend/worker nên ghi log:

```text id="bc8doy"
[APP_STARTED]
[DATABASE_CONNECTED]
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[TEXTRACT_OCR_STARTED]
[AI_ANALYSIS_STARTED]
[DOCUMENT_PROCESSING_FAILED]
```

Không nên log:

```text id="1zjxrq"
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
DATABASE_URL full password
AWS credentials
```

---

#### Test CloudWatch Logs

Sau khi attach policy vào role và chạy backend/worker, vào:

```text id="o0fprq"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Kiểm tra log stream có log mới hay không.

---

#### Test Secrets Manager

Có thể test đọc secret bằng AWS CLI:

```bash id="jnr6cp"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

Nếu role có quyền, lệnh sẽ trả về secret value.

---

#### Các lỗi thường gặp

| Lỗi                                    | Nguyên nhân                           | Cách xử lý                 |
| -------------------------------------- | ------------------------------------- | -------------------------- |
| Không ghi được CloudWatch log          | Thiếu `logs:PutLogEvents`             | Kiểm tra policy            |
| Không tạo được log stream              | Thiếu `logs:CreateLogStream`          | Thêm quyền                 |
| `AccessDeniedException` khi đọc secret | Thiếu `secretsmanager:GetSecretValue` | Kiểm tra policy            |
| Secret not found                       | Sai secret name                       | Kiểm tra `AWS_SECRET_NAME` |
| Log lộ secret                          | Log quá nhiều dữ liệu                 | Mask secret trước khi log  |
| Sai region                             | Secret/log group ở region khác        | Kiểm tra `AWS_REGION`      |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo policy CloudWatch + Secrets Manager.
* Policy có quyền ghi CloudWatch Logs.
* Policy có quyền đọc Secrets Manager.
* Resource ARN đúng region và account.
* Secret đặt theo prefix `docmind/`.
* Backend/worker không log secret.
* Policy sẵn sàng attach vào IAM Role.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có policy để backend/worker ghi log vào CloudWatch và đọc secret từ AWS Secrets Manager. Điều này giúp hệ thống dễ vận hành, bảo mật hơn và phù hợp hơn với môi trường production.
