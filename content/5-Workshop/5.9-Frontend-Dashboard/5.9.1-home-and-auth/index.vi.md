---
title : "Trang chủ và xác thực người dùng"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.9.1. </b> "
---

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng và kiểm tra **Home Page** cùng chức năng **Authentication** cho hệ thống **DocuMind AI**.

Trang chủ là nơi giới thiệu tổng quan về nền tảng DocuMind AI, mô tả các chức năng chính như upload tài liệu, OCR bằng Amazon Textract, phân tích bằng Gemini/OpenAI, AI Chat/RAG, Dashboard và Admin Panel. Đây cũng là điểm bắt đầu để người dùng đăng nhập, đăng ký tài khoản và truy cập vào hệ thống xử lý tài liệu.

Chức năng xác thực giúp hệ thống phân biệt người dùng thông thường và Admin, đảm bảo chỉ người dùng hợp lệ mới được upload tài liệu, xem kết quả phân tích và sử dụng các chức năng bảo mật.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

- Xây dựng giao diện Home Page cho DocuMind AI.
- Hiển thị đúng thông tin giới thiệu hệ thống.
- Tạo giao diện Login/Register.
- Kết nối frontend với backend authentication API.
- Lưu trạng thái đăng nhập của người dùng.
- Hiển thị avatar, tên người dùng và dropdown sau khi đăng nhập.
- Phân quyền giao diện giữa User và Admin.
- Kiểm tra logout hoạt động đúng.

---

#### Vai trò của Home Page trong DocuMind AI

Home Page có nhiệm vụ:

- Giới thiệu mục tiêu của DocuMind AI.
- Trình bày các tính năng chính của hệ thống.
- Cho người dùng biết workflow xử lý tài liệu.
- Cung cấp nút đăng nhập, đăng ký hoặc bắt đầu upload tài liệu.
- Điều hướng người dùng đến Dashboard, AI Models, Enterprise Sandbox hoặc Pricing.
- Tạo trải nghiệm ban đầu chuyên nghiệp cho hệ thống.

Các nội dung nên có trên Home Page:

- Hero section giới thiệu DocuMind AI.
- Mô tả ngắn về OCR và AI document analytics.
- Các tính năng chính: Upload, OCR, AI Analysis, AI Chat, Notification, Admin.
- CTA button như “Get Started”, “Upload Document” hoặc “Explore Dashboard”.
- Navbar có các mục điều hướng.
- Footer thông tin dự án.

---

#### Kiến trúc Home và Authentication

Luồng hoạt động:

```text
User
  |
  v
React Home Page
  |
  v
Login / Register Modal
  |
  v
Backend Auth API
  |
  v
JWT Token
  |
  v
Frontend Auth State
  |
  v
Dashboard / Upload / Admin