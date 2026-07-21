---
title: "Triển khai MuleSoft Runtime Fabric trên ROSA"
date: 2026-07-10
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---
{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

# Triển khai MuleSoft Runtime Fabric trên ROSA: Hiện đại hóa tích hợp Hybrid Cloud với AWS

Nhiều doanh nghiệp đang vận hành đồng thời hệ thống on-premises và cloud, khiến việc tích hợp dữ liệu và quản lý API ngày càng phức tạp. Thách thức đặt ra là làm sao triển khai các dịch vụ tích hợp một cách nhất quán, dễ mở rộng và giảm chi phí vận hành.

AWS, Red Hat và MuleSoft giới thiệu mô hình triển khai MuleSoft Runtime Fabric (RTF) trên Red Hat OpenShift Service on AWS (ROSA), giúp doanh nghiệp xây dựng nền tảng tích hợp Hybrid Cloud hiện đại dựa trên Kubernetes được quản lý hoàn toàn.

---

### 1. MuleSoft Runtime Fabric trên ROSA

MuleSoft Runtime Fabric (RTF) là nền tảng triển khai và điều phối các ứng dụng Mule trên Kubernetes, cho phép chạy các API và dịch vụ tích hợp trên nhiều môi trường như on-premises, private cloud và public cloud.

Khi triển khai trên ROSA (Red Hat OpenShift Service on AWS), Runtime Fabric được cài đặt dưới dạng Red Hat OpenShift Certified Operator, giúp tự động hóa việc triển khai, quản lý và mở rộng ứng dụng theo chuẩn Kubernetes.

![MuleSoft Runtime Fabric trên ROSA](/images/BlogsPosted/Bài 1/739437321_2240342243407182_7284734041828767801_n.jpg)

---

### 2. Lợi ích của ROSA Hosted Control Planes (HCP)

Bài viết tập trung vào kiến trúc ROSA Hosted Control Planes (HCP), trong đó Control Plane được Red Hat quản lý, còn doanh nghiệp chỉ cần vận hành các Worker Nodes trong tài khoản AWS của mình.

Mô hình này mang lại nhiều lợi ích:
* **Giảm chi phí hạ tầng** do không phải triển khai và vận hành Control Plane trong VPC.
* **Giảm gánh nặng quản trị** Kubernetes, nhờ AWS và Red Hat chịu trách nhiệm quản lý, cập nhật và vá lỗi cụm OpenShift.
* **Tăng khả năng sẵn sàng**, hỗ trợ triển khai đa Availability Zone và nâng cấp độc lập giữa Control Plane và Worker Nodes.

![ROSA HCP Architecture](/images/BlogsPosted/Bài 1/741120668_2240342270073846_7435324291169642636_n.jpg)

---

### 3. Kiến trúc tích hợp Hybrid Cloud

Sau khi đăng ký dịch vụ MuleSoft Anypoint Platform, doanh nghiệp có thể triển khai Runtime Fabric trên ROSA để chạy các ứng dụng tích hợp theo mô hình container.

Quy trình triển khai gồm:
1. Tải các thành phần Runtime Fabric từ Anypoint Platform.
2. Đóng gói thành container image và lưu trên Container Registry.
3. Triển khai lên cụm ROSA bằng Kubernetes/OpenShift.
4. Quản lý toàn bộ API và Integration thông qua Anypoint Platform.

Nhờ nền tảng Kubernetes thống nhất, các workload có thể di chuyển linh hoạt giữa môi trường on-premises và cloud mà vẫn giữ nguyên quy trình vận hành.

---

### 4. Giá trị cho doanh nghiệp

Việc kết hợp MuleSoft Runtime Fabric với ROSA mang lại nhiều lợi ích:
* **Chuẩn hóa nền tảng tích hợp** trên Kubernetes cho Hybrid Cloud và Multi-Cloud.
* **Đơn giản hóa quản lý** API, dịch vụ tích hợp và container trên cùng một nền tảng.
* **Giảm chi phí vận hành** nhờ dịch vụ OpenShift được quản lý hoàn toàn.
* **Dễ dàng mở rộng** khi số lượng API và ứng dụng tăng theo nhu cầu của doanh nghiệp.
* **Tăng tính linh hoạt**, cho phép triển khai workload ở nhiều môi trường mà không cần thay đổi kiến trúc ứng dụng.

---

### Kết luận

Việc triển khai MuleSoft Runtime Fabric trên Red Hat OpenShift Service on AWS (ROSA) mang đến một kiến trúc Hybrid Cloud hiện đại, kết hợp khả năng quản lý API của MuleSoft với nền tảng Kubernetes doanh nghiệp của Red Hat và hạ tầng AWS.

Đây là một mô hình phù hợp cho các doanh nghiệp đang hiện đại hóa hệ thống tích hợp, giúp chuẩn hóa việc triển khai ứng dụng trên Kubernetes, giảm chi phí vận hành và xây dựng nền tảng API linh hoạt cho quá trình chuyển đổi số.

*Chi tiết xem tại bài viết gốc:* [Deploy MuleSoft Runtime Fabric on ROSA for Hybrid Cloud Integration](https://aws.amazon.com/blogs/ibm-redhat/deploy-mulesoft-runtime-fabric-on-rosa-for-hybrid-cloud-integration/)
