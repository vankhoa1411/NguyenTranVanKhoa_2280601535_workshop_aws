---
title: "Deploy MuleSoft Runtime Fabric on ROSA"
date: 2026-07-10
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---
{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# Deploy MuleSoft Runtime Fabric on ROSA: Modernizing Hybrid Cloud Integration with AWS

Many businesses run on-premises systems and cloud resources concurrently, making data integration and API management increasingly complex. The challenge is deploying integration services consistently, escalably, and with reduced operational overhead.

AWS, Red Hat, and MuleSoft introduce a deployment model for MuleSoft Runtime Fabric (RTF) on Red Hat OpenShift Service on AWS (ROSA), helping enterprises build a modern, fully managed Kubernetes-based Hybrid Cloud integration platform.

---

### 1. MuleSoft Runtime Fabric on ROSA

MuleSoft Runtime Fabric (RTF) is a deployment and orchestration platform for Mule applications on Kubernetes, enabling APIs and integration services to run across various environments including on-premises, private cloud, and public cloud.

When deployed on ROSA (Red Hat OpenShift Service on AWS), Runtime Fabric is installed as a Red Hat OpenShift Certified Operator, automating application deployment, management, and scaling using standard Kubernetes mechanisms.

![MuleSoft Runtime Fabric on ROSA](/static/images/BlogsPosted/Bài 1/739437321_2240342243407182_7284734041828767801_n.jpg)

---

### 2. Benefits of ROSA Hosted Control Planes (HCP)

The article focuses on the ROSA Hosted Control Planes (HCP) architecture, where the Control Plane is fully managed by Red Hat, and the enterprise only operates Worker Nodes within their own AWS account.

This model provides several key benefits:
* **Reduced Infrastructure Cost**: Avoid provisioning and operating Control Plane nodes within your own VPC.
* **Minimized Administrative Overhead**: AWS and Red Hat handle OpenShift cluster management, updates, and security patching.
* **High Availability**: Supports multi-Availability Zone deployments and independent upgrades between the Control Plane and Worker Nodes.

![ROSA HCP Architecture](/static/images/BlogsPosted/Bài 1/741120668_2240342270073846_7435324291169642636_n.jpg)

---

### 3. Hybrid Cloud Integration Architecture

After subscribing to the MuleSoft Anypoint Platform, enterprises can deploy Runtime Fabric on ROSA to run containerized integration workloads.

The deployment workflow includes:
1. Downloading Runtime Fabric components from the Anypoint Platform.
2. Packaging them into container images and storing them in a Container Registry.
3. Deploying to the ROSA cluster using Kubernetes/OpenShift.
4. Managing all APIs and Integrations through the Anypoint Platform.

Thanks to a unified Kubernetes platform, workloads can move flexibly between on-premises and cloud environments while maintaining identical operational workflows.

---

### 4. Enterprise Value

Combining MuleSoft Runtime Fabric with ROSA delivers:
* **Standardized Integration Platform**: Consistent Kubernetes runtime for Hybrid Cloud and Multi-Cloud.
* **Simplified Management**: Centralized management of APIs, integrations, and containers.
* **Lower Operating Expenses**: Fully managed OpenShift services minimize operational effort.
* **Seamless Scalability**: Scale APIs and applications dynamically based on business demands.
* **Enhanced Flexibility**: Deploy workloads across multiple environments without restructuring application architectures.

---

### Conclusion

Deploying MuleSoft Runtime Fabric on Red Hat OpenShift Service on AWS (ROSA) offers a modern Hybrid Cloud architecture, combining MuleSoft's API management capabilities with Red Hat's enterprise Kubernetes platform and AWS's cloud infrastructure.

This model is ideal for organizations modernizing their integration systems, standardizing Kubernetes deployments, reducing operational costs, and building a flexible API foundation for digital transformation.

*For more details, refer to the original article:* [Deploy MuleSoft Runtime Fabric on ROSA for Hybrid Cloud Integration](https://aws.amazon.com/blogs/ibm-redhat/deploy-mulesoft-runtime-fabric-on-rosa-for-hybrid-cloud-integration/)
