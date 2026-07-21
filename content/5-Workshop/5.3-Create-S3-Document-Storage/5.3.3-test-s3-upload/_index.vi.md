---

title : "Kiểm tra upload tài liệu lên S3"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.3.3. </b> "
-------------------------

#### Tổng quan

Sau khi đã tạo S3 bucket và cấu hình CORS/bảo mật, bước tiếp theo là kiểm tra chức năng **upload tài liệu lên Amazon S3** cho hệ thống **DocuMind AI**.

Ở bước này, chúng ta sẽ kiểm tra xem backend có thể nhận file từ người dùng, upload file lên S3, lưu metadata vào **PostgreSQL + Prisma**, và chuẩn bị tạo job xử lý tài liệu cho **Amazon SQS** hay chưa.

Đây là bước quan trọng vì nếu upload file lên S3 không thành công thì các bước phía sau như **SQS Processing Queue**, **Textract OCR**, **Gemini/OpenAI AI Analysis**, **Notification** và **Dashboard** sẽ không thể hoạt động đúng.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra backend có thể upload file lên S3.
* Xác nhận file xuất hiện trong S3 bucket.
* Kiểm tra `s3Key` được tạo đúng.
* Kiểm tra metadata tài liệu được lưu vào PostgreSQL.
* Kiểm tra trạng thái tài liệu ban đầu là `UPLOADED`, `PENDING` hoặc `QUEUED`.
* Kiểm tra lỗi khi upload file sai định dạng.
* Chuẩn bị cho bước gửi message vào SQS ở phần 5.4.

---

#### Luồng kiểm tra upload

Luồng upload tài liệu trong DocuMind AI:

```text
User
  |
  v
React Frontend / Postman
  |
  v
Backend API: POST /api/documents/upload
  |
  v
Validate file type + file size
  |
  v
Upload file to Amazon S3
  |
  v
Save metadata to PostgreSQL
  |
  v
Return document response to frontend
```

![test-s3-upload](/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.3-test-s3-upload/test-s3-upload.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình kết quả upload file thành công từ Postman hoặc frontend, kèm ảnh file đã xuất hiện trong S3 bucket.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.3-test-s3-upload/test-s3-upload.png`

---

#### Điều kiện trước khi kiểm tra

Trước khi test upload, hãy đảm bảo bạn đã hoàn thành:

* Đã tạo S3 bucket.
* Bucket đang bật Block Public Access.
* Bucket đã cấu hình encryption.
* Đã cấu hình CORS nếu frontend upload trực tiếp hoặc dùng pre-signed URL.
* Backend đã có biến môi trường `AWS_REGION`.
* Backend đã có biến môi trường `AWS_S3_BUCKET_NAME`.
* Local đã cấu hình AWS CLI hoặc backend đang chạy với IAM Role.
* PostgreSQL và Prisma đã sẵn sàng.
* API upload tài liệu đã được tạo.

Ví dụ `.env` backend:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"

DATABASE_URL="postgresql://username:password@localhost:5432/docmind"
```

Nếu dùng OpenAI/Gemini ở bước sau, bạn có thể chuẩn bị thêm:

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

---

#### Bước 1: Chuẩn bị file test

Chuẩn bị một file hợp lệ để test upload.

Các định dạng nên dùng:

```text
PDF
PNG
JPG
JPEG
```

Ví dụ file test:

```text
invoice-demo.pdf
receipt-demo.jpg
contract-demo.pdf
```

Không nên dùng các file sau để test bước này nếu hệ thống chưa hỗ trợ chuyển đổi:

```text
DOCX
XLSX
PPTX
EXE
SH
BAT
```

Với DocuMind AI, các file như PDF hoặc ảnh sẽ phù hợp hơn vì các bước sau có thể xử lý bằng **Amazon Textract**.

---

#### Bước 2: Khởi động backend

Mở terminal tại thư mục backend và chạy:

```bash
npm install
```

Sau đó khởi động server:

```bash
npm run dev
```

Hoặc nếu dự án dùng lệnh khác:

```bash
npm start
```

Kiểm tra backend đang chạy, ví dụ:

```text
Server is running on port 3000
```

API upload dự kiến:

```text
POST http://localhost:3000/api/documents/upload
```

---

#### Bước 3: Test upload bằng Postman

Mở Postman và tạo request mới:

```text
Method: POST
URL: http://localhost:3000/api/documents/upload
```

Trong tab **Body**, chọn:

```text
form-data
```

Thêm field:

| Key  | Type | Value            |
| ---- | ---- | ---------------- |
| file | File | invoice-demo.pdf |

Nếu API yêu cầu đăng nhập, thêm token vào tab **Authorization**:

```text
Bearer Token
```

Hoặc thêm header:

```text
Authorization: Bearer your_jwt_token
```

Sau đó nhấn:

```text
Send
```

---

#### Bước 4: Kết quả API mong đợi

Nếu upload thành công, API có thể trả về response tương tự:

```json
{
  "success": true,
  "message": "Document uploaded successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "fileType": "application/pdf",
    "fileSize": 204800,
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "UPLOADED",
    "createdAt": "2026-06-10T10:00:00.000Z"
  }
}
```

Một số hệ thống có thể dùng trạng thái ban đầu là:

```text
PENDING
```

hoặc:

```text
QUEUED
```

Điều này vẫn hợp lệ nếu backend của bạn tiếp tục gửi job sang SQS sau khi upload.

---

#### Bước 5: Kiểm tra file trong Amazon S3

Vào AWS Console:

```text
Amazon S3
```

Chọn bucket của dự án:

```text
documind-document-storage
```

Mở folder object tương ứng, ví dụ:

```text
uploads/user_001/doc_001/
```

Kiểm tra file đã xuất hiện:

```text
invoice-demo.pdf
```

Kiểm tra thêm các thông tin:

* Object key
* Size
* Last modified
* Encryption
* Object URL

Lưu ý: Object URL có thể tồn tại nhưng không có nghĩa là file public. Nếu bucket private, truy cập trực tiếp Object URL sẽ bị từ chối, đây là hành vi đúng.

---

#### Bước 6: Kiểm tra metadata trong PostgreSQL

Sau khi upload thành công, kiểm tra bảng `Document` hoặc bảng tương ứng trong PostgreSQL.

Ví dụ dữ liệu mong đợi:

```text
id: doc_001
userId: user_001
fileName: invoice-demo.pdf
fileType: application/pdf
fileSize: 204800
s3Bucket: documind-document-storage
s3Key: uploads/user_001/doc_001/invoice-demo.pdf
status: UPLOADED
createdAt: 2026-06-10
```

Nếu dùng Prisma Studio, chạy:

```bash
npx prisma studio
```

Sau đó mở trình duyệt và kiểm tra bảng `Document`.

---

#### Bước 7: Kiểm tra log backend

Trong terminal backend, kiểm tra log sau khi upload.

Log mong đợi:

```text
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[DOCUMENT_METADATA_SAVED]
```

Nếu đã tích hợp CloudWatch, có thể kiểm tra thêm log group:

```text
/docmind/application
```

Không nên log toàn bộ nội dung tài liệu hoặc thông tin nhạy cảm.

---

#### Bước 8: Kiểm tra lỗi file sai định dạng

Upload thử một file không hợp lệ, ví dụ:

```text
test.docx
script.sh
malware.exe
```

Backend nên trả về lỗi rõ ràng:

```json
{
  "success": false,
  "message": "Unsupported file type. Only PDF, PNG, JPG and JPEG are allowed."
}
```

Việc chặn file sai định dạng giúp tránh lỗi ở bước Amazon Textract, vì Textract không hỗ trợ trực tiếp một số định dạng như DOCX, XLSX hoặc PPTX.

---

#### Bước 9: Kiểm tra lỗi file quá lớn

Upload thử một file vượt quá dung lượng cho phép.

Ví dụ nếu backend giới hạn 10MB, upload file lớn hơn 10MB.

Response mong đợi:

```json
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

Backend nên kiểm tra dung lượng trước khi upload lên S3 để tránh tốn tài nguyên không cần thiết.

---

#### Bước 10: Kiểm tra quyền truy cập S3

Nếu upload thất bại do quyền AWS, bạn có thể gặp lỗi:

```text
AccessDenied
```

hoặc:

```text
User is not authorized to perform: s3:PutObject
```

Cách kiểm tra:

* Đảm bảo AWS credentials local đúng nếu chạy local.
* Đảm bảo IAM Role có quyền `s3:PutObject`.
* Đảm bảo policy trỏ đúng tên bucket.
* Đảm bảo `AWS_REGION` đúng.
* Đảm bảo `AWS_S3_BUCKET_NAME` đúng.

Quyền tối thiểu cần có:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

---

#### Bước 11: Kiểm tra chuẩn bị gửi message sang SQS

Ở phần này, mục tiêu chính là upload thành công lên S3. Tuy nhiên, sau khi upload, backend nên sẵn sàng gửi message sang SQS ở phần tiếp theo.

Message dự kiến cho SQS:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT"
}
```

Nếu backend hiện tại chưa gửi SQS ngay, có thể để bước đó sang phần **5.4 - Create SQS Processing Queue**.

---

#### Các lỗi thường gặp

| Lỗi                     | Nguyên nhân                             | Cách xử lý                                   |
| ----------------------- | --------------------------------------- | -------------------------------------------- |
| `AccessDenied`          | Thiếu quyền S3                          | Kiểm tra IAM Policy hoặc IAM Role            |
| `NoSuchBucket`          | Sai tên bucket                          | Kiểm tra `AWS_S3_BUCKET_NAME`                |
| `PermanentRedirect`     | Sai region bucket                       | Kiểm tra `AWS_REGION`                        |
| `CORS policy error`     | Origin chưa được allow                  | Kiểm tra CORS trong S3 hoặc backend          |
| `File too large`        | File vượt giới hạn                      | Giảm dung lượng file hoặc tăng limit backend |
| `Unsupported file type` | File không nằm trong định dạng cho phép | Dùng PDF, PNG, JPG hoặc JPEG                 |
| `Unauthorized`          | Thiếu JWT token                         | Đăng nhập và gửi Authorization header        |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Upload file PDF hoặc ảnh thành công từ Postman/frontend.
* File xuất hiện trong S3 bucket.
* Object key được tạo đúng cấu trúc.
* Metadata được lưu trong PostgreSQL.
* Backend trả về response có `documentId`, `s3Bucket`, `s3Key` và `status`.
* File sai định dạng bị chặn.
* File quá lớn bị chặn.
* Không có lỗi `AccessDenied`, `NoSuchBucket` hoặc sai region.
* Hệ thống đã sẵn sàng chuyển sang phần 5.4 để tạo SQS queue.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có thể upload và lưu trữ tài liệu thành công trên Amazon S3. File tài liệu được quản lý bằng `s3Key`, metadata được lưu trong PostgreSQL và hệ thống đã sẵn sàng đưa tài liệu vào pipeline xử lý bất đồng bộ bằng Amazon SQS, Amazon Textract và AI Gateway.
