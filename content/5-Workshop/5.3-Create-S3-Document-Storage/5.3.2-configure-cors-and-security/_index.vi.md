---

title : "Cấu hình CORS và bảo mật S3"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.3.2. </b> "
-------------------------

#### Tổng quan

Sau khi tạo S3 bucket, bước tiếp theo là cấu hình **CORS** và các thiết lập bảo mật cần thiết cho bucket.

Trong dự án **DocuMind AI**, tài liệu người dùng có thể được upload thông qua backend hoặc thông qua pre-signed URL. Vì vậy, S3 cần được cấu hình đúng để frontend/backend có thể upload file hợp lệ, đồng thời vẫn đảm bảo tài liệu không bị public ngoài ý muốn.

CORS giúp trình duyệt cho phép frontend gọi đến S3 trong trường hợp upload trực tiếp từ web. Tuy nhiên, dù có cấu hình CORS, bucket vẫn phải giữ ở chế độ private và quyền truy cập nên được kiểm soát thông qua backend hoặc IAM Role.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Cấu hình CORS cho S3 bucket.
* Giữ bucket ở trạng thái private.
* Bật Block Public Access.
* Hiểu cách bảo vệ tài liệu người dùng.
* Chuẩn bị IAM Policy cho backend/worker truy cập S3.
* Xác định các định dạng file được phép upload.
* Tránh lỗi CORS khi frontend gọi upload file.

---

#### CORS là gì?

**CORS** là viết tắt của **Cross-Origin Resource Sharing**. Đây là cơ chế cho phép một website ở domain này được gọi tài nguyên từ domain khác.

Ví dụ:

```text
Frontend: http://localhost:5173
S3 Bucket: https://documind-document-storage.s3.ap-southeast-1.amazonaws.com
```

Nếu frontend upload file trực tiếp lên S3 mà chưa cấu hình CORS, trình duyệt có thể báo lỗi:

```text
Access to fetch at ... from origin ... has been blocked by CORS policy
```

Trong trường hợp backend nhận file rồi upload lên S3, lỗi CORS với S3 ít xảy ra hơn. Tuy nhiên, cấu hình CORS vẫn hữu ích nếu hệ thống dùng pre-signed URL.

---

#### Kiến trúc CORS và bảo mật

Luồng upload an toàn trong DocuMind AI:

```text
React Frontend
  |
  v
Backend API kiểm tra user và file
  |
  v
Amazon S3 Bucket private
  |
  v
PostgreSQL lưu metadata
```

Hoặc nếu dùng pre-signed URL:

```text
React Frontend
  |
  v
Backend API tạo pre-signed URL
  |
  v
Frontend upload trực tiếp lên S3
  |
  v
Backend lưu metadata và gửi job SQS
```

![s3-cors-security](/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.2-configure-cors-and-security/s3-cors-security.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình phần CORS configuration trong S3 bucket hoặc vẽ sơ đồ minh họa bucket private, backend kiểm soát quyền upload và frontend gọi API.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.2-configure-cors-and-security/s3-cors-security.png`

---

#### Bước 1: Mở S3 bucket

Vào AWS Console:

```text
Amazon S3
```

Chọn bucket của dự án, ví dụ:

```text
documind-document-storage
```

Sau đó chọn tab:

```text
Permissions
```

---

#### Bước 2: Kiểm tra Block Public Access

Trong tab **Permissions**, tìm phần:

```text
Block public access
```

Đảm bảo trạng thái là:

```text
On
```

Hoặc:

```text
Block all public access: Enabled
```

Nếu đang tắt, hãy bật lại toàn bộ các lựa chọn Block Public Access.

Cấu hình đúng:

```text
Block public access to buckets and objects granted through new access control lists (ACLs): On
Block public access to buckets and objects granted through any access control lists (ACLs): On
Block public access to buckets and objects granted through new public bucket policies: On
Block public and cross-account access to buckets and objects through any public bucket policies: On
```

---

#### Bước 3: Kiểm tra Bucket Policy

Với DocuMind AI, bucket không nên có policy public.

Không nên có policy dạng:

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::documind-document-storage/*"
}
```

Policy trên sẽ làm file có nguy cơ bị public.

Thay vào đó, quyền truy cập S3 nên được cấp cho backend/worker thông qua IAM Role hoặc IAM User trong môi trường local.

---

#### Bước 4: Cấu hình CORS

Trong tab **Permissions**, kéo xuống phần:

```text
Cross-origin resource sharing (CORS)
```

Nhấn:

```text
Edit
```

Thêm cấu hình CORS mẫu cho môi trường development:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": [
      "http://localhost:5173",
      "http://localhost:3000"
    ],
    "ExposeHeaders": ["ETag"]
  }
]
```

Sau đó nhấn:

```text
Save changes
```

---

#### CORS cho môi trường production

Khi deploy production, không nên để origin quá rộng.

Ví dụ production:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": [
      "https://your-documind-domain.com"
    ],
    "ExposeHeaders": ["ETag"]
  }
]
```

Không nên dùng:

```json
"AllowedOrigins": ["*"]
```

cho môi trường production nếu hệ thống xử lý tài liệu riêng tư.

---

#### Bước 5: Cấu hình file upload rule trong backend

Ngoài S3 CORS, backend vẫn cần kiểm tra file upload.

Các định dạng nên cho phép:

```text
.pdf
.png
.jpg
.jpeg
```

MIME type tương ứng:

```text
application/pdf
image/png
image/jpeg
```

Không nên cho phép:

```text
.exe
.sh
.bat
.js
.docx
.xlsx
.pptx
```

vì Amazon Textract không hỗ trợ trực tiếp một số định dạng văn phòng như DOCX, XLSX hoặc PPTX.

---

#### Bước 6: Giới hạn dung lượng file

Backend nên giới hạn dung lượng file upload để tránh người dùng tải file quá lớn gây tốn tài nguyên.

Ví dụ:

```text
Max file size: 10MB
```

Hoặc nếu cần OCR tài liệu lớn hơn:

```text
Max file size: 20MB
```

Khi file vượt quá giới hạn, backend nên trả về lỗi rõ ràng:

```json
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

---

#### Bước 7: Chuẩn bị IAM Policy cho S3

Backend hoặc worker cần quyền truy cập bucket.

Các quyền tối thiểu thường cần:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

Ví dụ policy tham khảo:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindS3ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::documind-document-storage/*"
    },
    {
      "Sid": "DocuMindS3ListBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::documind-document-storage"
    }
  ]
}
```

> ⚠️ **Lưu ý:**
> Nếu bucket của bạn có tên khác, hãy thay `documind-document-storage` bằng tên bucket thật.

---

#### Bước 8: Không hardcode AWS credentials

Trong môi trường local, bạn có thể dùng:

```bash
aws configure
```

để test nhanh.

Nhưng khi deploy production lên EC2/backend server, nên dùng:

```text
IAM Role
```

Không nên lưu các biến sau trong `.env` production:

```env
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Thay vào đó, backend dùng AWS SDK default credential chain để tự lấy quyền từ IAM Role gắn với EC2.

---

#### Bước 9: Kiểm tra CORS

Sau khi cấu hình CORS, hãy test upload từ frontend hoặc Postman.

Nếu lỗi CORS xảy ra, trình duyệt thường báo:

```text
CORS policy: No 'Access-Control-Allow-Origin' header is present
```

Cách kiểm tra:

* Kiểm tra lại `AllowedOrigins`.
* Đảm bảo frontend đang chạy đúng port.
* Đảm bảo method upload là `PUT` hoặc `POST` có trong `AllowedMethods`.
* Đảm bảo request không dùng header bị chặn.
* Nếu upload qua backend, kiểm tra CORS backend Express thay vì CORS S3.

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Bucket vẫn đang private.
* Block Public Access đang bật.
* Bucket không có public policy.
* CORS đã thêm origin local hoặc domain production.
* Backend có rule kiểm tra file type.
* Backend có rule giới hạn file size.
* IAM Policy cho S3 đã được chuẩn bị.
* Không hardcode AWS credentials trong production.

---

#### Kết quả kỳ vọng

Sau bước này, S3 bucket của DocuMind AI đã được cấu hình an toàn hơn và sẵn sàng cho quá trình upload tài liệu. Hệ thống có thể tránh lỗi CORS khi frontend upload file, đồng thời vẫn đảm bảo tài liệu người dùng không bị public ra ngoài.
