---
title: "Worklog Tuần 10"
date: 2026-06-22
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---


### Mục tiêu tuần 10:

* Phân tích yêu cầu của dự án Cloud-Native AI Document Extractor & Analytics.
* Tìm hiểu kiến trúc Serverless trên nền tảng AWS.
* Thiết kế sơ đồ kiến trúc tổng thể và luồng xử lý dữ liệu của hệ thống.
* Chuẩn bị các tài nguyên AWS phục vụ cho quá trình phát triển dự án.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------ | --------------- | ----------------------------------------- |
| 2 | - Tiếp nhận yêu cầu dự án **Cloud-Native AI Document Extractor & Analytics** <br> - Phân tích yêu cầu nghiệp vụ <br> - Xác định đầu vào, đầu ra của hệ thống <br> - Xác định các chức năng chính cần triển khai | 22/06/2026 | 22/06/2026 | Tài liệu dự án, AWS Documentation |
| 3 | - Nghiên cứu mô hình **Serverless Architecture** <br>&emsp; + Amazon S3 <br>&emsp; + AWS Lambda <br>&emsp; + Amazon Textract <br>&emsp; + Amazon Bedrock <br>&emsp; + Amazon DynamoDB / Amazon RDS <br> - Phân tích vai trò của từng dịch vụ trong hệ thống | 23/06/2026 | 24/06/2026 | https://docs.aws.amazon.com/ |
| 4 | - Thiết kế sơ đồ kiến trúc hệ thống <br>&emsp; + Luồng upload tài liệu <br>&emsp; + Luồng xử lý bằng Lambda <br>&emsp; + Luồng trích xuất dữ liệu bằng Textract <br>&emsp; + Luồng phân tích bằng Bedrock <br>&emsp; + Luồng lưu kết quả vào cơ sở dữ liệu | 25/06/2026 | 26/06/2026 | AWS Architecture Center |
| 5 | - Chuẩn bị môi trường phát triển <br>&emsp; + Tạo S3 Bucket <br>&emsp; + Chuẩn bị IAM Role cho Lambda <br>&emsp; + Khởi tạo Lambda Function <br>&emsp; + Kiểm tra quyền truy cập giữa các dịch vụ AWS | 27/06/2026 | 27/06/2026 | AWS Documentation |
| 6 | - Kiểm tra toàn bộ kiến trúc hệ thống <br> - Hoàn thiện sơ đồ kiến trúc bằng Draw.io <br> - Tổng hợp tài liệu thiết kế phục vụ cho giai đoạn phát triển hệ thống | 28/06/2026 | 28/06/2026 | https://aws.amazon.com/architecture/ |

### Kết quả đạt được tuần 10:

* Phân tích đầy đủ yêu cầu chức năng và yêu cầu phi chức năng của dự án Cloud-Native AI Document Extractor & Analytics.

* Xác định được luồng xử lý dữ liệu từ khi người dùng tải tài liệu lên cho đến khi hệ thống trả về kết quả phân tích.

* Hiểu được cách xây dựng một hệ thống theo mô hình Serverless trên nền tảng AWS.

* Thiết kế hoàn chỉnh sơ đồ kiến trúc hệ thống với các thành phần:
  * Amazon S3
  * AWS Lambda
  * Amazon Textract
  * Amazon Bedrock
  * Amazon DynamoDB/Amazon RDS

* Chuẩn bị thành công các tài nguyên AWS cần thiết để triển khai dự án.

* Cấu hình IAM Role cho phép AWS Lambda truy cập các dịch vụ S3, Textract và cơ sở dữ liệu.

* Hoàn thiện sơ đồ kiến trúc hệ thống bằng Draw.io, thể hiện rõ luồng xử lý dữ liệu giữa các dịch vụ AWS.

* Hoàn thành tài liệu phân tích và thiết kế hệ thống, tạo nền tảng cho quá trình phát triển và tích hợp các chức năng ở tuần tiếp theo.

* Nâng cao kỹ năng phân tích yêu cầu, thiết kế kiến trúc Cloud và xây dựng hệ thống theo mô hình Serverless.