---

title : "Gắn IAM Role vào Backend"
date : 2026-06-10
weight : 5
chapter : false
pre : " <b> 5.11.5. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ gắn các IAM Policy đã tạo vào **IAM Role**, sau đó gắn IAM Role đó vào backend/worker của **DocuMind AI**.

Nếu backend chạy trên **EC2**, chúng ta sẽ attach IAM Role vào EC2 instance. Khi đó, AWS SDK trong backend có thể tự lấy credential tạm thời từ IAM Role để gọi các dịch vụ AWS như S3, SQS, Textract, CloudWatch và Secrets Manager.

Cách này giúp hệ thống không cần lưu AWS Access Key trong `.env` production.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Attach S3 Policy vào IAM Role.
* Attach SQS + Textract Policy vào IAM Role.
* Attach CloudWatch + Secrets Manager Policy vào IAM Role.
* Gắn IAM Role vào EC2 backend server nếu có.
* Kiểm tra backend/worker dùng được IAM Role.
* Loại bỏ AWS Access Key khỏi production `.env`.

---

#### Kiến trúc gắn IAM Role

Luồng hoạt động:

```text id="b8kom4"
EC2 Backend Server
  |
  v
IAM Role: DocuMindBackendWorkerRole
  |
  |-- DocuMindS3AccessPolicy
  |-- DocuMindSQSTextractPolicy
  |-- DocuMindCloudWatchSecretsPolicy
  |
  v
AWS Services
```

![attach-role-backend](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.5-attach-role-to-backend/attach-role-backend.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình IAM Role đã attach đủ 3 policy S3, SQS/Textract và CloudWatch/Secrets, hoặc chụp EC2 instance đã được gắn IAM Role.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.5-attach-role-to-backend/attach-role-backend.png`

---

#### Bước 1: Mở IAM Role

Vào AWS Console:

```text id="0niwqt"
IAM
```

Chọn:

```text id="kvr2oz"
Roles
```

Mở role đã tạo:

```text id="2fiiux"
DocuMindBackendWorkerRole
```

---

#### Bước 2: Attach S3 Policy

Trong tab Permissions, chọn:

```text id="2tj42f"
Add permissions
```

Chọn:

```text id="m4cd1u"
Attach policies
```

Tìm policy:

```text id="82ja7f"
DocuMindS3AccessPolicy
```

Tick chọn policy và nhấn:

```text id="0dl3wz"
Add permissions
```

---

#### Bước 3: Attach SQS + Textract Policy

Tiếp tục attach policy:

```text id="fxsk4r"
DocuMindSQSTextractPolicy
```

Policy này cho phép backend/worker:

* Gửi message vào SQS.
* Nhận message từ SQS.
* Xóa message sau khi xử lý.
* Gọi Amazon Textract OCR.

---

#### Bước 4: Attach CloudWatch + Secrets Manager Policy

Tiếp tục attach policy:

```text id="rhl6h3"
DocuMindCloudWatchSecretsPolicy
```

Policy này cho phép backend/worker:

* Ghi log lên CloudWatch.
* Đọc secret từ AWS Secrets Manager.

---

#### Bước 5: Kiểm tra role đã có đủ policy

Sau khi attach, IAM Role nên có các policy:

```text id="w22gvq"
DocuMindS3AccessPolicy
DocuMindSQSTextractPolicy
DocuMindCloudWatchSecretsPolicy
```

Kiểm tra tab Permissions để đảm bảo không thiếu policy nào.

---

#### Bước 6: Gắn IAM Role vào EC2

Nếu backend chạy trên EC2, vào:

```text id="2zzlal"
EC2
```

Chọn instance backend.

Sau đó chọn:

```text id="lke7ve"
Actions
→ Security
→ Modify IAM role
```

Chọn role:

```text id="8pvbk6"
DocuMindBackendWorkerRole
```

Nhấn:

```text id="rbsxqt"
Update IAM role
```

---

#### Bước 7: Kiểm tra IAM Role trên EC2

SSH vào EC2 và chạy:

```bash id="99p9t5"
aws sts get-caller-identity
```

Kết quả mong đợi sẽ có dạng assumed role:

```text id="m1jwmg"
arn:aws:sts::123456789012:assumed-role/DocuMindBackendWorkerRole/i-xxxxxxxxxxxx
```

Nếu thấy ARN của role, EC2 đã dùng đúng IAM Role.

---

#### Bước 8: Cập nhật `.env` production

Production `.env` không cần các biến này:

```env id="6xy9mq"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Chỉ cần giữ cấu hình:

```env id="6o7qy4"
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

AWS SDK sẽ tự lấy credential từ IAM Role.

---

#### Bước 9: Kiểm tra backend AWS SDK

Trong backend, AWS SDK client chỉ cần region:

```ts id="a7q7n3"
import { S3Client } from "@aws-sdk/client-s3";

export const s3Client = new S3Client({
  region: process.env.AWS_REGION
});
```

Không cần truyền access key trực tiếp:

```ts id="zirrqk"
credentials: {
  accessKeyId: "...",
  secretAccessKey: "..."
}
```

---

#### Bước 10: Test nhanh các service

Test S3:

```bash id="fyz2jn"
aws s3 ls s3://documind-document-storage
```

Test SQS:

```bash id="3dmrct"
aws sqs get-queue-attributes \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --attribute-names All
```

Test Secrets Manager:

```bash id="7ioru4"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân                      | Cách xử lý                |
| -------------------------------- | -------------------------------- | ------------------------- |
| `Unable to locate credentials`   | EC2 chưa gắn IAM Role            | Modify IAM Role cho EC2   |
| `AccessDenied` S3                | Chưa attach S3 Policy            | Kiểm tra role permissions |
| `AccessDenied` SQS               | Chưa attach SQS Policy           | Kiểm tra policy ARN queue |
| `AccessDeniedException` Textract | Thiếu Textract permission        | Kiểm tra policy           |
| Không đọc được secret            | Thiếu Secrets Manager permission | Kiểm tra secret ARN       |
| `aws sts` không ra assumed-role  | AWS CLI đang dùng profile local  | Kiểm tra credential chain |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* IAM Role có đủ S3 Policy.
* IAM Role có đủ SQS + Textract Policy.
* IAM Role có đủ CloudWatch + Secrets Manager Policy.
* EC2 backend đã gắn IAM Role nếu deploy trên EC2.
* `aws sts get-caller-identity` hiển thị assumed role.
* Backend không hardcode AWS credentials.
* AWS SDK có thể gọi S3, SQS, Textract, CloudWatch và Secrets Manager.

---

#### Kết quả kỳ vọng

Sau bước này, backend và worker của DocuMind AI có thể truy cập các dịch vụ AWS thông qua IAM Role thay vì dùng access key cố định. Đây là cấu hình an toàn hơn và phù hợp với môi trường production.
