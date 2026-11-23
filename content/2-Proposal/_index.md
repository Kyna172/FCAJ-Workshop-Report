---
title: "Proposal"
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# RELOOP
## AI-Assisted Marketplace for Used and Broken Electronics, Built on a Serverless AWS Architecture

### 1. Executive Summary

ReLoop is a web-based marketplace where users in Vietnam can sell used or broken electronic devices (phones, laptops, monitors, and components) to nearby repair shops, secondhand buyers, and recycling businesses. The project was developed by **AWSS3nt1n4L**, a five-member team participating in the AWS First Cloud AI Journey (FCAJ) program in Ho Chi Minh City, during a 12-week internship (September 14 – December 4, 2026).

Sellers upload device photos, and Amazon Bedrock automatically suggests listing information. Each listing is then matched with all approved buyers whose branch service areas (1–15 km) cover the seller's location. Transactions are finalized based on a buyer's quotation or a price mutually agreed upon through chat.

As of November 8, 2026, the live environment had 14 registered accounts, 12 active listings from 7 sellers, and approximately 31,000 API requests during its first month. The serverless architecture is designed to scale automatically without requiring architectural redesign, with an approved AWS Lambda concurrency limit of up to 1,000.

The platform uses 12 managed AWS services, together with least-privilege AWS IAM permissions: **CloudFront, AWS WAF, S3, Cognito, API Gateway, Lambda, DynamoDB, Rekognition, Bedrock, EventBridge, CloudWatch, and SNS**.

Visitors can browse the public marketplace without an account. Authenticated users (`Owners`) can chat and purchase devices. Admin-approved partners (`Partners`) receive permission to create listings (Tier 1) or purchase devices and submit quotations (Tier 2). Meanwhile, `Admins` and `FinanceAdmins` moderate flagged content and manage monthly service-fee invoices.

{{% notice tip %}}
**Live Demo:** [https://d1cjl1tvcp0r6q.cloudfront.net/](https://d1cjl1tvcp0r6q.cloudfront.net/) (ReLoop's live HTTPS deployment through Amazon CloudFront)  
**Source Code & Terraform Repository:** [https://github.com/madlyham/ReLoop](https://github.com/madlyham/ReLoop)
{{% /notice %}}

#### Team Members

| Member | Role |
|---|---|
| Kung Cẩm Thành | Team Lead |
| Hoàng Quốc Việt | Tech Lead |
| Trần Hoàng Quốc Trung | Backend Engineer |
| Nguyễn Duy Quang | Frontend Engineer |
| Nguyễn Anh Kỳ | Cybersecurity / Documentation |

### 2. Problem Statement

#### Current Challenges

According to the Global E-waste Monitor 2024 (UNITAR/ITU), the world generated 62 million tonnes of electronic waste in 2022, but only 22.3% was formally collected and recycled. The same report estimated that Vietnam generated approximately 516,000 tonnes (about 5.3 kg per person) in 2022.

People who own used or broken electronic devices often do not know who might buy them or how much they are worth. Currently, they typically post devices on general classified websites or social media groups, manually describe the products, contact individual repair shops, and often receive no response.

General marketplaces such as Chợ Tốt and Facebook Marketplace serve many product categories and therefore do not:

- Match listings with repair shops whose service areas cover the seller's location.
- Display estimated prices calculated from buyers' own pricing rules.
- Maintain reputation records showing which buyers honor their quotations.

Small repair shops face a similar challenge: they lack a low-cost channel for sourcing nearby devices that match their purchasing needs.

#### Proposed Solution

ReLoop implements the complete workflow using managed AWS serverless services, eliminating the need to operate servers:

- **Photo Ingestion:** The browser uploads up to 6 images per listing (JPEG/PNG/WebP, maximum 5 MB per image) directly to a private Amazon S3 bucket using short-lived presigned POST URLs. Amazon Rekognition checks each image for sensitive content and extracts text printed on device labels. Amazon Bedrock Guardrails filters the extracted text to flag AI manipulation instructions (prompt injection) or harmful content for Admin review.

- **API & AI Processing:** A single AWS Lambda function (Node.js 22, arm64), behind Amazon API Gateway, handles 111 API routes and asynchronously invokes itself for time-consuming tasks:
  - AI image analysis using Kimi K3 on Amazon Bedrock (approximately 35 seconds per invocation, analyzing up to 3 images to suggest listing information).
  - Matching listings with nearby buyer branches using geohash and Haversine distance calculations.

  Amazon EventBridge invokes the same function every 15 minutes to automatically close listings that have exceeded their 7-day expiration period.

- **Data Storage:** A single on-demand Amazon DynamoDB table (partition/sort keys and one Global Secondary Index) stores all data, including users, partners, branches, pricing rules, listings, quotations, transactions, chats, reviews, expiration counters, invoices, and audit logs. Conditional writes and `Idempotency-Key` headers on fee-related APIs help prevent duplicate operations and data conflicts. S3 lifecycle rules delete unattached images after 1 day and listing images after 90 days.

- **Web Frontend:** A React single-page application (SPA) is served from a private S3 bucket through Amazon CloudFront. It includes the public marketplace, two-tier partner portal, and Admin portal.

- **Security:** Amazon Cognito handles registration and authentication using four groups (`Admins`, `FinanceAdmins`, `Partners`, and `Owners`). A Lambda Post-confirm trigger creates initial user profiles. API Gateway validates Cognito JWTs on all private routes. An AWS WAF web ACL attached to CloudFront applies AWS managed rules and IP-based rate limiting, while a secret `x-origin-verify` header prevents direct requests to API Gateway. Two least-privilege IAM roles separately control permissions for the Lambda functions.

Compared with general classified marketplaces, ReLoop provides AI-assisted listing creation, service-area-based matching, estimated prices based on pricing rules, a two-step transaction completion process (seller confirms the final price, then buyer accepts), and two-way reviews.

ReLoop does not hold customer funds. Buyers and sellers complete payments directly when devices are handed over.

#### Benefits and Return on Investment (ROI)

- **Reduced Manual Input:** AI suggests the title, category, brand, model, and condition from up to 3 uploaded images. Sellers only need to review or make minor adjustments. The team estimates that listing creation time can decrease from approximately 5 minutes of manual entry to under 1 minute.

- **Fewer Unanswered Listings:** Each new listing is automatically delivered to all eligible buyer branches within the service radius, with branches within 5 km prioritized.

- **Reliability:** Clearly defined state machines, DynamoDB conditional writes, and idempotent background jobs maintain consistency across listings, quotations, and transactions. The system includes 182 automated tests and has undergone end-to-end verification directly on AWS.

- **Learning Value:** The team gained practical experience with DynamoDB single-table design, event-driven Lambda processing, structured JSON responses from Amazon Bedrock with Guardrails, AWS WAF configuration on CloudFront, and Infrastructure as Code using Terraform.

- **Cost:** Approximately **USD 11.70/month (USD 140.40/year)** at the current scale (Section 6). Of this amount, **USD 10.00** is attributed to the AWS WAF web ACL on CloudFront. Without WAF, the estimated platform cost would be approximately **USD 1.70/month**. The remaining AWS credits (**USD 189.80**, recorded on November 21, 2026) are sufficient to cover operations until February 28, 2027, without additional cash expenditure.

- **Break-even Point:** Revenue is generated through a fee of VND 10,000 per completed transaction, paid by the seller. Using an approximate exchange rate of USD 1 ≈ VND 26,000 (November 2026), USD 11.70 is equivalent to approximately VND 304,200. The platform reaches break-even at **31 completed transactions per month**, or **5 transactions per month** without WAF.

### 3. Solution Architecture

ReLoop operates entirely on a serverless architecture in the **Asia Pacific (Singapore) Region, `ap-southeast-1`**, with global edge delivery and protection provided by Amazon CloudFront and AWS WAF.

![ReLoop Serverless Architecture on AWS](/images/2-Proposal/architecture.png)

#### End-to-End Processing Flow (Arrows 1 → 10)

1. **Users → CloudFront (`HTTPS (TLS)`):** Browsers connect to CloudFront over HTTPS. **AWS WAF** is attached as an associated web ACL to block malicious IP addresses, common web vulnerabilities, and abnormal request rates.

2. **CloudFront → S3 Website (`static files (OAC)`):** CloudFront retrieves the private React application files from S3 through Origin Access Control (OAC) and performs SPA path rewriting at the edge.

3. **Users → Cognito (`sign up / sign in`):** Users register and authenticate directly through Amazon Cognito. After email verification, Cognito invokes the **AWS Lambda (Post-confirm)** function to create the initial user profile in DynamoDB.

4. **CloudFront → API Gateway (`/api/* + x-origin-verify`):** CloudFront removes the `/api` prefix from incoming `/api/*` requests and forwards them to API Gateway with the secret `x-origin-verify` header, preventing direct access that bypasses CloudFront.

5. **API Gateway → Lambda (API) (`invoke`):** API Gateway validates Cognito JWT signatures through JWKS on all private routes before invoking the main Lambda API function (Node.js 22).

6. **Lambda (API) ↔ DynamoDB (`read/write users, listings & invoices`):** Lambda reads and writes user profiles, partner branches, listings, quotations, chats, expiration counters, and monthly service-fee invoices in a single on-demand table.

7. **Lambda (API) ↔ S3 Photos (`presign & tag`):** Lambda generates short-lived presigned URLs for uploading and viewing images and applies tags indicating moderation and attachment status.

8. **Users ↔ S3 Photos (`upload / view photos`):** Browsers upload and view device images directly through presigned URLs to the private S3 bucket, without transferring image files through Lambda.

9. **Lambda (API) ↔ Rekognition (`image moderation`):** Lambda uses Rekognition to moderate sensitive image content and extract text printed on device labels.

10. **Lambda (API) ↔ Bedrock (`AI analysis + Guardrail`):** Lambda filters extracted text through **Amazon Bedrock Guardrail** and invokes **Kimi K3** on Bedrock in `ap-southeast-1` to generate structured JSON listing suggestions.

**Background and Operational Workflows:**

- **EventBridge → Lambda (API) (`expire listings every 15 min`):** A scheduled job runs every 15 minutes to check due expiration buckets and automatically close expired listings.
- **Lambda (API) → CloudWatch (`logs & metrics`) → SNS (`alarm → email Admin`):** Structured Lambda logs are retained for 7 days. When Lambda errors exceed the configured threshold, an alarm sends an email notification to Admin through SNS.

#### AWS Services Used

- **Amazon CloudFront:** Provides global HTTPS delivery, serves the private React website from S3 through OAC, and forwards `/api/*` requests to API Gateway with the secret `x-origin-verify` header.

- **AWS WAF:** One CloudFront-scope web ACL (`associated web ACL`) containing 3 AWS managed rule groups (IP Reputation, Known Bad Inputs, and Common Rule Set) and 2 IP-based rate limits returning HTTP `429` (100 requests/minute for public APIs and 1,000 requests/minute for the entire system).

- **Amazon S3:** Two private buckets with Block Public Access enabled: one stores the static website build (accessible only through OAC), while the other stores device images (presigned POST/GET URLs and lifecycle rules deleting unattached images after 1 day and all images after 90 days).

- **Amazon Cognito:** An Essentials-tier user pool with email verification, Authorization Code with PKCE, and four groups (`Admins`, `FinanceAdmins`, `Partners`, and `Owners`). Browsers communicate directly with Cognito. A post-confirmation trigger invokes Lambda Post-confirm to create user profiles in DynamoDB. There are currently 14 users.

- **Amazon API Gateway (HTTP API):** Validates Cognito JWTs (JWKS) on private routes and applies throttling at 20 requests/second with a burst limit of 50.

- **AWS Lambda:** Two functions deployed in `ap-southeast-1`:
  - **Lambda API** (Node.js 22, arm64/Graviton, 512 MB): Handles 111 API routes, asynchronous AI and listing-matching tasks, and scheduled expiration jobs. Average execution duration is 144 ms, with a concurrency limit of 1,000.
  - **Lambda Post-confirm** (Python 3.12): Creates the initial user profile in DynamoDB immediately after Cognito account confirmation.

- **Amazon DynamoDB:** One on-demand table (`PK`/`SK` + `GSI1`) with Point-in-Time Recovery (PITR) and Time-to-Live (TTL) enabled. It currently stores approximately 5,600 records (2.4 MB).

- **Amazon Rekognition:** Performs content moderation (rejecting sensitive images and forwarding suspicious cases to Admin) and detects text in all uploaded images.

- **Amazon Bedrock:** Operates in `ap-southeast-1`. A Standard-tier **Guardrail** (`apac.guardrail.v1:0`) filters extracted image text to block prompt injection and harmful content. **Kimi K3** (`global.moonshotai.kimi-k3`) uses the Converse API to generate structured JSON device information. Results are validated against a strict backend schema, and each user is limited to 20 AI analyses per day.

- **Amazon EventBridge:** One recurring schedule (`rate(15 minutes)`) triggers the listing-expiration job.

- **Amazon CloudWatch:** Retains Lambda API logs for 7 days and includes one metric alarm monitoring Lambda errors (threshold: ≥ 3 errors within 5 minutes).

- **Amazon SNS:** Sends email notifications to Admin when the CloudWatch alarm enters the `ALARM` state.

- **AWS IAM:** Two separate least-privilege execution roles: one allows Lambda Post-confirm to create user profiles in DynamoDB, while the other restricts Lambda API access to the main DynamoDB table, S3 image paths, Rekognition, Bedrock models and Guardrails, and CloudWatch Logs.

#### Component Design

- **User Interface (Edge / Client):** A React 18 + TypeScript single-page application (Vite) uses Leaflet + OpenStreetMap for location selection. Visitors can browse the public marketplace. Authenticated users can access the partner portal (Tier 1 individual sellers: up to 3 active listings; Tier 2 purchasing businesses with branches, pricing rules, and quotations: up to 10 active listings) or the Admin portal.

- **Photo Ingestion:**
  1. `POST /uploads/presign` returns a presigned POST URL valid for 5 minutes, allowing the browser to upload directly to private S3.
  2. `POST /uploads/moderate` invokes Rekognition for content moderation and text detection, then applies an approved moderation tag to safe images.
  3. When a listing is created, its images are tagged as attached so that lifecycle rules retain them beyond the 1-day threshold until the 90-day expiration period.

- **Processing:** The HTTP router validates JWT claims, idempotency keys, and input data. State machines for Listings, Quotes, and Transactions permit only valid state transitions. AI analysis and branch matching run asynchronously, while time-bucketed key structures allow the 15-minute EventBridge job to find expired listings without scanning the entire table.

- **Storage:** All queries follow a centralized single-table key design without table scans. Public marketplace listings are partitioned by month, pagination uses validated cursors, and all monetary amounts are stored as integer VND values.

- **Web Interface:** The public marketplace includes listing-specific chat (refreshed every 5 seconds) with price offers, transaction workflows (handover scheduling, price adjustments, confirmation, and disputes), and two-way reviews.

- **User Management:** Cognito handles registration, email verification, and group-based authorization. Admin approves partner applications, while Finance Admin issues and confirms monthly service-fee payments. Invoices overdue by more than 3 days prevent new listings; invoices overdue by more than 7 days result in account suspension until payment.

- **Monitoring:** Structured JSON Lambda logs are retained for 7 days in CloudWatch Logs, with a Lambda error alarm sending email notifications to Admin through SNS.

### 4. Technical Implementation

#### Implementation Phases

1. **Research & Architecture (September 28 – October 8, 2026 — Completed):** Converted the v1.0 specification (1,189 paragraphs) into 45 business decisions (QĐ-01 → QĐ-45), designed all DynamoDB single-table query patterns, and selected a centralized Lambda API architecture behind HTTP API Gateway.

2. **Cost Estimation & Feasibility (October 5–8, 2026 — Completed; Proposal Part 1 submitted on October 8, 2026):**
   - Replaced online bank deposits and withdrawals with direct payments upon device handover.
   - Selected DynamoDB on-demand and Lambda arm64 to remain within the AWS Free Tier under low workloads.
   - Separated AWS service calls into small adapters, enabling the team to develop and test locally without incurring AWS charges.

3. **Development & Initial AWS Deployment (October 9–23, 2026 — Completed):** Developed the main DynamoDB table, S3 photo workflow, Cognito authentication with the Post-confirm trigger, React website, and Terraform infrastructure code. The complete marketplace workflow was deployed to AWS on October 19, and automated AWS smoke tests passed on October 22.

4. **Architecture Refinement (October 26 – November 6, 2026 — Completed):**
   - Redesigned the user interface according to the product design and added the Admin portal.
   - Migrated AI processing to Kimi K3 with Bedrock Guardrail in `ap-southeast-1` (October 28).
   - Completed 5 QA rounds (146 issues addressed, November 3).
   - Increased Lambda concurrency from 10 to 1,000 and attached AWS WAF to CloudFront (November 6).

5. **Verification, Finalization & Handover (November 9 – December 4, 2026 — In Progress):**
   - *Completed (November 9–21):* Submitted Proposal Part 2 (November 13), reran load tests to verify API Gateway throttling (20 requests/second, burst 50) without further 5xx errors, standardized a single deployment workflow with individual IAM users for team members, rebuilt the complete Workshop, and verified architecture arrows 1 → 10 directly on AWS.
   - *Remaining (November 23 – December 4):* Review and clean up unused resources, compare actual costs against remaining credits, rehearse the live demo, and package handover documentation.

#### Technical Requirements

- **Application Stack:** Node.js 22, TypeScript 5, React 18 + Vite 5, Leaflet, Cognito client SDK, AWS SDK v3, esbuild, and Vitest; Python 3.12 for Lambda Post-confirm and the end-to-end verification suite.
- **Infrastructure & CI:** Terraform ≥ 1.10 (AWS provider 6.x) and AWS CLI ≥ 2.32 for consistent infrastructure recreation. GitHub Actions runs type checks, 182 automated tests, and website builds for every pull request.
- **Hardware:** Only team members' laptops are required; no on-premises equipment is needed.
- **Skills:** DynamoDB single-table design, least-privilege IAM, prompt engineering and Bedrock Guardrails, AWS WAF rule tuning on CloudFront, React SPA development, and testing with fake service implementations.

### 5. Roadmap & Milestones

| Week | Period | Status | Deliverables |
|---|---|---|---|
| Week 4 | 05/10–09/10 | Completed | Selected ReLoop as the project topic (October 8); outlined the concept, architecture, and DynamoDB query patterns; submitted Proposal Part 1 (October 8); initialized the main table, Cognito, and React website (October 9). |
| Week 5 | 12/10–16/10 | Completed | Analyzed ReLoop requirements, user roles, and business workflows; planned the AWS serverless architecture (S3, CloudFront OAC, Cognito, API Gateway, Lambda, DynamoDB); implemented direct S3 image uploads and Rekognition moderation. |
| Week 6 | 19/10–24/10 | Completed | Established the development environment using Terraform; deployed the complete marketplace workflow to AWS (October 19); passed end-to-end smoke tests (October 22); implemented the 15-minute EventBridge expiration schedule. |
| Week 7 | 26/10–30/10 | Completed | Introduced the redesigned interface (public marketplace, partner portal, Admin portal); integrated Rekognition text detection with Bedrock Guardrail; migrated AI listing analysis to Kimi K3 (October 28). |
| Week 8 | 02/11–06/11 | Completed | Inventoried ReLoop AWS infrastructure (`ap-southeast-1`, November 4); implemented asynchronous branch matching; completed 5 QA rounds (146 issues addressed, November 3); increased Lambda concurrency to 1,000; enforced a limit of 20 AI analyses per day; attached AWS WAF to CloudFront (November 6). |
| Week 9 | 09/11–13/11 | Completed | Reviewed security, implemented idempotency and security patches, and submitted Proposal Part 2 with actual AWS cost figures (November 13). |
| Week 10 | 16/11–21/11 | Completed | Conducted an overall review and refined the ReLoop AWS architecture diagram; reran load tests after throttling adjustments (20 requests/second, burst 50); standardized deployment with individual IAM users; verified the complete Workshop workflow (arrows 1 → 10). |
| Week 11 | 23/11–27/11 | Completed | Performed final integration verification (S3 image upload limits, Rekognition/Guardrail moderation, AI-assisted listing creation), cleaned up unused resources, reviewed costs, and submitted the completed report (November 24–25). |
| Week 12 | 30/11–04/12 | Planned | Research architectural expansion options (SES, WebSockets, multi-region), explore advanced AWS security, review resources/IAM, and complete handover before December 4, 2026. |

### 6. Budget Estimation

Estimates are based on publicly available AWS pricing for **`ap-southeast-1` (Singapore)** and can be recalculated using the [AWS Pricing Calculator](https://calculator.aws/).

The detailed cost breakdown is available in [reloop-cost-estimate.csv](/files/2-proposal/reloop-cost-estimate.csv).

#### Infrastructure Costs

Region: `ap-southeast-1` (Singapore), with Amazon CloudFront providing the global edge layer.

Usage assumptions are based on actual Cost Explorer and CloudWatch data (approximately 31,000 API requests/month and 14 registered users).

| Service | USD / Month | Assumptions |
|---|---:|---|
| AWS WAF – CloudFront web ACL | 10.00 | 1 web ACL (USD 5.00) + 5 rules (5 × USD 1.00) + USD 0.60 per 1 million requests |
| Amazon Bedrock (Kimi K3 + Guardrail) | 1.00 | Actual monthly usage (~USD 0.87), limited to 20 AI analyses/user/day |
| Amazon Rekognition | 0.30 | Content moderation and text detection for approximately 70–100 images/month |
| Amazon DynamoDB (on-demand) | 0.10 | Approximately 5,600 records, 2.4 MB, with PITR enabled |
| Amazon API Gateway (HTTP API) | 0.10 | Approximately 31,000 requests/month |
| Amazon S3 (website + images) | 0.05 | Private website build and images (1-day unattached / 90-day image lifecycle) |
| Other (data transfer and minor charges) | 0.15 | Allowance for data transfer and small usage charges |
| AWS Lambda (API + Post-confirm) | 0.00 | ~2,250 GB-s/month — Always Free (1 million requests + 400,000 GB-s) |
| Amazon CloudFront | 0.00 | Always Free (1 TB data transfer + 10 million requests/month) |
| Amazon Cognito | 0.00 | 14 users — Free Tier up to 10,000 MAU |
| Amazon CloudWatch + SNS + EventBridge | 0.00 | 7-day logs, 1 alarm, SNS email alerts, 15-minute schedule — Free Tier |
| **Monthly Total** | **11.70** | |
| **12-Month Total** | **140.40** | |

- **One-time Hardware Cost:** USD 0.
- **Credits:** The account is on the AWS Free plan with **USD 189.80** in remaining credits (recorded on November 21, 2026), valid until February 28, 2027. This amount is sufficient to cover approximately 3.2 remaining months (≈ USD 38 at USD 11.70/month) without additional cash expenditure.
- **Cost-saving Option:** Removing the AWS WAF web ACL from CloudFront during periods without load testing would reduce estimated costs to approximately **USD 1.70/month**, at the expense of losing edge-level rate limiting and attack protection rules.
- *Pricing is based on publicly available AWS rates for `ap-southeast-1` in November 2026 and may change. Usage-based charges will increase with traffic.*

### 7. Risk Assessment

#### Risk Matrix

| Risk | Impact | Probability |
|---|---|---|
| R1. Two team members deploy conflicting code to the same Lambda function (occurred on October 28, 2026) | High | Medium |
| R2. AWS credits are exhausted or unexpected charges occur (the Free plan account will be suspended when credits run out) | High | Low |
| R3. Lambda or API throttling during traffic spikes (initial load tests at concurrency 10 produced 26% 5xx errors with 80 simultaneous requests) | Medium | Medium |
| R4. AI misidentifies devices or encounters malicious prompt injection text embedded in images | Medium | Medium |
| R5. Unauthorized access or abuse of the public website, APIs, or private image storage | High | Low |
| R6. Buyers and sellers complete transactions outside the platform to avoid service fees | Medium | High |
| R7. Cognito's default email limit (~50 emails/day) prevents registration during a demo | Medium | Medium |

#### Mitigation Strategies

- **R1:** Use a single deployment workflow based on reviewed commits, assign individual IAM users to team members, and avoid root access keys.
- **R2:** Compare the cost of every paid service against remaining credits before enabling it, maintain a strict limit of 20 AI analyses/user/day, and regularly check AWS Billing.
- **R3:** Lambda concurrency has been increased to 1,000. API Gateway throttling (20 requests/second, burst 50) and AWS WAF rate limiting on CloudFront return controlled `429` responses instead of 5xx errors.
- **R4:** AI only provides suggestions, and sellers must confirm each field before publishing. Rekognition text detection combined with Bedrock Guardrail automatically flags suspicious images for Admin review.
- **R5:** AWS WAF on CloudFront applies managed rules and rate limits. Private S3 buckets use OAC and presigned URLs. API Gateway requires both the secret `x-origin-verify` header from CloudFront and a valid Cognito JWT. IAM roles follow the principle of least privilege.
- **R6:** Apply a listing removal fee when a listing is deleted within 72 hours after a canceled transaction. Fee disputes are handled by Finance Admin, and repeat offenders are flagged.
- **R7:** Plan to migrate to a custom email delivery service before opening registration to a larger audience beyond the small demo environment.

#### Contingency Plans

- **Deployment Failure:** Redeploy the latest stable commit using Terraform and synchronize the static website. Always preview infrastructure changes using `terraform plan`.
- **Unexpected Cost Increase:** Remove the AWS WAF web ACL from CloudFront and immediately lower the daily AI analysis limit.
- **Bedrock Service Interruption:** AI-assisted form completion is optional; sellers can still manually enter information and publish listings.
- **WAF False Positives:** Adjust rules or rate thresholds directly in the CloudFront WAF web ACL. Changes take effect at the edge without redeploying application code.
- **Data Loss or Corruption:** Restore the DynamoDB table using Point-in-Time Recovery (up to 35 days). Sellers can reupload device images if necessary.

### 8. Expected Outcomes

#### Technical Improvements

- A complete serverless marketplace running on AWS, with 111 API routes, 3 user portals, 182 automated tests, and direct verification of all 10 architecture flows.
- Consistently reproducible infrastructure operating at approximately **USD 11.70/month** (**less than USD 1/month** in variable costs for Lambda, DynamoDB, API Gateway, and S3 at the current workload).
- Clearly defined multi-layer security: edge protection through CloudFront WAF, private S3 storage (OAC + presigned URLs), API isolation through the `x-origin-verify` header, Cognito JWT authentication, and image filtering through Rekognition and Bedrock Guardrail.
- Fast response performance: average Lambda API execution duration of 144 ms, supported by 7-day CloudWatch Logs retention and automatic SNS email error alerts.

#### Long-Term Value

- Storing AI-generated suggestions alongside seller-confirmed values creates a labeled dataset that can support future evaluation and improvements in device recognition accuracy.
- The adapter-based backend design allows services to be replaced or extended (such as custom email delivery, WebSocket chat, or banking payment APIs) without rewriting core business logic.
- The same Terraform modules can recreate a separate production environment or expand the platform to other cities, since geohash-based branch matching is not tied to a specific geographic region.
- Repair shops and recycling businesses gain a consistent channel for sourcing nearby used electronics, helping more devices enter formal reuse and recycling processes instead of remaining unused in households.
