---

title : "Tạo Dead Letter Queue"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.4.2. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **Dead Letter Queue (DLQ)** cho hàng đợi xử lý tài liệu của **DocuMind AI**.

Dead Letter Queue được sử dụng để lưu các message xử lý thất bại nhiều lần. Ví dụ, nếu worker nhận message từ SQS nhưng liên tục xử lý lỗi do file không tồn tại trên S3, thiếu quyền Textract hoặc dữ liệu message không hợp lệ, message đó sẽ được chuyển sang DLQ sau số lần retry nhất định.

DLQ giúp hệ thống dễ theo dõi lỗi, tránh việc một message lỗi bị retry vô hạn và làm ảnh hưởng đến pipeline xử lý tài liệu.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được Dead Letter Queue cho DocuMind AI.
* Liên kết DLQ với main processing queue.
* Cấu hình số lần retry trước khi message bị chuyển sang DLQ.
* Hiểu cách kiểm tra message lỗi trong DLQ.
* Chuẩn bị cơ chế xử lý lỗi cho worker OCR/AI.

---

#### Vai trò của Dead Letter Queue trong DocuMind AI

Trong hệ thống DocuMind AI, DLQ được dùng để:

* Lưu message xử lý thất bại nhiều lần.
* Hỗ trợ debug lỗi OCR, S3, Textract hoặc AI provider.
* Tránh retry vô hạn các message lỗi.
* Giúp Admin hoặc Developer kiểm tra lại tài liệu bị lỗi.
* Tăng độ ổn định cho pipeline xử lý bất đồng bộ.

Ví dụ lỗi có thể đưa message vào DLQ:

* File không tồn tại trong S3.
* Worker thiếu quyền `s3:GetObject`.
* Worker thiếu quyền gọi Amazon Textract.
* Message thiếu `documentId` hoặc `s3Key`.
* File không đúng định dạng OCR.
* AI provider bị lỗi liên tục.
* Worker crash trước khi xử lý xong.

---

#### Kiến trúc DLQ

Luồng xử lý với Dead Letter Queue:

```text
Main SQS Queue
  |
  v
Worker xử lý message
  |
  |-- Success --> Delete message khỏi queue
  |
  |-- Failed nhiều lần --> Dead Letter Queue
```

![dead-letter-queue](/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.2-create-dead-letter-queue/dead-letter-queue.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình cấu hình Redrive policy của SQS hoặc vẽ sơ đồ Main Queue → Worker → Dead Letter Queue khi message xử lý thất bại.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.2-create-dead-letter-queue/dead-letter-queue.png`

---

#### Bước 1: Tạo Dead Letter Queue

Vào AWS Console:

```text
Amazon SQS
```

Chọn:

```text
Create queue
```

Chọn loại queue:

```text
Standard
```

Đặt tên DLQ:

```text
docmind-document-processing-dlq
```

Cấu hình khuyến nghị:

| Thành phần                | Cấu hình đề xuất    |
| ------------------------- | ------------------- |
| Queue type                | Standard            |
| Message retention period  | 4 days hoặc 14 days |
| Receive message wait time | 20 seconds          |
| Encryption                | SSE-SQS             |

Vì DLQ dùng để giữ message lỗi phục vụ debug, bạn có thể đặt retention dài hơn main queue nếu cần.

---

#### Bước 2: Lưu lại DLQ ARN

Sau khi tạo DLQ thành công, mở trang chi tiết queue và copy:

```text
ARN
```

Ví dụ:

```text
arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-dlq
```

ARN này sẽ dùng để cấu hình Redrive policy cho main queue.

---

#### Bước 3: Gắn DLQ vào main queue

Mở main queue:

```text
docmind-document-processing-queue
```

Chọn tab hoặc phần:

```text
Dead-letter queue
```

Chọn:

```text
Edit
```

Bật:

```text
Enable
```

Chọn DLQ vừa tạo:

```text
docmind-document-processing-dlq
```

---

#### Bước 4: Cấu hình Maximum receives

Ở phần **Maximum receives**, đặt số lần retry trước khi message bị chuyển sang DLQ.

Khuyến nghị:

```text
3
```

Điều này nghĩa là nếu một message được worker nhận và xử lý thất bại 3 lần, message đó sẽ được chuyển sang Dead Letter Queue.

Cấu hình:

```text
Maximum receives: 3
```

---

#### Bước 5: Cập nhật biến môi trường nếu cần

Nếu backend hoặc admin dashboard cần hiển thị DLQ status, thêm Queue URL vào `.env`:

```env
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

Nếu worker chỉ cần xử lý main queue, biến này có thể chưa cần dùng ngay trong demo.

---

#### Bước 6: Cập nhật quy trình xử lý lỗi trong worker

Khi worker xử lý message thất bại, không nên xóa message khỏi queue ngay. Nếu worker không gọi `DeleteMessage`, message sẽ quay lại main queue sau khi hết visibility timeout.

Pseudo flow:

```text
Receive message
  |
  v
Process OCR/AI
  |
  |-- Success: DeleteMessage
  |
  |-- Failed: Do not delete message
  |
  v
SQS retry
```

Sau số lần retry vượt quá `Maximum receives`, SQS sẽ tự chuyển message sang DLQ.

---

#### Bước 7: Kiểm tra DLQ trên AWS Console

Trong SQS Console, chọn DLQ:

```text
docmind-document-processing-dlq
```

Kiểm tra các thông tin:

* Available messages
* Messages in flight
* Oldest message age
* Queue URL
* ARN

Nếu có message trong DLQ, nghĩa là có job xử lý thất bại nhiều lần và cần kiểm tra lại nguyên nhân.

---

#### Message lỗi ví dụ

Một message lỗi trong DLQ có thể có dạng:

```json
{
  "documentId": "doc_error_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_error_001/missing-file.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Nguyên nhân có thể là `s3Key` không tồn tại hoặc worker không có quyền lấy file từ S3.

---

#### Quyền IAM liên quan

Backend/worker thường cần quyền với main queue:

```text
sqs:SendMessage
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

Nếu admin dashboard cần đọc DLQ, có thể cần thêm:

```text
sqs:GetQueueAttributes
sqs:ReceiveMessage
```

Không nên cho phép xóa message DLQ tùy tiện nếu chưa có cơ chế kiểm tra hoặc audit log.

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo Dead Letter Queue.
* DLQ cùng loại với main queue.
* DLQ đã bật encryption.
* Main queue đã gắn Redrive policy.
* Maximum receives được cấu hình hợp lý.
* Biết cách kiểm tra message lỗi trong DLQ.
* Worker được thiết kế không xóa message khi xử lý thất bại.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có cơ chế lưu lại các job xử lý tài liệu bị lỗi nhiều lần. Điều này giúp hệ thống dễ debug, tránh retry vô hạn và tăng độ tin cậy cho pipeline xử lý OCR/AI.
