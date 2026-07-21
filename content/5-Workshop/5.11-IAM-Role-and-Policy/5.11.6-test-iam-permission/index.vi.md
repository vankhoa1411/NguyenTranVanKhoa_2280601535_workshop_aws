---

title : "Kiểm tra IAM Permission"
date : 2026-06-10
weight : 6
chapter : false
pre : " <b> 5.11.6. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ kiểm tra toàn bộ quyền IAM đã cấu hình cho backend và worker của **DocuMind AI**.

Sau khi tạo IAM Role, tạo các policy và attach role vào backend/worker, cần kiểm tra xem hệ thống có thể thực hiện đầy đủ các thao tác với **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **CloudWatch Logs** và **AWS Secrets Manager** hay không.

Đây là bước xác nhận quan trọng trước khi chạy pipeline xử lý tài liệu end-to-end.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra backend/worker đang dùng đúng IAM Role.
* Kiểm tra quyền truy cập Amazon S3.
* Kiểm tra quyền gửi và nhận message SQS.
* Kiểm tra quyền gọi Amazon Textract.
* Kiểm tra quyền ghi log CloudWatch.
* Kiểm tra quyền đọc secret từ Secrets Manager.
* Xác định và sửa lỗi `AccessDenied` nếu có.

---

#### Luồng kiểm tra IAM Permission

```text id="qr6w2c"
Backend / Worker
  |
  v
IAM Role
  |
  |-- Test S3
  |-- Test SQS
  |-- Test Textract
  |-- Test CloudWatch
  |-- Test Secrets Manager
  |
  v
Permission Verified
```

![test-iam-permission](/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.6-test-iam-permission/test-iam-permission.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình terminal test `aws sts get-caller-identity`, test S3, test SQS hoặc log CloudWatch cho thấy quyền IAM hoạt động đúng.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.6-test-iam-permission/test-iam-permission.png`

---

#### Điều kiện trước khi kiểm tra

Trước khi test, hãy đảm bảo:

* Đã tạo IAM Role.
* Đã tạo S3 Policy.
* Đã tạo SQS + Textract Policy.
* Đã tạo CloudWatch + Secrets Manager Policy.
* Đã attach các policy vào IAM Role.
* Nếu dùng EC2, đã gắn IAM Role vào EC2 instance.
* Backend/worker có `AWS_REGION` đúng.
* Bucket, queue và secret đã tồn tại.

---

#### Bước 1: Kiểm tra identity hiện tại

Chạy lệnh:

```bash id="c2dgle"
aws sts get-caller-identity
```

Kết quả mong đợi nếu chạy trên EC2 có IAM Role:

```json id="1qk9sf"
{
  "UserId": "AROAxxxxxxxx:i-xxxxxxxx",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/DocuMindBackendWorkerRole/i-xxxxxxxx"
}
```

Nếu không thấy `assumed-role`, có thể EC2 chưa được gắn IAM Role hoặc AWS CLI đang dùng credential khác.

---

#### Bước 2: Test quyền S3 ListBucket

Chạy:

```bash id="0pwo2p"
aws s3 ls s3://documind-document-storage
```

Nếu thành công, role có quyền `s3:ListBucket`.

Nếu lỗi:

```text id="yrw09e"
AccessDenied
```

hãy kiểm tra policy có quyền `s3:ListBucket` trên bucket ARN:

```text id="g1ylnz"
arn:aws:s3:::documind-document-storage
```

---

#### Bước 3: Test quyền S3 PutObject

Tạo file test:

```bash id="lq7u1z"
echo "DocuMind IAM test file" > iam-test.txt
```

Upload lên S3:

```bash id="h5c7qe"
aws s3 cp iam-test.txt s3://documind-document-storage/test/iam-test.txt
```

Nếu thành công, role có quyền `s3:PutObject`.

---

#### Bước 4: Test quyền S3 GetObject

Download file test:

```bash id="qir9vr"
aws s3 cp s3://documind-document-storage/test/iam-test.txt downloaded-iam-test.txt
```

Nếu thành công, role có quyền `s3:GetObject`.

---

#### Bước 5: Test quyền SQS SendMessage

Gửi message test:

```bash id="w4kv2p"
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_iam_test","action":"PROCESS_DOCUMENT","source":"IAM_TEST"}'
```

Nếu thành công, AWS CLI sẽ trả về `MessageId`.

---

#### Bước 6: Test quyền SQS ReceiveMessage

Nhận message:

```bash id="qpicsz"
aws sqs receive-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

Nếu nhận được message, role có quyền `sqs:ReceiveMessage`.

---

#### Bước 7: Test quyền SQS DeleteMessage

Sau khi receive message, copy `ReceiptHandle` và chạy:

```bash id="8jnvbd"
aws sqs delete-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --receipt-handle "your-receipt-handle"
```

Nếu không lỗi, role có quyền `sqs:DeleteMessage`.

---

#### Bước 8: Test quyền Textract

Dùng một file PDF hoặc ảnh đã có trong S3.

Ví dụ object:

```text id="jje6ny"
s3://documind-document-storage/test/invoice-demo.pdf
```

Worker hoặc script backend gọi Textract với file này. Nếu dùng AWS SDK, test bằng chức năng OCR đã tạo ở phần 5.5.

Log mong đợi:

```text id="3pjy9b"
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
```

Nếu thiếu quyền, có thể gặp:

```text id="u41g4n"
AccessDeniedException
```

Nếu sai file hoặc key, có thể gặp:

```text id="adzjrj"
InvalidS3ObjectException
```

---

#### Bước 9: Test quyền CloudWatch Logs

Chạy backend hoặc worker và kiểm tra log có được gửi lên CloudWatch hay không.

Vào AWS Console:

```text id="jdbtzn"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Log mong đợi:

```text id="hx1mzb"
[APP_STARTED]
[SQS_WORKER_STARTED]
[DOCUMENT_UPLOAD_STARTED]
```

Nếu không tạo được log stream, kiểm tra quyền:

```text id="c92e2u"
logs:CreateLogStream
logs:PutLogEvents
```

---

#### Bước 10: Test quyền Secrets Manager

Chạy:

```bash id="lce44g"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

Nếu thành công, role có quyền:

```text id="a3ge3p"
secretsmanager:GetSecretValue
```

Nếu lỗi `AccessDeniedException`, kiểm tra secret ARN trong policy.

---

#### Bước 11: Test bằng backend pipeline

Sau khi test từng dịch vụ, chạy test end-to-end:

1. Start backend.
2. Start worker.
3. Upload file PDF từ frontend hoặc Postman.
4. Backend upload file lên S3.
5. Backend gửi message vào SQS.
6. Worker nhận message.
7. Worker gọi Textract.
8. Worker gọi Gemini/OpenAI.
9. Worker lưu kết quả vào PostgreSQL.
10. Frontend hiển thị document status `COMPLETED`.

Nếu pipeline chạy thành công, IAM Permission đã đúng.

---

#### Bảng kiểm tra quyền

| Service         | Test                          | Kết quả mong đợi       |
| --------------- | ----------------------------- | ---------------------- |
| STS             | `aws sts get-caller-identity` | Hiển thị assumed role  |
| S3              | `aws s3 ls`                   | List bucket thành công |
| S3              | `aws s3 cp` upload            | PutObject thành công   |
| S3              | `aws s3 cp` download          | GetObject thành công   |
| SQS             | `send-message`                | Có MessageId           |
| SQS             | `receive-message`             | Nhận được message      |
| SQS             | `delete-message`              | Xóa message thành công |
| Textract        | OCR file test                 | OCR completed          |
| CloudWatch      | Check log group               | Có log mới             |
| Secrets Manager | `get-secret-value`            | Đọc được secret        |

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân                         | Cách xử lý                      |
| -------------------------------- | ----------------------------------- | ------------------------------- |
| `Unable to locate credentials`   | Không có IAM Role hoặc credential   | Gắn role vào EC2                |
| `AccessDenied` S3                | Thiếu S3 policy                     | Kiểm tra bucket ARN             |
| `QueueDoesNotExist`              | Sai Queue URL hoặc region           | Kiểm tra `.env`                 |
| `AccessDenied` SQS               | Thiếu quyền SQS                     | Kiểm tra SQS policy             |
| `AccessDeniedException` Textract | Thiếu quyền Textract                | Kiểm tra Textract actions       |
| `InvalidS3ObjectException`       | Sai S3 key hoặc quyền S3            | Kiểm tra object và S3 GetObject |
| Không có CloudWatch log          | Thiếu quyền logs hoặc config logger | Kiểm tra log policy             |
| Secret not found                 | Sai secret name                     | Kiểm tra `AWS_SECRET_NAME`      |
| Region mismatch                  | Service nằm khác region             | Đồng bộ `AWS_REGION`            |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* `aws sts get-caller-identity` hiển thị đúng IAM Role.
* Test S3 List/Put/Get thành công.
* Test SQS Send/Receive/Delete thành công.
* Worker gọi Textract thành công.
* Backend/worker ghi được log CloudWatch.
* Backend đọc được Secrets Manager nếu có dùng.
* Không còn lỗi `AccessDenied`.
* Pipeline upload → S3 → SQS → Textract → AI Analysis hoạt động.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã xác nhận IAM Role và IAM Policy hoạt động đúng. Backend và worker có đủ quyền cần thiết để vận hành pipeline xử lý tài liệu trên AWS mà không cần hardcode AWS Access Key trong production.
