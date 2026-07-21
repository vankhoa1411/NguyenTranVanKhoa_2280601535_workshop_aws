
---
title : "Document Dashboard"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.9.2. </b> "
---

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng **Document Dashboard** cho hệ thống **DocuMind AI**.

Document Dashboard là nơi người dùng quản lý các tài liệu đã upload, theo dõi trạng thái xử lý, xem kết quả OCR, xem kết quả AI Analysis và truy cập vào chức năng AI Chat/RAG. Đây là giao diện chính giúp người dùng quan sát toàn bộ pipeline xử lý tài liệu từ lúc upload đến khi hoàn thành.

Dashboard sẽ kết nối với Backend API để lấy danh sách tài liệu, trạng thái xử lý, kết quả OCR và kết quả phân tích AI được lưu trong PostgreSQL.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

- Xây dựng giao diện danh sách tài liệu.
- Hiển thị trạng thái xử lý tài liệu.
- Hiển thị tiến trình xử lý theo từng status.
- Gọi API lấy danh sách tài liệu.
- Gọi API lấy chi tiết tài liệu.
- Hiển thị OCR result và AI Analysis.
- Tạo nút upload tài liệu mới.
- Cho phép user xem tài liệu của chính họ.
- Cho phép Admin xem tổng quan nhiều tài liệu nếu có quyền.

---

#### Vai trò của Document Dashboard

Trong DocuMind AI, Document Dashboard giúp người dùng:

- Xem danh sách tài liệu đã upload.
- Biết tài liệu đang ở bước nào trong pipeline.
- Xem file name, file type, file size và thời gian upload.
- Theo dõi trạng thái `QUEUED`, `PROCESSING`, `OCR_COMPLETED`, `AI_ANALYZING`, `COMPLETED`, `FAILED`.
- Mở chi tiết tài liệu để xem OCR và AI Analysis.
- Truy cập nhanh vào AI Chat/RAG.
- Nhận thông báo khi tài liệu xử lý xong.

---

#### Kiến trúc Document Dashboard

Luồng dữ liệu:

```text
React Dashboard
  |
  v
GET /api/documents
  |
  v
Backend API
  |
  v
PostgreSQL + Prisma
  |
  v
Return documents + status + analysis summary