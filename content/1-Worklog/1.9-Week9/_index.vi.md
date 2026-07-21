---
title: "Worklog Tuần 9"
date: 2026-06-15
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---



### Mục tiêu tuần 9:

* Tìm hiểu kiến thức cơ bản về AWS Networking và Amazon VPC.
* Hiểu cách xây dựng một hệ thống mạng trên nền tảng AWS.
* Thực hành tạo VPC, Subnet và Internet Gateway.
* Nắm được cách các dịch vụ AWS giao tiếp với nhau trong cùng một hệ thống.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2 | - Tìm hiểu tổng quan về **Amazon Virtual Private Cloud (VPC)** <br> - Khái niệm CIDR Block, IP Address <br> - Vai trò của VPC trong việc xây dựng hạ tầng mạng trên AWS | 15/06/2026 | 15/06/2026 | <https://docs.aws.amazon.com/vpc/> |
| 3 | - Thực hành tạo **Amazon VPC** <br>&emsp; + Tạo VPC mới <br>&emsp; + Cấu hình CIDR Block <br>&emsp; + Tạo Public Subnet và Private Subnet <br>&emsp; + Kiểm tra địa chỉ IP được cấp phát | 16/06/2026 | 17/06/2026 | <https://docs.aws.amazon.com/vpc/latest/userguide/> |
| 4 | - Tìm hiểu **Internet Gateway** và **Route Table** <br>&emsp; + Gắn Internet Gateway vào VPC <br>&emsp; + Cấu hình Route Table cho Public Subnet <br>&emsp; + Kiểm tra khả năng truy cập Internet | 18/06/2026 | 19/06/2026 | <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html> |
| 5 | - Tìm hiểu cơ chế bảo mật mạng trên AWS <br>&emsp; + Security Group <br>&emsp; + Network ACL (NACL) <br>&emsp; + So sánh Security Group và NACL <br>&emsp; + Mô hình kết nối giữa EC2 và các dịch vụ AWS | 20/06/2026 | 20/06/2026 | <https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html> |
| 6 | - Tổng hợp kiến thức về AWS Networking <br> - Vẽ sơ đồ mô hình mạng VPC đã thực hành <br> - Chuẩn bị kiến thức phục vụ thiết kế kiến trúc hệ thống Cloud-Native ở tuần tiếp theo | 21/06/2026 | 21/06/2026 | <https://explore.skillbuilder.aws/> |

### Kết quả đạt được tuần 9:

* Hiểu được vai trò của Amazon VPC trong việc xây dựng hạ tầng mạng riêng trên AWS.

* Nắm được khái niệm CIDR Block và cách phân chia địa chỉ IP trong hệ thống.

* Tạo thành công một VPC mới phục vụ cho việc triển khai các tài nguyên AWS.

* Hiểu sự khác nhau giữa Public Subnet và Private Subnet cũng như các trường hợp sử dụng của từng loại.

* Cấu hình thành công Internet Gateway và Route Table để cho phép tài nguyên trong Public Subnet kết nối Internet.

* Hiểu được vai trò của Security Group và Network ACL trong việc bảo vệ tài nguyên trên AWS.

* Biết cách xây dựng mô hình mạng cơ bản cho một ứng dụng triển khai trên nền tảng Cloud.

* Hình dung được luồng kết nối giữa các dịch vụ như EC2, S3, RDS và Internet trong cùng một hệ thống.

* Hoàn thành sơ đồ mạng VPC và ghi chép đầy đủ quy trình cấu hình để phục vụ cho việc thiết kế kiến trúc dự án.

* Có đủ kiến thức nền tảng về Networking để triển khai dự án **Cloud-Native AI Document Extractor & Analytics** theo mô hình Serverless ở các tuần tiếp theo.