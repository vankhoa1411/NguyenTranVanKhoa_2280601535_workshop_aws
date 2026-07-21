---
title : "WAF Protection"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.12.3. </b> "
---

#### Overview

In this step, we will configure **AWS WAF** to enhance security for the **DocuMind AI** system.

AWS WAF is a Web Application Firewall that helps protect web applications from malicious web requests such as SQL Injection, Cross-Site Scripting, bot traffic, or unusual application requests. In DocuMind AI, WAF can be attached to **Amazon CloudFront** or an **Application Load Balancer** (ALB) when deployed to a production environment.

WAF does not replace backend application security, but provides an additional layer of defense at the edge of the application.

---

#### Objectives of This Step

Upon completing this step, you will:

* Understand the role of AWS WAF in DocuMind AI.
* Create a Web ACL for the application.
* Add AWS Managed Rule Groups.
* Configure rule protections against SQL Injection and XSS.
* Attach the WAF to CloudFront or an ALB (if deployed).
* Verify requests are tracked or blocked.
* Configure logging and monitoring metrics for security events.

---

#### The Role of WAF in DocuMind AI

AWS WAF helps protect:

* Login/Register APIs.
* Document Upload APIs.
* Admin Panel APIs.
* AI Chat/RAG APIs.
* Enterprise Sandbox APIs.
* Endpoints susceptible to abuse or brute-forcing.
* Requests containing SQL Injection or XSS payloads.
* Malicious bot traffic and scrapers.

Threats mitigated by WAF:

```text id="r3hchh"
SQL Injection
Cross-Site Scripting
Bad bot traffic
Suspicious request patterns
Common web exploits
```

---

#### WAF Protection Architecture

WAF placement at the application perimeter:

```text id="s85fhi"
User
  |
  v
AWS WAF
  |
  v
CloudFront or Application Load Balancer
  |
  v
Backend API / Frontend
```

![waf-protection](/images/5-Workshop/5.12-Monitoring-and-Security/5.12.3-waf-protection/waf-protection.png)

> ⚠️ **Screenshot Suggestion:**
> Capture the created AWS WAF Web ACL dashboard showing rule groups and Associated AWS resources (such as CloudFront or ALB).
> Save the image at the path:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.3-waf-protection/waf-protection.png`

---

#### Important Note

AWS WAF cannot be attached directly to a Node.js/Express backend server.

WAF is typically attached to:

```text id="07ck60"
Amazon CloudFront
Application Load Balancer
API Gateway
AppSync
```

For this workshop, if you do not have CloudFront or an ALB deployed, create the Web ACL and describe how it would attach to production resources. In real deployments, place the backend behind an ALB or CloudFront so WAF can inspect traffic.

---

#### Step 1: Open the AWS WAF Console

Log in to the AWS Console, search for:

```text id="rn5w3e"
WAF
```

Select:

```text id="s0w2iu"
AWS WAF & Shield
```

Navigate to:

```text id="ymcxyy"
Web ACLs
```

Click:

```text id="d88mt8"
Create web ACL
```

---

#### Step 2: Configure Web ACL Properties

Input name:

```text id="pt36st"
docmind-web-acl
```

Description:

```text id="avq4l9"
Web ACL for DocuMind AI frontend and backend protection.
```

Select the resource scope type:

* If deploying behind CloudFront:
  `Amazon CloudFront distributions`
* If deploying behind ALB/API Gateway:
  `Regional resources`

Select your target deployment region if using Regional resources.

---

#### Step 3: Associate AWS Resources

If you have an active CloudFront distribution or ALB, select it to associate with the Web ACL:

* `CloudFront Distribution`
* `Application Load Balancer`

If resources are not yet provisioned, you can skip this step and associate them after deploying production resources.

---

#### Step 4: Add AWS Managed Rule Groups

Click:

```text id="rczfkd"
Add managed rule groups
```

Add these key rule groups:

```text id="sdguyv"
AWSManagedRulesCommonRuleSet
AWSManagedRulesKnownBadInputsRuleSet
AWSManagedRulesSQLiRuleSet
AmazonIpReputationList
```

Descriptions:

| Rule Group             | Purpose                                           |
| ---------------------- | ------------------------------------------------- |
| CommonRuleSet          | Protects against common web exploits              |
| KnownBadInputsRuleSet  | Detects and blocks known malicious payloads       |
| SQLiRuleSet            | Detects and blocks SQL Injection payloads         |
| AmazonIpReputationList | Blocks requests from known malicious IP addresses |

---

#### Step 5: Configure Rules to Count Mode First

When deploying rule sets initially, configure rule actions to:

```text id="bgv2s0"
Count
```

This monitors matching request patterns without blocking legitimate traffic.

After confirming no false positives affect valid requests, change actions to:

```text id="56fhg5"
Block
```

This ensures WAF does not accidentally block valid system traffic.

---

#### Step 6: Define Default Web ACL Action

Select the default action:

```text id="u947l1"
Allow
```

This allows normal traffic to pass through unless matched and blocked by a specific rule.

---

#### Step 7: Enable CloudWatch Metrics

Under visibility configurations, enable:

```text id="hdtj85"
CloudWatch metrics
Sampled requests
```

Metric name:

```text id="v16cjl"
docmind-web-acl
```

Enabling metrics helps administrators review request counts, blocks, and rule triggers.

---

#### Step 8: Create the Web ACL

Review configurations:

* Web ACL name.
* Resource type.
* Rule groups added.
* Default action set to Allow.
* CloudWatch metrics enabled.

Then click:

```text id="n21ks7"
Create web ACL
```

---

#### Step 9: Verify SQL Injection Filtering

Simulate a query containing a SQLi pattern:

```text id="1fyasu"
/api/auth/login?email=' OR '1'='1
```

Or test via request body payloads:

```json id="ft5uz2"
{
  "email": "' OR '1'='1",
  "password": "test"
}
```

* If rules are in `Count` mode, requests pass through but increment WAF metric counters.
* If rules are in `Block` mode, requests are blocked with a `403 Forbidden` response.

---

#### Step 10: Verify XSS Filtering

Simulate an XSS payload:

```text id="eq3q75"
<script>alert('xss')</script>
```

Post a request:

```json id="tf90eg"
{
  "question": "<script>alert('xss')</script>"
}
```

The WAF detects the malicious string and acts according to the rule group configuration.

---

#### Step 11: Inspect WAF Metrics

Navigate to the CloudWatch Console:

```text id="n8t1jq"
CloudWatch
→ Metrics
→ AWS/WAFV2
```

Inspect these metrics:

```text id="6o4493"
AllowedRequests
BlockedRequests
CountedRequests
```

Verify that matched requests increment the corresponding counters.

---

#### Step 12: Inspect Sampled Requests

Inside your Web ACL configuration page, click:

```text id="9u2b3i"
Sampled requests
```

This displays recent requests that matched rule groups, assisting in diagnosing false positives.

*Note: Ensure sensitive data is masked in screenshot captures.*

---

#### Troubleshoot Scenarios

| Error                         | Cause                                    | Solution                                  |
| ----------------------------- | ---------------------------------------- | ----------------------------------------- |
| Requests are not blocked      | WAF not attached to CloudFront or ALB    | Check associated resources in WAF console |
| Metrics are not appearing     | CloudWatch metrics disabled              | Enable visibility configuration           |
| Valid requests are blocked    | Rule configurations are too strict       | Change action to Count or add rule exception|
| Cannot attach to CloudFront   | Incorrect Web ACL scope                  | CloudFront requires Global scope Web ACL  |
| Cannot attach to ALB          | Region mismatch                          | Verify Web ACL and ALB reside in same region|
| Backend receives exploit traffic| Backend is not behind WAF/ALB           | Ensure backend is only reachable via ALB  |

---

#### Completion Checklist

You have completed this step when:

* The Web ACL is created for DocuMind AI.
* AWS Managed Rule groups are added.
* CloudWatch metrics are enabled.
* Sampled requests logging is active.
* WAF is attached to CloudFront/ALB (if deployed).
* Simulated SQLi/XSS requests trigger WAF metrics or blocks.
* You understand how to toggle between Count and Block modes.
* Administrators can monitor WAF metrics.

---

#### Expected Outcome

Following this step, DocuMind AI has WAF protection configured at the edge. The system detects or blocks malicious requests, securing the backend and providing security metrics in production.
