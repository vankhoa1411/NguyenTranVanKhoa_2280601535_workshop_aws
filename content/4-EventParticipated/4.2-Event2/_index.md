---
title : "Event 2: GenAI-powered App-DB Modernization Workshop"
date : 2026-05-23
weight : 2
chapter : false
pre : " <b> 4.2. </b> "
---

### Event Objectives & Overview

The 'GenAI-powered App-DB Modernization' workshop focuses heavily on practical architectural methods and next-generation AWS cloud services. The event equips engineers with a comprehensive application modernization mindset to achieve business agility, cost optimization, enhanced security at the Edge, and maximized system performance.

The core objectives include:
- Sharing best practices in modern software application design and operations.
- Introducing Domain-Driven Design (DDD) methodologies and asynchronous event-driven architectures.
- Providing deep technical guidance on selecting and allocating optimal cloud compute resources (Compute services).
- Visualizing groundbreaking Generative AI (GenAI) developer utilities to automate the entire Software Development Lifecycle (SDLC).

### Eminent Speakers List

- **Jignesh Shah** – Director, Open Source Databases
- **Erica Liu** – Sr. GTM Specialist, AppMod
- **Fabrianne Effendi** – Associate Specialist SA, Serverless Amazon Web Services
- **Duc Dao (Mr. Duc Dao)** – Solutions Architect, Cloud Kinetics
- **Nguyen Tuan Thinh (Mr. Nguyen Tuan Thinh)** – DevOps Engineer, First Cloud AI Journey Project
- **Anh Pham (Mr. Anh Pham)** – AWS Community Builder (Security Lead)
- **Pham Ngu Hai Anh (Mr. Pham Ngu Hai Anh)** – AI/BI Solutions Specialist

### Deep Technical Key Highlights

#### 3.1 Core Limitations of Legacy Architectures (Monolithic)
Traditional monolithic architectures present major barriers to business agility, development velocity, and system stability:
- **Prolonged Release Cycles:** Oversized, tightly dependent codebases slow down deployment velocity, leading to delayed feature rollouts, lost revenue, and missed market opportunities.
- **Operational Inefficiencies:** Tightly coupled systems lead to wasteful resource provisioning, reduced developer productivity, and inflated infrastructure costs.
- **Security & Compliance Risks:** Outdated technologies fail to meet modern security standards, leaving workloads vulnerable to exploitation, causing severe reputational damage and data breaches.

#### 3.2 Transitioning to Modern Microservices Architectures
Application modernization requires shifting to a modular system where each business function operates as an independent service communicating via events, built on three technical pillars:
- **Queue Management:** Handling asynchronous tasks, offloading traffic from core backend systems, and smoothing out sudden traffic spikes.
- **Caching Strategy:** Storing temporary data intelligently to reduce load on backend databases and optimize system response latencies.
- **Message Handling:** Establishing flexible communication protocols to ensure seamless data flow across decentralized, independent services.

#### 3.3 Domain-Driven Design (DDD) Methodology
DDD provides a strategic framework to decompose large, complex enterprise systems into manageable, isolated sub-domains through a standardized 4-step process:

<div style="display: flex; flex-wrap: wrap; justify-content: center; align-items: center; gap: 10px; margin: 20px 0; font-weight: bold; font-size: 0.95em;">
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">1. Identify Domain Events</div>
  <div style="color: #64748b; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">2. Arrange Chronological Sequence</div>
  <div style="color: #64748b; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">3. Identify System Actors</div>
  <div style="color: #64748b; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">4. Define Bounded Contexts</div>
</div>

Using a live **Bookstore Case Study**, the speakers vividly demonstrated Context Mapping techniques along with 7 relationship patterns designed to secure and govern cross-context communication.

#### 3.4 Event-Driven Architecture & Integration Patterns
EDA serves as the underlying nervous system connecting decentralized microservices. The three core integration models include:
- **Publish/Subscribe (Pub/Sub):** Broadcasting events to multiple consuming services simultaneously without the publisher requiring knowledge of their destinations.
- **Point-to-Point:** Routing messages directly into a specific queue to process tasks sequentially and accurately.
- **Event Streaming:** Continuously transmitting chronological state changes in real-time, powering high-velocity data analytics pipelines.

> **Trade-offs:** While synchronous REST API calls introduce tight coupling and risk cascading failures, asynchronous communication isolates faults, maximizes scalability, and enhances overall system durability.

#### 3.5 The Evolution of Compute Infrastructure & The Serverless Era
The AWS Shared Responsibility Model exhibits a dramatic shift from manual hardware management toward maximizing operational value:

<div style="display: flex; flex-wrap: wrap; justify-content: center; align-items: center; gap: 10px; margin: 20px 0; font-weight: bold; font-size: 0.95em;">
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">Amazon EC2 (Virtual Machines)</div>
  <div style="color: #64748b; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">Amazon ECS (Container Management)</div>
  <div style="color: #64748b; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">AWS Fargate (Serverless Containers)</div>
  <div style="color: #64748b; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #f0f4f8; color: #1a365d; padding: 8px 12px; border-radius: 6px; border: 1px solid #cbd5e1;">AWS Lambda (Serverless Functions)</div>
</div>

Serverless architectures deliver superior operational benefits: eliminating OS/server administration entirely, providing instant auto-scaling matching request volume, and enforcing a strict pay-for-value pricing model. The criteria for choosing between **Lambda** and **Fargate** depends on task execution duration, cold start latency tolerances, and stateful vs. stateless application characteristics.

#### 3.6 Automating the SDLC with Amazon Q Developer
The AI-powered assistant, Amazon Q Developer, automates tasks across the entire Software Development Lifecycle:
- **Automated Code Transformation:** Accelerating application upgrade velocities (e.g., modernizing legacy Java versions or upgrading legacy .NET codebases).
- **AWS Transform Agents:** Empowering specialized migration campaigns to transition legacy infrastructure—such as VMware virtualization environments, mainframe systems, or legacy .NET workloads—into cloud-native architectures.

#### 3.7 LLM Non-Determinism Despite Temperature = 0 Configurations
An essential scientific insight shared by speaker Duc Dao exposed the misconception of absolute determinism when configuring Large Language Models at a temperature of zero ($temperature = 0$). In deep learning theory, LLMs generate text by calculating raw scores called 'logits' for every word in the vocabulary, which are then passed through a Softmax probability function:
$$P(y_i) = \frac{\exp(y_i / T)}{\sum \exp(y_j / T)}$$
Configuring temperature $T \to 0$ activates Greedy Decoding, forcing the model to systematically select the token with the absolute highest probability ($argmax$). However, rigorous benchmark testing across 5 major LLMs (**GPT-3.5 Turbo, GPT-4o, Llama-3-70B, Llama-3-8B, Mixtral-8x7B**) evaluating 8 distinct NLP tasks over 10 repeated iterations proved that outputs still fluctuate, with baseline Accuracy drifting up to 15%. This variance stems from the non-associative nature of floating-point operations on parallel GPU architectures combined with dynamic batching mechanisms implemented on public, shared endpoints.

#### 3.8 High-Performance Edge Networks, Caching & Security via CloudFront
Designing global-scale infrastructure demands shifting compute logic, security rules, and caching mechanisms to Edge locations closest to end-users using Amazon CloudFront:
- **Edge Logic Processing:** Deploying CloudFront Functions or Lambda@Edge to perform URL rewrites, geo-routing, A/B testing splits, and request filtering directly at the Edge without hitting the origin servers.
- **Multi-Tier Caching Architecture:** Leveraging Regional Edge Caches and Origin Shield to maximize Cache Hit Ratios. Persistent connection reuse eliminates repetitive, latency-heavy TCP/TLS handshakes.
- **Advanced Origin Cloaking:** Protecting origin assets using AWS Origin Access Control (OAC) for S3 buckets or Lambda Function URLs. The VPC Origin solution securely routes traffic into Private Subnets via automatically provisioned ENIs, concealing the origin completely from the public Internet.
- **Mutual TLS (mTLS):** Supporting strict, bidirectional cryptographic certificate verification between clients, the Edge, and the backend origin, satisfying rigid regulatory standards for Financial APIs.
- **Hardware Offloading & Auto-Compression:** Delegating intensive computing tasks like TLS decryption, ECDSA signature verification, and automated data compression (Brotli/Gzip reducing payload sizes up to 82%) to CloudFront Edge reduces EC2 Origin CPU utilization from 5% to below 1%.
- **DDoS Volumetric Mitigation:** Utilizing CloudFront's massive global network capacity (400 GbE network interfaces, Anycast routing, built-in SYN proxy filters) to inspect, absorb, and mitigate massive volumetric DDoS attacks right at the Edge line.

#### 3.9 Shaping Cloud Financials: Fixed-Price CDN & Security Frameworks
Speaker Next Gen Cloud Architect introduced predictable alternatives to standard Pay-as-you-go pricing models, which can fluctuate wildly or spike bills during unexpected traffic surges or DDoS attacks, by leveraging tiered, fixed-price operational plans:
- **Free Tier ($0/month):** Allocates 100GB of data transfer and 1 million requests monthly. Ideal for personal sites, student sandboxes, and baseline security rule testing.
- **Pro Tier ($15/month):** Allocates 50GB of premium data transfer and 10 million requests. Designed for WordPress sites and mid-sized corporate blogs, adding advanced protection against DB, PHP, and SQL Injection attacks.
- **Business Tier ($200/month):** Allocates 500GB of data transfer and 125 million requests. Tailored for enterprise bots management, custom mTLS enforcement, and hiding private origins via VPC integration.
- **Premium Tier ($1,000/month):** Allocates 1.750TB of elite data transfer and 500 million requests. Engineered for fast-growing enterprises, providing dedicated global routing and automated multi-region failover.

#### 3.10 Cognitive Engineering: Shifting from Simple Prompts to Memory & Multi-Agent Frameworks
Presentations by Anh Pham and Pham Ngu Hai Anh fundamentally redefined how developers engage with AI tools:

<div style="display: flex; flex-wrap: wrap; justify-content: center; align-items: center; gap: 10px; margin: 20px 0; font-weight: bold; font-size: 0.95em;">
  <div style="background-color: #fff7ed; color: #9a3412; padding: 8px 12px; border-radius: 6px; border: 1px solid #ffedd5;">Prompt (Isolated Q and A)</div>
  <div style="color: #9a3412; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #fff7ed; color: #9a3412; padding: 8px 12px; border-radius: 6px; border: 1px solid #ffedd5;">Context (Document-Grounded Knowledge)</div>
  <div style="color: #9a3412; font-size: 1.2em;">&rarr;</div>
  <div style="background-color: #fff7ed; color: #9a3412; padding: 8px 12px; border-radius: 6px; border: 1px solid #ffedd5;">Memory (Deep Personalization)</div>
</div>

- **Second Brain Architecture:** Seamlessly blending high-fidelity context with persistent long-term memory systems, adhering to the PARA organization methodology and linking external utilities via the Model Context Protocol (MCP).
- **Amazon Quick Suite Integrations:** Combining 40+ native enterprise data connectors with advanced Amazon Bedrock LLMs to automatically transcribe meeting audio records, compile executive summaries, and perform automated compliance auditing.

#### 3.11 AI System Deployment Roadmaps & ROI Metrics
A standardized 4-phase architecture lifecycle blueprint to graduate GenAI applications from conceptual code prototypes into fully governed Production environments:
- **Phase 1 - Foundation:** Initializing isolated sandboxes, implementing core Agent logic routines, and configuring baseline Amazon Bedrock Guardrails.
- **Phase 2 - Integration:** Linking internal enterprise data repositories, tightening environment access security, and executing thorough User Acceptance Testing (UAT) across APIs.
- **Phase 3 - Pilot:** Rolling out restricted beta releases to production, gathering active end-user feedback loops, and validating compliance standards.
- **Phase 4 - Scale:** Executing wide-scale production deployment, activating real-time observability matrices, and tuning compute efficiencies to capture peak Return on Investment (ROI).

### Key Takeaways

#### Architectural Thinking
- **Business-First Approach:** Software architecture choices must stem directly from concrete business domain realities, establishing a 'ubiquitous language' shared equally by engineering and business teams.
- **Asynchronous Architecture Mandate:** Substituting synchronous API links with asynchronous event-driven integration layers whenever possible to eliminate cascading structural faults and unlock massive elasticity.
- **Leveraging AI Assistants:** Infusing Amazon Q Developer natively into day-to-day engineering workflows to automatically refactor codebases, modernize tech stacks, and systematically drive down technical debt.

### Professional Applications
The knowledge and practical insights gained from this workshop can be directly applied to real-world software development and cloud engineering projects.
- Apply Domain-Driven Design (DDD) to analyze business requirements and design scalable software architectures that align with organizational goals.
- Design and implement Microservices and Event-Driven Architectures to build resilient, loosely coupled systems capable of handling large-scale workloads.
- Select the most suitable AWS compute services, including Amazon EC2, AWS Lambda, AWS Fargate, and Amazon ECS, based on workload characteristics to optimize performance, scalability, and operational costs.
- Utilize Amazon Q Developer to accelerate software development by assisting with code generation, code modernization, technical documentation, debugging, and automated code reviews.
- Improve application performance and security by implementing Amazon CloudFront, Edge Computing, caching strategies, Origin Access Control (OAC), and DDoS protection mechanisms.
- Integrate Amazon Bedrock and Amazon Q Business to develop enterprise Generative AI solutions, such as intelligent document processing, AI-powered assistants, knowledge management systems, and workflow automation.
Adopt modern cloud-native development practices, including Serverless computing, Infrastructure as Code (IaC), continuous integration and continuous deployment (CI/CD), and automated monitoring to improve software quality and operational efficiency.
### Event Experience

Attending the **“GenAI-powered App-DB Modernization”** workshop was a highly valuable experience, providing a multi-dimensional look into modernizing enterprise application layers and database clusters using next-generation automation frameworks.

Key experiential highlights include:
- **Learning from Elite Industry Mentors:** Gathering deep technical insights and real-world production incident case studies from global AWS technical directors and domestic solutions architects.
- **Immersive Technical Demonstrations:** Witnessing live data demonstrating LLM non-determinism across parallel hardware architectures and grasping advanced engineering mechanics like Origin Cloaking and automated DDoS mitigation.
- **Ecosystem Networking:** Engaging face-to-face with professional cloud engineers within the community, discussing methods to optimize actual corporate cloud billing footprints.


**Summary:** Ultimately, this workshop provided deep technical guidance and redefined my approach to building secure, modern cloud architectures. It reinforced the exact infrastructure patterns and edge security controls required for my professional path toward becoming a future Cloud Solutions Architect.