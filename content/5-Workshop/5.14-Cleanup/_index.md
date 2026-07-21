---
title : "Clean up resources"
date : 2024-01-01
weight : 14
chapter : false
pre : " <b> 5.14. </b> "
---

#### Clean Up Resources After Workshop

After completing the testing of the **DocuMind AI** system, cleaning up AWS resources is highly important to avoid incurring unwanted costs in your account.

---

#### Detailed Cleanup Procedure

1. **Clean up Local PostgreSQL Database**:
   - Open a terminal or SQL client and connect to the local database.
   - Run the SQL command to drop the test database:
     ```sql
     DROP DATABASE documind;
     ```
   - (Optional) Stop the PostgreSQL service if no longer needed:
     ```bash
     brew services stop postgresql
     ```

> ⚠️ **Screenshot Suggestion:**
> ![DROP DATABASE documind](/images/5-Workshop/5.6-Cleanup/pg-delete.png)

---

2. **Delete Amazon S3 Bucket**:
   - Access the **S3 Console**, select the bucket `documind-assets-<your-name>`.
   - Click **Empty** to permanently delete all document files inside first, then type `permanently delete` to confirm.
   - Return to the Bucket list, select the bucket, click **Delete**, and type the bucket name to confirm deletion.

> ⚠️ **Screenshot Suggestion:**
> ![documind-assets-<your-name>](/images/5-Workshop/5.6-Cleanup/s3-delete.png)

---

3. **Delete Amazon SQS Queue**:
   - Access the **SQS Console**, select the queue `documind-processing-queue`.
   - Click **Delete** on the top menu bar and confirm the deletion.

---

4. **Delete Other Security Configurations**:
   - **AWS Secrets Manager**: Select the secret `documind/config`, click **Actions > Delete secret**, and set the minimum waiting period (7 days) for the system to schedule its deletion.
   - **AWS WAF**: Select your Web ACL, click **Delete** (make sure to disassociate it from the API Gateway or Load Balancer before deleting).

---

#### Workshop Summary
Congratulations on successfully designing, integrating, and securing the **DocuMind AI** system on the AWS Cloud environment! Through this hands-on workshop, you have mastered building an intelligent information extraction pipeline using a resilient asynchronous architecture.
