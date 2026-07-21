---
title: "Tự động hóa giao diện AI Agent: Chuẩn hóa Generative UI bằng giao thức AG-UI trên Amazon Bedrock AgentCore"
date: 2026-07-10
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---
{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

# Tự động hóa giao diện AI Agent: Chuẩn hóa Generative UI bằng giao thức AG-UI trên Amazon Bedrock AgentCore

Các AI Agent ngày nay không còn chỉ dừng lại ở việc trả lời bằng văn bản. Để Agent có thể hiển thị biểu đồ tương tác, cập nhật bảng công việc theo thời gian thực hoặc tạm dừng để chờ người dùng phê duyệt, cần có một chuẩn giao tiếp thống nhất giữa backend và frontend.

AWS vừa giới thiệu cách tích hợp giao thức mã nguồn mở AG-UI (Agent-User Interaction Protocol) với Amazon Bedrock AgentCore Runtime, giúp xây dựng các ứng dụng AI có giao diện tương tác phong phú một cách linh hoạt, bảo mật và dễ mở rộng.

---

### 1. AG-UI – Chuẩn hóa giao tiếp giữa Backend và Frontend

AG-UI sử dụng luồng sự kiện Server-Sent Events (SSE) để chuẩn hóa cách AI Agent giao tiếp với giao diện người dùng.

Nhờ đó, lập trình viên có thể tự do kết hợp nhiều framework AI ở backend như Strands Agents, LangGraph, CrewAI với các framework frontend như React, Angular hoặc Vue mà không cần xây dựng cơ chế giao tiếp riêng cho từng framework.

Frontend chỉ cần triển khai một AG-UI client để nhận và xử lý các sự kiện theo chuẩn AG-UI, bất kể Agent phía sau được xây dựng bằng công nghệ nào.

---

### 2. CopilotKit + FAST – Mang Generative UI vào AI Agent

Khi kết hợp với mẫu ứng dụng FAST và thư viện CopilotKit, AI Agent có thể mang lại những trải nghiệm vượt xa một cửa sổ chat truyền thống.

* **Generative UI**: Thay vì chỉ trả về văn bản, Agent gửi các sự kiện AG-UI mô tả giao diện cần hiển thị. Frontend sẽ tự động render các React component như biểu đồ, bảng dữ liệu hoặc dashboard dựa trên dữ liệu thực tế.
* **Shared State (Đồng bộ trạng thái)**: Trạng thái được đồng bộ hai chiều giữa Agent và giao diện. Khi người dùng chỉnh sửa danh sách công việc trên màn hình, Agent sẽ nhận biết ngay lập tức và ngược lại.
* **Human-in-the-loop**: Agent có thể tạm dừng quá trình xử lý để yêu cầu người dùng xác nhận thông tin, chẳng hạn chọn thời gian họp bằng Time Picker, trước khi tiếp tục thực hiện các bước tiếp theo.

![Kiến trúc AG-UI trên Bedrock AgentCore](/static/images/BlogsPosted/Bài 3/735076173_2235605227214217_4643365699851048466_n.jpg)

---

### 3. Vận hành trên Amazon Bedrock AgentCore Runtime

Khi kích hoạt hỗ trợ AG-UI, Amazon Bedrock AgentCore Runtime đảm nhiệm việc truyền luồng sự kiện giữa Agent và frontend, đồng thời xử lý nhiều tác vụ hạ tầng quan trọng như:
* Xác thực người dùng thông qua Amazon Cognito hoặc AWS Signature Version 4 (SigV4).
* Quản lý và cô lập phiên làm việc (Session Isolation).
* Streaming dữ liệu theo thời gian thực.
* Tự động mở rộng quy mô khi số lượng Agent hoặc người dùng tăng lên.

Ngoài ra, hệ thống có thể tích hợp AgentCore Memory để lưu trữ ngữ cảnh và trạng thái lâu dài của Agent, giúp duy trì trải nghiệm liên tục giữa nhiều phiên làm việc.

---

### Kết quả

Việc chuẩn hóa giao tiếp bằng AG-UI mang lại nhiều lợi ích:
* **Trải nghiệm tốt hơn**: Biến AI từ một chatbot hỏi–đáp thành không gian làm việc trực quan với biểu đồ, bảng dữ liệu và các thành phần tương tác.
* **Linh hoạt**: Có thể thay đổi framework backend (ví dụ từ Strands Agents sang LangGraph) mà không cần sửa mã nguồn frontend.
* **Sẵn sàng cho doanh nghiệp**: Dễ triển khai bằng AWS CDK, tích hợp sẵn cơ chế bảo mật, quản lý phiên làm việc và khả năng mở rộng trên Amazon Bedrock AgentCore.

---

### Tổng kết

Đây là một pattern kiến trúc đáng chú ý cho các ứng dụng GenAI thế hệ mới: chuẩn hóa lớp giao tiếp (Protocol-driven) để tách biệt hoàn toàn logic AI với tầng hiển thị giao diện.

Nhờ đó, backend và frontend có thể phát triển độc lập, dễ dàng thay thế framework AI mà vẫn duy trì trải nghiệm người dùng nhất quán. Đây cũng là nền tảng quan trọng để xây dựng các AI Agent hiện đại với khả năng Generative UI, Shared State và Human-in-the-loop trên quy mô doanh nghiệp.

*Chi tiết xem tại bài viết gốc:* [Build Generative UI for AI agents on Amazon Bedrock AgentCore with the AG-UI protocol](https://aws.amazon.com/blogs/machine-learning/build-generative-ui-for-ai-agents-on-amazon-bedrock-agentcore-with-the-ag-ui-protocol/)
