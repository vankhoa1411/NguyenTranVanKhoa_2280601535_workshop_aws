---

title : "Tạo SQS Queue"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.4.1. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **Amazon SQS Queue** để làm hàng đợi xử lý tài liệu cho hệ thống **DocuMind AI**.

Sau khi người dùng upload tài liệu thành công lên **Amazon S3**, backend sẽ gửi một message vào SQS. Message này chứa các thông tin như `documentId`, `userId`, `s3Bucket`, `s3Key` và loại tác vụ cần xử lý. Worker sẽ đọc message từ queue để tiếp tục xử lý OCR bằng **Amazon Textract** và phân tích tài liệu bằng **Gemini/OpenAI** ở các bước sau.

Việc sử dụng SQS giúp hệ thống tách biệt phần upload tài liệu và phần xử lý tài liệu nền, từ đó giúp backend phản hồi nhanh hơn và hệ thống ổn định hơn khi có nhiều tài liệu được upload cùng lúc.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được một SQS Standard Queue cho DocuMind AI.
* Hiểu vai trò của SQS trong pipeline xử lý tài liệu.
* Cấu hình queue phù hợp cho worker xử lý OCR/AI.
* Lấy được Queue URL để cấu hình trong backend.
* Chuẩn bị cho bước tạo Dead Letter Queue ở phần tiếp theo.

---

#### Vai trò của SQS Queue trong DocuMind AI

Trong hệ thống DocuMind AI, SQS Queue được dùng để:

* Nhận job xử lý tài liệu sau khi upload lên S3.
* Truyền message từ Backend API sang Worker.
* Giúp xử lý OCR và AI theo cơ chế bất đồng bộ.
* Giảm thời gian chờ của người dùng khi upload tài liệu.
* Cho phép worker xử lý tài liệu theo hàng đợi.
* Hỗ trợ retry khi quá trình xử lý gặp lỗi tạm thời.

Luồng xử lý trong bước này:

```text
Backend API
  |
  v
Amazon SQS Queue
  |
  v
Worker xử lý nền
```

![create-sqs-queue](/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.1-create-sqs-queue/create-sqs-queue.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình trang tạo SQS Queue trên AWS Console hoặc vẽ sơ đồ Backend API gửi message vào SQS và Worker nhận message từ SQS.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.1-create-sqs-queue/create-sqs-queue.png`

---

#### Bước 1: Truy cập Amazon SQS

Đăng nhập vào AWS Console, sau đó tìm kiếm dịch vụ:

```text
SQS
```

Chọn:

```text
Amazon Simple Queue Service
```

Sau đó nhấn:

```text
Create queue
```

---

#### Bước 2: Chọn loại queue

Ở phần **Type**, chọn:

```text
Standard
```

Với DocuMind AI, **Standard Queue** là lựa chọn phù hợp vì mỗi tài liệu có `documentId` riêng và hệ thống không yêu cầu xử lý đúng thứ tự tuyệt đối.

Không cần chọn FIFO Queue trong workshop này, trừ khi hệ thống yêu cầu xử lý theo đúng thứ tự từng tài liệu hoặc từng người dùng.

---

#### Bước 3: Đặt tên queue

Đặt tên queue theo dự án:

```text
docmind-document-processing-queue
```

Hoặc có thể dùng tên khác:

```text
documind-document-processing-queue
docmind-ai-processing-queue
docmind-ocr-analysis-queue
```

Tên queue nên thể hiện rõ mục đích là xử lý tài liệu, OCR và AI analysis.

---

#### Bước 4: Cấu hình queue

Cấu hình khuyến nghị:

| Thành phần                | Cấu hình đề xuất |
| ------------------------- | ---------------- |
| Visibility timeout        | 300 seconds      |
| Message retention period  | 4 days           |
| Delivery delay            | 0 seconds        |
| Receive message wait time | 20 seconds       |
| Maximum message size      | 256 KB           |
| Encryption                | SSE-SQS          |

Giải thích nhanh:

* **Visibility timeout**: Thời gian message bị ẩn sau khi worker nhận. Nếu worker xử lý lâu, nên đặt đủ dài để tránh message bị xử lý lặp.
* **Message retention period**: Thời gian giữ message trong queue nếu chưa được xử lý.
* **Receive message wait time**: Nên đặt 20 giây để bật long polling, giúp giảm số request rỗng.
* **Encryption**: Nên bật SSE-SQS để bảo vệ message trong queue.

---

#### Bước 5: Cấu hình Access Policy

Ở môi trường demo, có thể để mặc định:

```text
Only the queue owner
```

Điều này nghĩa là chỉ tài khoản AWS đang sở hữu queue mới có quyền thao tác queue.

Trong production, quyền gửi và nhận message nên được quản lý bằng **IAM Role and Policy**. Backend cần quyền `sqs:SendMessage`, còn worker cần quyền `sqs:ReceiveMessage`, `sqs:DeleteMessage` và `sqs:GetQueueAttributes`.

---

#### Bước 6: Tạo queue

Sau khi kiểm tra cấu hình, nhấn:

```text
Create queue
```

Khi tạo thành công, AWS sẽ chuyển bạn đến trang chi tiết của queue.

---

#### Bước 7: Lấy Queue URL

Trong trang chi tiết của queue, tìm mục:

```text
URL
```

Copy Queue URL, ví dụ:

```text
https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue
```

Thêm vào file `.env` backend:

```env
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
```

---

#### Bước 8: Chuẩn bị message format

Message gửi vào queue nên có cấu trúc rõ ràng:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Worker sẽ dựa vào `documentId` và `s3Key` để lấy tài liệu từ S3, xử lý OCR và cập nhật kết quả vào PostgreSQL.

---

#### Bước 9: Kiểm tra queue sau khi tạo

Trong trang chi tiết của queue, kiểm tra:

| Thành phần                | Trạng thái mong muốn |
| ------------------------- | -------------------- |
| Queue type                | Standard             |
| Visibility timeout        | 300 seconds          |
| Encryption                | Enabled              |
| Receive message wait time | 20 seconds           |
| Queue URL                 | Đã copy vào `.env`   |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo SQS Standard Queue.
* Queue có tên đúng với dự án.
* Queue URL đã được copy vào `.env`.
* Queue đã bật encryption.
* Queue đã bật long polling.
* Backend đã sẵn sàng gửi message vào queue.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có hàng đợi trung gian để tiếp nhận job xử lý tài liệu. Đây là nền tảng để worker có thể nhận message, lấy tài liệu từ S3 và xử lý OCR bằng Amazon Textract trong các bước tiếp theo.
