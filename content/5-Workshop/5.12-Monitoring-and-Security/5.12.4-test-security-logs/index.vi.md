---

title : "Kiểm tra Security Logs"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.12.4. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ kiểm tra **Security Logs** cho hệ thống **DocuMind AI**.

Sau khi đã cấu hình CloudWatch Logs, Secrets Manager và AWS WAF, cần kiểm tra xem các sự kiện bảo mật có được ghi nhận đúng hay không. Các sự kiện này có thể bao gồm đăng nhập thất bại, truy cập trái quyền, upload file sai định dạng, AI provider lỗi, WAF phát hiện SQL Injection/XSS hoặc worker xử lý thất bại.

Security Logs giúp Admin phát hiện vấn đề sớm và hỗ trợ điều tra khi hệ thống có sự cố.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra log đăng nhập thất bại.
* Kiểm tra log truy cập trái quyền.
* Kiểm tra log upload file sai định dạng.
* Kiểm tra log lỗi worker.
* Kiểm tra log AI provider fallback.
* Kiểm tra WAF metrics và sampled requests.
* Đảm bảo log không chứa dữ liệu nhạy cảm.
* Chuẩn bị security monitoring cho Admin Panel.

---

#### Vai trò của Security Logs

Security Logs giúp theo dõi:

* Login failed.
* Forbidden access.
* Unauthorized request.
* Unsupported file upload.
* Suspicious payload.
* WAF blocked/counted request.
* AI provider failure.
* Secrets Manager access error.
* IAM AccessDenied.
* Worker processing failed.

---

#### Kiến trúc Security Logs

Luồng ghi nhận security event:

```text id="b5owud"
Frontend / User Request
  |
  v
AWS WAF
  |
  v
Backend Security Middleware
  |
  v
CloudWatch Logs
  |
  v
AuditLog / Admin Notification
  |
  v
Admin Panel
```

![test-security-logs](/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.4-test-security-logs/test-security-logs.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình CloudWatch Logs hoặc Admin Audit Log hiển thị các sự kiện như `SECURITY_FORBIDDEN_ACCESS`, `LOGIN_FAILED`, `UNSUPPORTED_FILE_UPLOAD` hoặc WAF `CountedRequests`.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.4-test-security-logs/test-security-logs.png`

---

#### Các security event nên kiểm tra

Các event đề xuất:

```text id="gyr3d3"
LOGIN_FAILED
UNAUTHORIZED_REQUEST
SECURITY_FORBIDDEN_ACCESS
UNSUPPORTED_FILE_UPLOAD
FILE_SIZE_EXCEEDED
AI_PROVIDER_FALLBACK_USED
DOCUMENT_PROCESSING_FAILED
IAM_ACCESS_DENIED
WAF_REQUEST_COUNTED
WAF_REQUEST_BLOCKED
```

---

#### Bước 1: Test login thất bại

Gọi API login với sai mật khẩu:

```text id="281kkh"
POST /api/auth/login
```

Body:

```json id="fzpc0i"
{
  "email": "demo@docmind.ai",
  "password": "wrong-password"
}
```

Response mong đợi:

```json id="ohs337"
{
  "success": false,
  "message": "Invalid email or password."
}
```

Log mong đợi:

```text id="3b2wml"
[LOGIN_FAILED]
```

Audit Log có thể ghi:

```text id="6c929e"
USER_LOGIN_FAILED
```

---

#### Bước 2: Test truy cập không có token

Gọi API cần đăng nhập nhưng không gửi JWT:

```text id="u91uu4"
GET /api/documents
```

Response mong đợi:

```json id="o4lvyh"
{
  "success": false,
  "message": "Unauthorized."
}
```

Log mong đợi:

```text id="o68dqt"
[UNAUTHORIZED_REQUEST]
```

---

#### Bước 3: Test truy cập document không thuộc quyền

Đăng nhập bằng User A, sau đó gọi document của User B:

```text id="l6yjxh"
GET /api/documents/doc_of_user_b/status
```

Response mong đợi:

```json id="y81s56"
{
  "success": false,
  "message": "You do not have permission to access this document."
}
```

Log mong đợi:

```text id="yngsmn"
[SECURITY_FORBIDDEN_ACCESS]
```

Audit Log nên lưu:

```json id="fov8yl"
{
  "action": "SECURITY_FORBIDDEN_ACCESS",
  "target": "Document",
  "severity": "warning",
  "metadata": {
    "documentId": "doc_of_user_b"
  }
}
```

---

#### Bước 4: Test upload file sai định dạng

Upload file không được hỗ trợ:

```text id="fuzmwl"
test.exe
test.docx
test.sh
```

Response mong đợi:

```json id="r5czc0"
{
  "success": false,
  "message": "Unsupported file type. Only PDF, PNG, JPG and JPEG are allowed."
}
```

Log mong đợi:

```text id="6k7re1"
[UNSUPPORTED_FILE_UPLOAD]
```

Nếu user upload sai nhiều lần, có thể tạo Admin Notification:

```text id="ld4xsm"
Multiple unsupported file uploads detected.
```

---

#### Bước 5: Test file quá lớn

Upload file vượt giới hạn, ví dụ hơn 10MB nếu hệ thống giới hạn 10MB.

Response mong đợi:

```json id="e4ndqk"
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

Log mong đợi:

```text id="0mt07c"
[FILE_SIZE_EXCEEDED]
```

---

#### Bước 6: Test AI Provider fallback

Tạm thời làm sai API key provider chính.

Ví dụ:

```env id="ms47th"
AI_PROVIDER="openai"
OPENAI_API_KEY="wrong-key"
```

Sau đó gọi AI Analysis.

Log mong đợi:

```text id="h3lofi"
[AI_PROVIDER_SELECTED] openai
[AI_PROVIDER_PRIMARY_FAILED] openai
[AI_PROVIDER_FALLBACK_USED] gemini
```

Admin Notification mong đợi:

```text id="xs72z7"
AI provider fallback used.
```

Audit Log có thể ghi:

```text id="twuddi"
AI_PROVIDER_FALLBACK_USED
```

---

#### Bước 7: Test worker processing failed

Gửi message vào SQS với `s3Key` sai:

```json id="5ztv1e"
{
  "documentId": "doc_error_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/not-found.pdf",
  "action": "PROCESS_DOCUMENT"
}
```

Worker log mong đợi:

```text id="v4vvlo"
[DOCUMENT_PROCESSING_FAILED]
[INVALID_S3_OBJECT]
```

Document status nên là:

```text id="6uqqjw"
FAILED
```

Admin Notification:

```text id="dfx46o"
Document processing failed.
```

---

#### Bước 8: Test WAF SQL Injection

Gửi request có SQLi pattern:

```text id="xlk3bp"
/api/auth/login?email=' OR '1'='1
```

Nếu WAF ở Count mode, request được ghi nhận trong metric:

```text id="jgx40q"
CountedRequests
```

Nếu WAF ở Block mode, request có thể bị chặn và metric tăng:

```text id="kdeuux"
BlockedRequests
```

Kiểm tra tại:

```text id="sjsl6x"
CloudWatch
→ Metrics
→ AWS/WAFV2
```

---

#### Bước 9: Test WAF XSS

Gửi payload:

```text id="guxpom"
<script>alert('xss')</script>
```

Ví dụ trong AI Chat:

```json id="qr96tp"
{
  "question": "<script>alert('xss')</script>"
}
```

Kiểm tra WAF sampled requests hoặc metrics.

---

#### Bước 10: Kiểm tra CloudWatch Logs

Vào:

```text id="0sbuj2"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Tìm các event:

```text id="d07q5a"
LOGIN_FAILED
UNAUTHORIZED_REQUEST
SECURITY_FORBIDDEN_ACCESS
UNSUPPORTED_FILE_UPLOAD
AI_PROVIDER_FALLBACK_USED
DOCUMENT_PROCESSING_FAILED
```

---

#### Bước 11: Kiểm tra Audit Log trong Admin Panel

Admin Panel nên hiển thị:

| Time       | Action                     | Severity | Target   | Detail |
| ---------- | -------------------------- | -------- | -------- | ------ |
| 2026-06-10 | SECURITY_FORBIDDEN_ACCESS  | warning  | Document | View   |
| 2026-06-10 | DOCUMENT_PROCESSING_FAILED | error    | Document | View   |
| 2026-06-10 | AI_PROVIDER_FALLBACK_USED  | warning  | AI       | View   |

---

#### Bước 12: Kiểm tra không lộ dữ liệu nhạy cảm

Kiểm tra CloudWatch và Audit Log không chứa:

```text id="8icawt"
password
JWT token
OPENAI_API_KEY
GEMINI_API_KEY
DATABASE_URL full password
AWS credentials
full document content
```

Nếu có, cần chỉnh logger để mask hoặc loại bỏ field nhạy cảm.

---

#### Các lỗi thường gặp

| Lỗi                        | Nguyên nhân                         | Cách xử lý                     |
| -------------------------- | ----------------------------------- | ------------------------------ |
| Không có security log      | Chưa gọi logger/audit service       | Kiểm tra middleware            |
| Forbidden không ghi audit  | Thiếu log trong authorization guard | Thêm audit log                 |
| WAF không có metric        | Chưa bật CloudWatch metrics         | Kiểm tra Web ACL visibility    |
| WAF không bắt request      | Chưa attach CloudFront/ALB          | Kiểm tra Associated resources  |
| Log lộ token/key           | Logger log toàn bộ request          | Mask sensitive fields          |
| Admin không thấy alert     | Chưa tạo admin notification         | Kiểm tra notification service  |
| Worker lỗi nhưng không log | Catch error thiếu logger            | Kiểm tra worker error handling |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Login failed được ghi log.
* Unauthorized request được ghi log.
* Forbidden access được ghi Audit Log.
* Upload file sai định dạng được ghi log.
* File quá lớn được ghi log.
* AI provider fallback được ghi log và tạo admin notification.
* Worker processing failed được ghi log.
* WAF SQLi/XSS test có metric Counted hoặc Blocked.
* CloudWatch Logs hiển thị security events.
* Admin Panel hiển thị Audit Log và Admin Notifications.
* Không có secret/token/password trong log.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có khả năng ghi nhận và kiểm tra các security logs quan trọng. Admin có thể theo dõi lỗi, truy cập trái quyền, request bất thường, WAF metrics và các cảnh báo hệ thống để vận hành nền tảng an toàn hơn.
