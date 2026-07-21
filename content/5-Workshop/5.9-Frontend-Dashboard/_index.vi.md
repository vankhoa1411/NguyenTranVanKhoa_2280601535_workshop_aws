---

title : "Frontend Dashboard"
date : 2026-06-10
weight : 9
chapter : false
pre : " <b> 5.9. </b> "
-----------------------

#### Xây dựng Frontend Dashboard

Trong phần này, chúng ta sẽ xây dựng **Frontend Dashboard** cho hệ thống **DocuMind AI**.

Frontend Dashboard là giao diện chính giúp người dùng tương tác với hệ thống. Người dùng có thể đăng nhập, upload tài liệu, theo dõi trạng thái xử lý, xem kết quả OCR, xem kết quả AI Analysis, chat với tài liệu và thử nghiệm các chức năng nâng cao trong Enterprise Sandbox.

Sau khi backend đã có API upload tài liệu, SQS Worker, Textract OCR, AI Gateway và PostgreSQL + Prisma, frontend sẽ đóng vai trò hiển thị toàn bộ pipeline xử lý tài liệu theo cách trực quan và dễ sử dụng.

---

#### Vai trò của Frontend trong DocuMind AI

Trong dự án **DocuMind AI**, frontend có nhiệm vụ:

* Hiển thị Home Page giới thiệu hệ thống.
* Cho phép người dùng đăng ký và đăng nhập.
* Hiển thị trạng thái đăng nhập, avatar và thông tin người dùng.
* Cho phép upload tài liệu PDF, PNG, JPG hoặc JPEG.
* Hiển thị danh sách tài liệu đã upload.
* Theo dõi trạng thái xử lý tài liệu.
* Hiển thị OCR Result từ Amazon Textract.
* Hiển thị AI Analysis từ Gemini/OpenAI.
* Hỗ trợ AI Chat/RAG với tài liệu.
* Cung cấp Enterprise Sandbox để test semantic search, entities và analytics.
* Hỗ trợ phân quyền giao diện giữa User và Admin.

---

#### Kiến trúc Frontend Dashboard

Luồng kết nối tổng thể:

```text
React Frontend
  |
  v
Backend API Node.js/Express
  |
  |-- Auth API
  |-- Document Upload API
  |-- Document Status API
  |-- OCR Result API
  |-- AI Analysis API
  |-- AI Chat/RAG API
  |
  v
PostgreSQL + Prisma
  |
  v
S3 / SQS / Textract / Gemini / OpenAI
```

![frontend-dashboard-flow](/images/5-Workshop/5.9-Frontend-Dashboard/frontend-dashboard-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa React Frontend kết nối với Backend API, sau đó backend làm việc với PostgreSQL, Amazon S3, Amazon SQS, Amazon Textract, Gemini API và OpenAI ChatGPT API.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.9-Frontend-Dashboard/frontend-dashboard-flow.png`

---

#### Các màn hình chính

Frontend Dashboard của DocuMind AI gồm các màn hình chính:

| Màn hình           | Mục đích                                          |
| ------------------ | ------------------------------------------------- |
| Home Page          | Giới thiệu hệ thống DocuMind AI                   |
| Login/Register     | Xác thực người dùng                               |
| Document Dashboard | Quản lý tài liệu đã upload                        |
| Document Detail    | Xem OCR Result và AI Analysis                     |
| AI Chat/RAG        | Hỏi đáp với tài liệu                              |
| Enterprise Sandbox | Thử nghiệm semantic search, entities và analytics |
| Admin Panel        | Quản lý hệ thống nếu user là Admin                |

---

#### Nội dung chi tiết

Trong phần **5.9 - Frontend Dashboard**, chúng ta sẽ thực hiện các bước sau:

* [Trang chủ và xác thực người dùng](5.9.1-home-and-auth/)
* [Document Dashboard](5.9.2-document-dashboard/)
* [AI Chat và RAG](5.9.3-ai-chat-rag/)
* [Enterprise Sandbox](5.9.4-enterprise-sandbox/)

---

#### Công nghệ frontend sử dụng

Frontend của DocuMind AI có thể sử dụng:

```text
React
TypeScript
TailwindCSS
Framer Motion
Axios
React Router
Context API hoặc Zustand
```

Vai trò từng công nghệ:

| Công nghệ       | Vai trò                            |
| --------------- | ---------------------------------- |
| React           | Xây dựng giao diện người dùng      |
| TypeScript      | Tăng tính an toàn kiểu dữ liệu     |
| TailwindCSS     | Thiết kế UI nhanh và đồng bộ       |
| Framer Motion   | Tạo hiệu ứng chuyển động           |
| Axios           | Gọi Backend API                    |
| React Router    | Điều hướng giữa các trang          |
| Context/Zustand | Quản lý auth state và global state |

---

#### Phong cách giao diện đề xuất

Frontend nên giữ phong cách hiện đại, phù hợp với một nền tảng AI SaaS:

* Nền xanh nhẹ hoặc sky background.
* Glassmorphism card.
* Bo góc lớn.
* Shadow mềm.
* Animation nhẹ.
* Icon rõ ràng.
* Dashboard gọn, dễ đọc.
* Responsive trên desktop, tablet và mobile.
* Tránh quá nhiều màu gây rối giao diện.

Các màu chủ đạo đề xuất:

```text
Blue
Sky
White
Slate
Emerald for success
Red for error
Amber for processing
```

---

#### Luồng sử dụng của người dùng

Workflow chính của người dùng:

```text
User mở Home Page
  |
  v
Đăng ký / Đăng nhập
  |
  v
Vào Document Dashboard
  |
  v
Upload tài liệu
  |
  v
Theo dõi status xử lý
  |
  v
Xem OCR Result và AI Analysis
  |
  v
Chat với tài liệu
  |
  v
Thử Enterprise Sandbox nếu cần
```

---

#### Trạng thái xử lý hiển thị trên Dashboard

Frontend cần hiển thị các trạng thái xử lý tài liệu:

```text
PENDING
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Mapping progress đề xuất:

| Status          | Progress | Ý nghĩa               |
| --------------- | -------: | --------------------- |
| `PENDING`       |        5 | Tạo metadata ban đầu  |
| `UPLOADED`      |       15 | File đã upload lên S3 |
| `QUEUED`        |       25 | Job đã gửi vào SQS    |
| `PROCESSING`    |       45 | Worker đang xử lý OCR |
| `OCR_COMPLETED` |       60 | OCR hoàn tất          |
| `AI_ANALYZING`  |       75 | AI đang phân tích     |
| `COMPLETED`     |      100 | Hoàn thành            |
| `FAILED`        |        0 | Xử lý thất bại        |

---

#### API frontend cần gọi

Các API chính frontend sử dụng:

| API                                    | Mục đích                      |
| -------------------------------------- | ----------------------------- |
| `POST /api/auth/register`              | Đăng ký                       |
| `POST /api/auth/login`                 | Đăng nhập                     |
| `GET /api/auth/me`                     | Lấy user hiện tại             |
| `POST /api/auth/logout`                | Đăng xuất                     |
| `POST /api/documents/upload`           | Upload tài liệu               |
| `GET /api/documents`                   | Lấy danh sách tài liệu        |
| `GET /api/documents/:id`               | Lấy chi tiết tài liệu         |
| `GET /api/documents/:id/status`        | Lấy trạng thái xử lý          |
| `GET /api/documents/:id/ocr`           | Lấy kết quả OCR               |
| `GET /api/documents/:id/analysis`      | Lấy kết quả AI Analysis       |
| `POST /api/ai/chat`                    | Chat với tài liệu             |
| `POST /api/enterprise/semantic-search` | Semantic search trong Sandbox |

---

#### Auth State

Frontend cần quản lý trạng thái đăng nhập:

```text
user
token
isAuthenticated
role
loading
```

Khi user đăng nhập thành công:

* Lưu token.
* Gọi `/api/auth/me`.
* Hiển thị avatar/name.
* Ẩn Login/Register.
* Cho phép truy cập Dashboard.
* Nếu role là `ADMIN`, hiển thị Admin Panel.

---

#### Document Dashboard State

Dashboard nên quản lý các state:

```text
documents
selectedDocument
uploading
processingStatus
loading
error
```

Khi user upload tài liệu thành công:

* Thêm document mới vào danh sách.
* Status ban đầu là `QUEUED`.
* Bắt đầu polling status.
* Hiển thị notification upload thành công.

---

#### AI Chat/RAG State

AI Chat cần quản lý:

```text
messages
question
provider
loading
sources
confidence
```

Khi user gửi câu hỏi:

* Gửi `documentId`, `question`, `provider` lên backend.
* Backend lấy OCR context.
* AI Gateway gọi Gemini/OpenAI.
* Frontend hiển thị câu trả lời.
* Lưu lịch sử chat vào database.

---

#### Enterprise Sandbox

Enterprise Sandbox dùng để thử nghiệm các chức năng nâng cao:

* Provider selector OpenAI/Gemini.
* AI Chat/RAG nâng cao.
* Semantic Search Simulation.
* Auto-classification.
* Entity Extraction.
* Analytics Export.

Phần này không sử dụng Amazon Bedrock. UI cần dùng các tên trung lập như:

```text
AI Embeddings Engine
Multi-Model AI Intelligence Core
DocuMind Multi-AI Compliance Active
OpenAI/Gemini Embedding Models
```

Không hiển thị:

```text
AWS Bedrock
Amazon Titan
Bedrock Embeddings
```

---

#### Kiểm tra bảo mật frontend

Frontend cần đảm bảo:

* Không lưu API key Gemini/OpenAI ở frontend.
* Không expose AWS credentials.
* Chỉ gửi JWT token đến backend.
* Không cho user xem tài liệu không thuộc quyền.
* Route Dashboard cần yêu cầu đăng nhập.
* Route Admin cần kiểm tra role `ADMIN`.
* Không hiển thị toàn bộ lỗi kỹ thuật nhạy cảm cho user.

---

#### Loading, Empty và Error State

Frontend cần có các trạng thái UI rõ ràng:

| State        | Mô tả                     |
| ------------ | ------------------------- |
| Loading      | Đang tải dữ liệu          |
| Empty        | Chưa có tài liệu          |
| Uploading    | Đang upload file          |
| Processing   | Tài liệu đang xử lý       |
| Completed    | Tài liệu đã hoàn thành    |
| Failed       | Tài liệu xử lý lỗi        |
| Unauthorized | Người dùng chưa đăng nhập |
| Forbidden    | Không có quyền truy cập   |

Ví dụ empty state:

```text
No documents yet.
Upload your first document to start extracting and analyzing content with AI.
```

---

#### Test frontend end-to-end

Các bước test:

1. Mở Home Page.
2. Đăng ký hoặc đăng nhập.
3. Kiểm tra avatar/name hiển thị.
4. Vào Document Dashboard.
5. Upload file PDF hoặc ảnh.
6. Kiểm tra tài liệu xuất hiện trong danh sách.
7. Kiểm tra status chuyển từ `QUEUED` sang `PROCESSING`.
8. Chạy worker xử lý OCR và AI.
9. Kiểm tra status thành `COMPLETED`.
10. Mở Document Detail.
11. Kiểm tra OCR Result và AI Analysis.
12. Mở AI Chat/RAG.
13. Hỏi một câu dựa trên tài liệu.
14. Kiểm tra Enterprise Sandbox.
15. Logout và kiểm tra route protected.

---

#### Các lỗi thường gặp

| Lỗi                                 | Nguyên nhân                        | Cách xử lý                       |
| ----------------------------------- | ---------------------------------- | -------------------------------- |
| Login thành công nhưng UI không đổi | Auth state chưa cập nhật           | Kiểm tra AuthContext/Zustand     |
| Upload lỗi CORS                     | Backend hoặc S3 CORS sai           | Kiểm tra CORS config             |
| Dashboard không cập nhật status     | Chưa polling hoặc worker chưa chạy | Kiểm tra Document Status API     |
| AI Analysis không hiện              | Tài liệu chưa `COMPLETED`          | Kiểm tra worker và AI Gateway    |
| 401 Unauthorized                    | Token hết hạn                      | Refresh token hoặc đăng nhập lại |
| 403 Forbidden                       | User không sở hữu tài liệu         | Kiểm tra quyền backend           |
| UI lỗi mobile                       | Responsive chưa hoàn chỉnh         | Kiểm tra Tailwind breakpoint     |
| Vẫn còn chữ Bedrock                 | UI còn text cũ                     | Search và thay Bedrock/Titan     |

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được:

* Home Page hiển thị đúng.
* Login/Register hoạt động.
* Auth state được lưu và khôi phục khi reload.
* Document Dashboard hiển thị danh sách tài liệu.
* Upload tài liệu từ frontend hoạt động.
* Status xử lý được cập nhật đúng.
* Document Detail hiển thị OCR và AI Analysis.
* AI Chat/RAG hoạt động.
* Enterprise Sandbox hoạt động với OpenAI/Gemini.
* Không còn nội dung Bedrock/Titan trong UI.
* Frontend responsive và có trải nghiệm ổn định.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI đã có giao diện frontend hoàn chỉnh để người dùng sử dụng toàn bộ pipeline xử lý tài liệu. Người dùng có thể đăng nhập, upload tài liệu, theo dõi trạng thái xử lý, xem kết quả OCR/AI, chat với tài liệu và thử nghiệm các chức năng nâng cao trong Enterprise Sandbox.
