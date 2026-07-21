---
title: "Sự kiện 2: Workshop Hiện đại hóa Ứng dụng & Cơ sở dữ liệu với GenAI"
date: 2026-05-23
weight: 2
chapter: false
pre: " <b> 4.2. </b> "
---

### Mục tiêu và Tổng quan sự kiện

Workshop **"GenAI-powered App-DB Modernization"** tập trung vào các phương pháp kiến trúc thực tiễn và các dịch vụ điện toán đám mây thế hệ mới của AWS. Sự kiện giúp các kỹ sư xây dựng tư duy hiện đại hóa ứng dụng một cách toàn diện nhằm nâng cao tính linh hoạt của doanh nghiệp, tối ưu chi phí, tăng cường bảo mật tại Edge và cải thiện hiệu năng hệ thống.

Các mục tiêu chính bao gồm:

- Chia sẻ các phương pháp hay nhất trong thiết kế và vận hành phần mềm hiện đại.
- Giới thiệu phương pháp **Domain-Driven Design (DDD)** và kiến trúc hướng sự kiện bất đồng bộ (Event-Driven Architecture).
- Hướng dẫn chuyên sâu về cách lựa chọn và phân bổ các dịch vụ tính toán (Compute Services) phù hợp trên AWS.
- Trình bày các công cụ **Generative AI (GenAI)** hỗ trợ tự động hóa toàn bộ vòng đời phát triển phần mềm (SDLC).

### Danh sách diễn giả

- **Jignesh Shah** – Giám đốc Open Source Databases
- **Erica Liu** – Chuyên gia GTM cấp cao, AppMod
- **Fabrianne Effendi** – Associate Specialist SA, Serverless, Amazon Web Services
- **Đào Đức** – Solutions Architect, Cloud Kinetics
- **Nguyễn Tuấn Thịnh** – DevOps Engineer, Dự án First Cloud AI Journey
- **Phạm Anh** – AWS Community Builder (Security Lead)
- **Phạm Ngũ Hải Anh** – AI/BI Solutions Specialist

### Nội dung kỹ thuật nổi bật

#### 3.1 Hạn chế của kiến trúc Monolithic truyền thống

Kiến trúc nguyên khối (Monolithic) tồn tại nhiều hạn chế ảnh hưởng đến khả năng mở rộng và phát triển hệ thống:

- **Chu kỳ phát hành kéo dài:** Mã nguồn lớn và phụ thuộc chặt chẽ khiến việc triển khai tính năng mới trở nên chậm chạp.
- **Hiệu quả vận hành thấp:** Các thành phần phụ thuộc lẫn nhau làm tăng chi phí hạ tầng và giảm năng suất phát triển.
- **Rủi ro bảo mật:** Công nghệ cũ khó đáp ứng các tiêu chuẩn bảo mật hiện đại, làm tăng nguy cơ bị tấn công và rò rỉ dữ liệu.

#### 3.2 Chuyển đổi sang kiến trúc Microservices

Việc hiện đại hóa ứng dụng yêu cầu chuyển đổi sang mô hình Microservices, trong đó mỗi chức năng nghiệp vụ hoạt động độc lập và giao tiếp thông qua các sự kiện.

Ba thành phần quan trọng gồm:

- **Queue Management:** Quản lý hàng đợi và xử lý bất đồng bộ.
- **Caching Strategy:** Lưu trữ dữ liệu tạm thời nhằm giảm tải cho cơ sở dữ liệu.
- **Message Handling:** Thiết lập cơ chế trao đổi dữ liệu linh hoạt giữa các dịch vụ.

#### 3.3 Domain-Driven Design (DDD)

DDD cung cấp phương pháp phân chia hệ thống lớn thành các miền nghiệp vụ độc lập thông qua 4 bước:

<div style="display:flex;flex-wrap:wrap;justify-content:center;align-items:center;gap:10px;margin:20px 0;font-weight:bold;font-size:0.95em;">
<div style="background:#f0f4f8;padding:8px 12px;border-radius:6px;">1. Xác định Domain Events</div>
<div>→</div>
<div style="background:#f0f4f8;padding:8px 12px;border-radius:6px;">2. Sắp xếp theo trình tự thời gian</div>
<div>→</div>
<div style="background:#f0f4f8;padding:8px 12px;border-radius:6px;">3. Xác định các Actor</div>
<div>→</div>
<div style="background:#f0f4f8;padding:8px 12px;border-radius:6px;">4. Xác định Bounded Context</div>
</div>

Thông qua ví dụ **Bookstore**, các diễn giả trình bày kỹ thuật **Context Mapping** cùng 7 kiểu quan hệ giữa các Bounded Context.

#### 3.4 Event-Driven Architecture

Kiến trúc hướng sự kiện đóng vai trò kết nối các Microservices thông qua ba mô hình:

- **Publish/Subscribe (Pub/Sub):** Phát tán sự kiện đến nhiều dịch vụ.
- **Point-to-Point:** Gửi thông điệp đến một hàng đợi cụ thể.
- **Event Streaming:** Truyền liên tục các thay đổi trạng thái theo thời gian thực.

> **Nhận xét:** So với REST API đồng bộ, giao tiếp bất đồng bộ giúp giảm sự phụ thuộc giữa các dịch vụ, tăng khả năng mở rộng và nâng cao độ ổn định của hệ thống.

#### 3.5 Sự phát triển của hạ tầng tính toán

Mô hình Shared Responsibility của AWS cho thấy sự chuyển dịch từ việc tự quản lý máy chủ sang Serverless:

<div style="display:flex;justify-content:center;gap:10px;flex-wrap:wrap;margin:20px 0;font-weight:bold;">
<div>Amazon EC2</div>
<div>→</div>
<div>Amazon ECS</div>
<div>→</div>
<div>AWS Fargate</div>
<div>→</div>
<div>AWS Lambda</div>
</div>

Kiến trúc Serverless mang lại nhiều lợi ích:

- Không cần quản trị máy chủ.
- Tự động mở rộng theo lưu lượng truy cập.
- Chỉ thanh toán theo mức sử dụng thực tế.

Việc lựa chọn giữa **AWS Lambda** và **AWS Fargate** phụ thuộc vào thời gian thực thi, độ trễ khởi động và đặc điểm của ứng dụng.

#### 3.6 Amazon Q Developer

Amazon Q Developer hỗ trợ tự động hóa nhiều công đoạn trong SDLC:

- Chuyển đổi mã nguồn từ các phiên bản Java hoặc .NET cũ.
- Hỗ trợ di chuyển hệ thống VMware, Mainframe và các ứng dụng truyền thống lên AWS.

#### 3.7 Tính không xác định của LLM

Diễn giả Đào Đức giải thích rằng ngay cả khi đặt **Temperature = 0**, mô hình ngôn ngữ lớn vẫn có thể tạo ra các kết quả khác nhau.

Nguyên nhân chủ yếu đến từ:

- Sai số của phép tính dấu chấm động trên GPU.
- Dynamic Batching.
- Kiến trúc xử lý song song.

Qua thử nghiệm trên GPT-3.5 Turbo, GPT-4o, Llama-3 và Mixtral, độ chính xác vẫn có thể dao động khoảng **15%**.

#### 3.8 CloudFront và Edge Computing

CloudFront giúp nâng cao hiệu năng và bảo mật thông qua:

- Xử lý logic tại Edge bằng CloudFront Functions hoặc Lambda@Edge.
- Bộ nhớ đệm nhiều tầng với Regional Edge Cache và Origin Shield.
- Bảo vệ Origin bằng OAC và VPC Origin.
- Hỗ trợ Mutual TLS.
- Giảm tải CPU nhờ offload TLS và nén Brotli/Gzip.
- Giảm thiểu tấn công DDoS bằng mạng Anycast toàn cầu.

#### 3.9 Tối ưu chi phí

Diễn giả giới thiệu các gói cố định thay cho mô hình Pay-as-you-go:

- **Free:** 100 GB dữ liệu, 1 triệu request/tháng.
- **Pro:** 50 GB Premium Data Transfer, 10 triệu request.
- **Business:** 500 GB dữ liệu, 125 triệu request.
- **Premium:** 1.75 TB dữ liệu, 500 triệu request.

#### 3.10 Prompt → Context → Memory

Các diễn giả giới thiệu xu hướng phát triển AI:

<div style="display:flex;justify-content:center;gap:10px;flex-wrap:wrap;margin:20px 0;font-weight:bold;">
<div>Prompt</div>
<div>→</div>
<div>Context</div>
<div>→</div>
<div>Memory</div>
</div>

Các nội dung nổi bật:

- Xây dựng **Second Brain Architecture**.
- Tích hợp **Amazon Q Business** và Amazon Bedrock.
- Kết nối hơn 40 nguồn dữ liệu doanh nghiệp.
- Tự động ghi chú cuộc họp và tạo báo cáo.

#### 3.11 Lộ trình triển khai AI

Quy trình triển khai GenAI gồm 4 giai đoạn:

1. Foundation
2. Integration
3. Pilot
4. Scale

### Bài học rút ra

#### Về tư duy kiến trúc

- Thiết kế hệ thống cần xuất phát từ nghiệp vụ.
- Ưu tiên Event-Driven Architecture thay cho giao tiếp đồng bộ.
- Tận dụng Amazon Q Developer để giảm Technical Debt.

### Ứng dụng trong học tập

Những kiến thức tiếp thu từ workshop có giá trị thực tiễn cao và có thể áp dụng trực tiếp vào quá trình học tập cũng như thực hiện các đồ án về điện toán đám mây.

- Vận dụng tư duy **Domain-Driven Design (DDD)** để phân tích yêu cầu nghiệp vụ và xây dựng kiến trúc hệ thống rõ ràng hơn trong các đồ án phát triển phần mềm.
- Áp dụng mô hình **Microservices** kết hợp **Event-Driven Architecture** để thiết kế các ứng dụng có khả năng mở rộng, giảm sự phụ thuộc giữa các thành phần và nâng cao tính sẵn sàng của hệ thống.
- Lựa chọn và sử dụng phù hợp các dịch vụ tính toán của AWS như **Amazon EC2**, **AWS Lambda** và **AWS Fargate** dựa trên đặc điểm của từng bài toán nhằm tối ưu hiệu năng và chi phí.
- Tận dụng **Amazon Q Developer** để hỗ trợ phân tích mã nguồn, sinh tài liệu kỹ thuật, giải thích code và tăng năng suất trong quá trình học tập cũng như phát triển dự án.
- Áp dụng các kiến thức về **Amazon CloudFront**, bộ nhớ đệm (Caching), bảo mật tại Edge và tối ưu phân phối nội dung để cải thiện hiệu năng cho các ứng dụng web triển khai trên AWS.
- Hiểu rõ hơn về đặc điểm của các mô hình ngôn ngữ lớn (LLMs), từ đó xây dựng prompt hiệu quả và đánh giá kết quả sinh ra một cách khách quan khi phát triển các ứng dụng AI.
- Vận dụng kiến thức về **Amazon Bedrock** và **Amazon Q Business** để nghiên cứu các giải pháp tích hợp Generative AI vào hệ thống quản lý tài liệu, chatbot hỗ trợ người dùng và các ứng dụng xử lý dữ liệu doanh nghiệp.

Những kiến thức và kinh nghiệm thu được từ workshop sẽ là nền tảng quan trọng để tôi tiếp tục nghiên cứu các công nghệ Cloud Computing, Serverless và Generative AI, đồng thời hỗ trợ hiệu quả cho việc thực hiện đồ án tốt nghiệp cũng như định hướng phát triển nghề nghiệp trong lĩnh vực Cloud Solutions Architecture và AI trên nền tảng AWS.

### Cảm nhận về sự kiện

Workshop **"GenAI-powered App-DB Modernization"** mang lại nhiều kiến thức chuyên sâu về hiện đại hóa ứng dụng và cơ sở dữ liệu trên nền tảng AWS.

Những điểm nổi bật gồm:

- Học hỏi kinh nghiệm thực tế từ các chuyên gia AWS.
- Quan sát các minh họa trực tiếp về AI, CloudFront và DDoS Protection.
- Mở rộng mạng lưới kết nối với cộng đồng kỹ sư Cloud.


**Tổng kết:** Workshop đã giúp tôi hiểu rõ hơn về kiến trúc hiện đại trên AWS, từ Microservices, Serverless đến Event-Driven Architecture và GenAI. Đây là những kiến thức rất hữu ích cho định hướng trở thành Cloud Solutions Architect trong tương lai.

