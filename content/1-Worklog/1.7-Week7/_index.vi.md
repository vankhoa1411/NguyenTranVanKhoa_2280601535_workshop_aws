---
title: "Worklog Tuần 7"
date: 2026-06-01
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---


### Mục tiêu tuần 7:

* Tìm hiểu chi tiết về dịch vụ lưu trữ Amazon S3.
* Hiểu mô hình Object Storage và cách tổ chức dữ liệu trên AWS.
* Thực hành quản lý Bucket, Upload dữ liệu và phân quyền truy cập.
* Triển khai Website tĩnh (Static Website) bằng Amazon S3.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2 | - Tìm hiểu tổng quan về **Amazon S3** <br> - Khái niệm Object Storage <br> - Cấu trúc Bucket, Object và Key <br> - Các lớp lưu trữ (Storage Classes) của Amazon S3 | 01/06/2026 | 01/06/2026 | <https://docs.aws.amazon.com/AmazonS3/latest/userguide/> |
| 3 | - Thực hành tạo S3 Bucket <br>&emsp; + Đặt tên Bucket <br>&emsp; + Cấu hình Region <br>&emsp; + Upload hình ảnh, tài liệu và các tệp mẫu <br>&emsp; + Quản lý Object trong Bucket | 02/06/2026 | 03/06/2026 | <https://docs.aws.amazon.com/AmazonS3/latest/userguide/> |
| 4 | - Tìm hiểu Bucket Policy và ACL <br>&emsp; + Public Access Block <br>&emsp; + Bucket Policy <br>&emsp; + Phân quyền truy cập tệp <br>&emsp; + Kiểm tra quyền truy cập từ trình duyệt | 04/06/2026 | 05/06/2026 | <https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-policy-language-overview.html> |
| 5 | - Thực hành **Static Website Hosting** <br>&emsp; + Cấu hình Website Hosting <br>&emsp; + Upload file index.html và assets <br>&emsp; + Kiểm tra Website qua Endpoint của S3 <br>&emsp; + Khắc phục lỗi truy cập thường gặp | 06/06/2026 | 06/06/2026 | <https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html> |
| 6 | - Tổng hợp kiến thức về Amazon S3 <br> - Ghi chép quy trình tạo Bucket và triển khai Website tĩnh <br> - Chuẩn bị kiến thức cho việc lưu trữ tài liệu của dự án ở các tuần tiếp theo | 07/06/2026 | 07/06/2026 | <https://explore.skillbuilder.aws/> |

### Kết quả đạt được tuần 7:

* Hiểu được nguyên lý hoạt động của dịch vụ lưu trữ Amazon S3.

* Nắm được mô hình Object Storage và cách tổ chức dữ liệu thông qua Bucket và Object.

* Tạo thành công nhiều S3 Bucket phục vụ cho việc lưu trữ dữ liệu.

* Thực hiện thành công việc Upload, Download và quản lý các tệp trên Amazon S3.

* Hiểu được sự khác nhau giữa Bucket Policy, ACL và Block Public Access trong việc phân quyền truy cập.

* Cấu hình thành công quyền truy cập công khai cho Website tĩnh theo đúng yêu cầu.

* Triển khai thành công một Website tĩnh bằng tính năng **Static Website Hosting** của Amazon S3.

* Biết cách kiểm tra Endpoint của Website và xử lý các lỗi phổ biến như **403 Forbidden** hoặc **Access Denied**.

* Hiểu được vai trò của Amazon S3 trong việc lưu trữ hình ảnh, tài liệu và dữ liệu đầu vào cho các hệ thống Serverless.

* Hoàn thiện tài liệu hướng dẫn sử dụng Amazon S3, làm nền tảng cho việc xây dựng dự án **Cloud-Native AI Document Extractor & Analytics** ở các tuần tiếp theo.