---
title: "Proposal"
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# RELOOP
## An AI-assisted marketplace for used and damaged electronics, built on AWS serverless architecture

### 1. Executive Summary

ReLoop is a web-based marketplace that enables users in Vietnam to sell used or damaged electronic devices (phones, laptops, monitors, and components) to nearby repair shops, electronics buyers, and recycling businesses. The project is developed by **AWSS3nt1n4L**, a five-member team participating in the AWS First Cloud Journey (FCAJ) program in Ho Chi Minh City, during a 12-week internship (September 14 – December 4, 2026).

Sellers upload photos, and Amazon Bedrock automatically suggests listing information. The listing is then distributed to all approved buyers whose branch service areas (1–15 km) cover the seller's location. Transactions are finalized through buyer quotations or mutually agreed prices negotiated through chat.

As of November 8, 2026, the development environment had 14 accounts, 12 active listings from 7 sellers, and approximately 31,000 API requests during its first month. The serverless architecture supports scalability without requiring architectural redesign, with an approved Lambda concurrency quota of 1,000.

The primary AWS services include CloudFront, AWS WAF, API Gateway, Lambda, DynamoDB, S3, Cognito, Rekognition, Bedrock, EventBridge, CloudWatch, and SNS.

Guests can browse the Marketplace without an account. Authenticated users can chat and purchase items. Only Admin-approved partners can sell, while content moderation and invoice management are restricted to authorized Admin and Finance Admin roles.

#### Team Members

| Team Member | Role |
|---|---|
| Kung Cẩm Thành | Team Lead |
| Hoàng Quốc Việt | Tech Lead |
| Trần Hoàng Quốc Trung | Backend Engineer |
| Nguyễn Duy Quang | Frontend Engineer |
| Nguyễn Anh Kỳ | Cybersecurity / Documentation |

### 2. Problem Statement

#### Current Challenges

According to the Global E-waste Monitor 2024 (UNITAR/ITU), the world generated 62 million tonnes of electronic waste in 2022, but only 22.3% was formally collected and recycled. According to the same report, Vietnam generated approximately 516,000 tonnes of electronic waste (approximately 5.3 kg per capita) in 2022.

Owners of used or damaged electronic devices often do not know who is willing to purchase their devices or how much those devices are worth. Currently, they rely on general classified websites or social media groups, manually describe their devices, contact individual repair shops, and frequently receive no response.

General marketplaces such as Chợ Tốt and Facebook Marketplace serve a wide range of product categories and therefore lack:

- Listing matching based on repair shops' service areas covering the seller's location.
- Estimated prices calculated according to individual buyers' pricing rules.
- Reputation records reflecting whether buyers honor their quotations.

Small repair shops face similar challenges from the buyer's perspective, as they lack an affordable channel for receiving relevant electronic devices from nearby sellers.

#### Solution

ReLoop operates entirely on managed AWS services without requiring the team to maintain dedicated servers.

- **Ingestion:** browsers upload up to 6 images per listing (JPEG/PNG/WebP, up to 5 MB per image) directly to Amazon S3 using presigned POST URLs. Amazon Rekognition performs image content moderation and text detection. Detected text is evaluated through Amazon Bedrock Guardrails to identify suspicious content and route it for Admin review.

- **Processing:** a single AWS Lambda function (Node.js 22, arm64) serves 111 API routes behind Amazon API Gateway. The function also invokes itself asynchronously for time-consuming tasks:
  - AI-powered image analysis using Kimi K3 on Amazon Bedrock, taking approximately 35 seconds per analysis.
  - Matching listings with buyer branches using geohash and Haversine calculations.

  Amazon EventBridge invokes the same function every 15 minutes to process expired listings, automatically complete transactions after 48 hours, send reminders, and generate monthly invoices.

- **Storage:** a single Amazon DynamoDB table (PK/SK + 1 GSI) stores all entity types, including users, partners, branches, pricing rules, listings, quotations, transactions, chats, reviews, invoices, and Admin audit records. Write operations are conditional, and financial API operations require an `Idempotency-Key`. S3 Lifecycle rules delete unused images after 1 day and all images after 90 days.

- **Web Interface:** a React single-page application served from a private S3 bucket through Amazon CloudFront. It includes a public Marketplace, a two-tier partner portal, and an Admin portal.

- **Security:** Amazon Cognito manages user authentication and authorization. API Gateway validates JWTs on private routes. AWS WAF helps protect the web application and authentication process. The platform applies Content-Security-Policy (CSP) and the principle of Least Privilege through AWS IAM.

Compared with general classified platforms, ReLoop provides AI-assisted listing creation, service-area matching, rule-based price estimation, two-step transaction completion (seller confirms the final price and buyer accepts it), and two-way reviews.

ReLoop does not process payments directly. Both parties handle payments independently during device handover.

#### Benefits and Return on Investment (ROI)

- **Reduced manual input:** AI suggests the title, category, brand, model, and condition of electronic devices from up to 3 images. Sellers only need to confirm or modify the suggestions. Creating a listing takes approximately 5 minutes manually and less than 1 minute with AI assistance, based on the team's estimates.
- **Fewer unanswered listings:** each new listing is automatically matched with buyer branches within their service areas, with branches within 5 km prioritized.
- **Reliability:** state machines, conditional writes, and idempotent background jobs maintain consistency across listings, quotations, and transactions. The project includes 182 automated tests and an end-to-end smoke test running on AWS.
- **Learning value:** the team gains practical experience in DynamoDB single-table design, event-driven Lambda processing, structured JSON responses from Bedrock, Infrastructure as Code using Terraform, AWS WAF configuration, and security testing.
- **Cost efficiency:** approximately **USD 18.70/month (USD 224.40/year)** at the current scale (Section 6). AWS WAF accounts for USD 17.00 of this amount. Without WAF, the estimated infrastructure cost would be approximately USD 1.70/month. Until February 28, 2027, costs are covered by the remaining AWS account credits (USD 197.14 as of November 8, 2026).
- **Break-even point:** revenue is generated through a VND 10,000 fee paid by the seller for each completed transaction. Assuming an approximate exchange rate of USD 1 ≈ VND 26,000 in November 2026, USD 18.70 is equivalent to approximately VND 486,200. Therefore, the platform reaches infrastructure cost break-even at **49 completed transactions per month**, or 5 transactions per month under a cost scenario excluding WAF.

### 3. Solution Architecture

ReLoop uses a serverless architecture with backend resources deployed in the ap-southeast-1 (Singapore) region:

- AWS WAF and Amazon CloudFront protect and distribute web application content. Static files are served from a private S3 bucket, while API requests are forwarded to Amazon API Gateway.
- A single Lambda function contains the application's business logic. It interacts with DynamoDB, S3, Rekognition, and Bedrock through lightweight adapters, allowing the same codebase to run locally using Node.js and in-memory storage.
- Amazon Cognito manages user registration, sign-in, and authentication.
- Application logs and business metrics are collected by CloudWatch. SNS sends email notifications to administrators when alarms are triggered.
- All infrastructure is defined using Terraform, with its state stored in a versioned S3 bucket.

![ReLoop Architecture](https://raw.githubusercontent.com/Kyna172/FCAJ-Workshop-Report/main/static/images/2-Proposal/reloop-architecture.png)

*Original architecture diagram: [reloop-architecture.drawio](https://github.com/Kyna172/FCAJ-Workshop-Report/blob/main/static/images/2-Proposal/reloop-architecture.drawio) (generated using `diagram/make_architecture.py`).*

#### AWS Services Used

- **Amazon CloudFront:** delivers the web application and forwards `/api/*` requests to API Gateway. Implements security headers such as CSP, HSTS, and X-Frame-Options.
- **AWS WAF:** uses two Web ACLs to protect the web application and authentication system, combining AWS Managed Rules with rate-limiting mechanisms.
- **Amazon S3:** uses three buckets for the web application build (private, with OAC), device images (presigned POST/GET and tag-based lifecycle management), and Terraform state (with versioning).
- **Amazon API Gateway (HTTP API):** validates JWTs on private routes and applies throttling to control API traffic.
- **AWS Lambda:** runs Node.js 22 on arm64 (Graviton) with 512 MB of memory. Handles API requests, asynchronous AI and matching workers, scheduled jobs, and Cognito pre-sign-up triggers. The average execution duration is 144 ms, with a concurrency quota of 1,000.
- **Amazon DynamoDB:** uses a single on-demand table with PK/SK and GSI1, with PITR and TTL enabled (notifications automatically expire after 90 days). Currently contains approximately 5,600 records (2.4 MB).
- **Amazon Cognito:** provides an email-verified user pool and manages user authorization groups. Currently supports 14 users.
- **Amazon Rekognition:** performs image content moderation and text detection.
- **Amazon Bedrock:** uses Kimi K3 through the Converse API with tool choice to generate structured JSON responses. Guardrails support content filtering and AI safety.
- **Amazon EventBridge:** uses one scheduled rule that runs every 15 minutes to invoke background processing jobs.
- **Amazon CloudWatch:** provides application logs, custom business metrics, one dashboard, and 10 alarms for system monitoring.
- **Amazon SNS:** delivers email alerts to administrators.
- **AWS IAM:** manages resource access permissions according to the principle of Least Privilege.
- **AWS Security Agent:** performs automated security testing on the development environment (3.63 task-hours on November 8, 2026; remediation updates deployed on November 10, 2026).

#### Component Design

- **Edge / Client:** a React 18 + TypeScript single-page application built with Vite, using Leaflet and OpenStreetMap for location selection. Guests can browse the Marketplace. After authentication, users can access the partner portal (Tier 1: individual sellers, up to 3 active listings; Tier 2: businesses with branches, pricing rules, and quotations, up to 10 active listings) or the Admin portal.

- **Ingestion:**
  1. The system generates a presigned POST URL, allowing browsers to upload images directly to Amazon S3.
  2. Amazon Rekognition performs image content moderation.
  3. When a listing is created, the system updates the image status to support appropriate storage lifecycle management.

- **Processing:** the HTTP router handles authentication, idempotency, and input validation. State machines for Listing, Quote, and Transaction enforce valid state transitions. AI processing and listing matching run asynchronously. The background job runner organizes tasks by time intervals to identify scheduled work without scanning the entire table.

- **Storage:** data access patterns are defined in `backend/src/store/keys.ts`. Marketplace listings are partitioned by month, with cursor-based pagination and cursor format validation. Monetary values are stored as integer VND amounts.

- **Web Interface:** a public Marketplace, listing-specific chat (updated every 5 seconds) with price negotiation, transaction management (handover scheduling, price adjustments, confirmation, and disputes), and reviews.

- **User Management:** Cognito registration requires email verification. The system validates registered accounts. Administrators approve partner applications, while Finance Admins generate and confirm monthly invoices. Sellers with invoices overdue by more than 3 days cannot create new listings; accounts with invoices overdue by more than 7 days are suspended.

- **Monitoring:** structured JSON logs, metric filters for business events, CloudWatch dashboards, SNS email notifications, and WAF log management with sensitive information protection.

### 4. Technical Implementation

#### Implementation Phases

1. **Initial Project Research and Project Selection (September 28 – October 9, 2026):**
   - Formed the team and researched an AI-powered solution for analyzing and detecting suspicious emails.
   - On October 8, the ReLoop concept was proposed in the team's shared document.
   - On October 9, the team held an in-person discussion at the office and officially agreed to proceed with the development of ReLoop.

2. **ReLoop Requirements Analysis, Architecture Design & Feasibility Assessment (from October 9, 2026):**
   - Analyzed business requirements, identified core features, and established the serverless architecture approach on AWS.
   - Identified DynamoDB access patterns and the approach of combining AWS Lambda with Amazon API Gateway HTTP API.
   - Removed bank deposit and withdrawal functionality due to the lack of banking APIs, replacing it with direct payment during device handover.
   - Selected DynamoDB on-demand and Lambda arm64.
   - Abstracted AWS service calls behind interfaces so that team members could develop locally without incurring AWS costs.
   - Evaluated implementation costs and technical feasibility.

3. **Development & Initial Deployment (October 9 – October 23, 2026):**
   - Started application development on October 9 with the main database table, image bucket, Cognito authentication, and React application.
   - Created the initial Terraform configuration for Cognito, API Gateway, Lambda, S3, and CloudFront.
   - Completed the full Marketplace workflow on AWS (October 19), including the two-tier partner model, quotations, chat-based price negotiation, transactions, reviews, service fees, and content moderation.
   - Successfully completed the end-to-end smoke test `smoke-aws.py` on October 22.

4. **Architecture Refinement (October 26 – November 6, 2026):**
   - Redesigned the entire user interface based on the Product Owner's design and added a new Admin portal.
   - Migrated AI processing to Kimi K3 on Amazon Bedrock (October 28).
   - Conducted five QA rounds covering 146 items (November 3).
   - Increased the Lambda concurrency quota from 10 to 1,000.
   - Integrated AWS WAF with CloudFront and Cognito (November 6).

5. **Testing, Security Hardening & Handover (November 9 – December 4, 2026):**
   - Conduct security testing and remediation (week of November 9).
   - Apply CSP security policies.
   - Standardize the CI/CD deployment workflow and manage access permissions for individual team members.
   - Implement SES for email, AWS Budgets, load testing, and the production environment.
   - Submit Proposal Part 2 with actual cost figures and security testing results (submitted on November 13, 2026).
   - Prepare final documentation and a demonstration according to the team's plan.

#### Technical Requirements

- **Software:** Node.js 22, TypeScript 5, React 18 + Vite 5, Leaflet, amazon-cognito-identity-js, AWS SDK v3, esbuild, Vitest, and Python 3 for AWS smoke testing.
- **Infrastructure as Code:** Terraform ≥ 1.10 (AWS provider 6.x), state stored in S3; `infra/deploy.sh` (build → `terraform apply` → web upload → CloudFront cache invalidation); AWS CLI ≥ 2.32.
- **CI:** GitHub Actions performs type checking, runs 182 tests, and builds the web application for every pull request.
- **Hardware:** only team members' laptops are required, with no on-premises infrastructure.
- **Skills:** DynamoDB single-table modeling, IAM Least Privilege, Bedrock prompt and Guardrails design, AWS WAF configuration, React development, and testing with simulated AWS service components.

### 5. Timeline & Implementation Milestones

The following timeline reflects my learning, technical research, and contributions to the ReLoop project in my cybersecurity and documentation role. Major project milestones are aligned with the team's shared timeline, while weekly activities reflect my individual responsibilities and internship experience.

| Week | Period | Activities and Deliverables |
|---|---|---|
| Week 1 | 14/09–18/09 | Set up my AWS account, FCAJ Portal, and personal GitHub repository; learn program guidelines, AWS Credits, and cloud computing fundamentals. |
| Week 2 | 21/09–26/09 | Study and practice AWS IAM and Amazon VPC; strengthen knowledge of identity management and network security; attend the Buildrathon Kickoff event on September 26. |
| Week 3 | 28/09–02/10 | Study and practice Amazon EC2; attend Cloud & AI events; participate in initial team formation and research an AI-powered solution for analyzing suspicious emails. |
| Week 4 | 05/10–09/10 | Explore Amazon RDS and AWS Budgets; evaluate the initial project concept; ReLoop is proposed on October 8 and officially selected by the team on October 9. |
| Week 5 | 12/10–16/10 | Explore Amazon CloudWatch and Hybrid DNS with Amazon Route 53 Resolver; study ReLoop's objectives, features, and business workflows; finalize team roles on October 16. |
| Week 6 | 19/10–24/10 | Study ReLoop's system workflows; participate in initial demo testing from October 21; document UI and functional issues, suggest improvements, and attend the FCAJ Meetup. |
| Week 7 | 26/10–30/10 | Explore AWS Backup and Amazon ECS; perform regression and end-to-end workflow testing on the updated ReLoop application; write a technical blog about Amazon Bedrock cost management. |
| Week 8 | 02/11–06/11 | Consolidate internship documentation; research AWS Budgets, Cost Anomaly Detection, and Cost Explorer; study the roles of AWS WAF and API Gateway Throttling in web application protection. |
| Week 9 | 09/11–14/11 | Finalize and publish technical blogs about API security and Amazon Bedrock usage management; consolidate technical documentation and collaborate with the team on Proposal Part 2 on November 13; attend Cloud Mastery. |
| Week 10 | 16/11–20/11 | Strengthen AWS WAF knowledge through a workshop; review the Proposal against project progress; discuss the Workshop structure, scope, and required supporting materials with the team. |
| Week 11 | 23/11–27/11 | Explore AWS cost visualization and analysis using Cost Explorer; collaborate on Workshop development, technical documentation, and diagrams; learn about report website deployment using Hugo and GitHub Pages. |
| Week 12 | 30/11–04/12 | Finalize the Worklog, Self-Evaluation, Feedback, and remaining report sections; deploy and review the bilingual report website; prepare the required materials for final internship report submission. |

#### Post-launch Project Operation Plan

According to the team's shared plan, ReLoop's development and production environments will continue operating within the AWS credit validity period until February 28, 2027, to monitor operational costs and collect actual data on AI usage and listing-matching effectiveness.

*The team officially agreed to proceed with ReLoop on October 9, 2026. Activities before this date were part of the learning, team formation, and initial project research stages.*

*Ongoing activities in Week 10 and milestones from Week 11 onward will be updated according to actual progress.*

### 6. Budget Estimation

The cost estimation is based on publicly available AWS pricing for the ap-southeast-1 region (use the [AWS Pricing Calculator](https://calculator.aws/) to recalculate the estimates).

The detailed cost breakdown is available in [reloop-cost-estimate.csv](https://github.com/Kyna172/FCAJ-Workshop-Report/blob/main/static/files/2-proposal/reloop-cost-estimate.csv).

#### Infrastructure Costs

Region: ap-southeast-1 (Singapore). Usage statistics are based on actual Cost Explorer and CloudWatch data collected from September 26 to November 8, 2026: approximately 31,000 API requests and 14 users.

| Service | USD / Month | Assumptions |
|---|---|---|
| AWS WAF – CloudFront Web ACL | 10.00 | 1 Web ACL (5.00) + 5 rules (5 × 1.00) + 0.60 per million requests |
| AWS WAF – Cognito Web ACL | 7.00 | 1 Web ACL (5.00) + 2 rules (2 × 1.00) |
| Amazon Bedrock (Kimi K3 + Guardrails) | 1.00 | October usage (USD 0.87), with AI usage limits |
| Amazon Rekognition | 0.30 | Content moderation and text detection for approximately 70–100 images/month |
| Amazon DynamoDB (on-demand) | 0.10 | Approximately 5,600 records, 2.4 MB, PITR enabled |
| Amazon API Gateway (HTTP API) | 0.10 | Approximately 31,000 requests/month |
| Amazon S3 | 0.05 | Web build, images (1/90-day lifecycle), and Terraform state |
| Other (data transfer and miscellaneous charges) | 0.15 | Allowance for minor charges observed in October |
| AWS Lambda | 0.00 | Approximately 2,250 GB-s/month — Free Tier (1 million requests + 400,000 GB-s) |
| Amazon CloudFront | 0.00 | Free Tier (1 TB + 10 million requests/month) |
| Amazon Cognito | 0.00 | 14 users — Free Tier of up to 10,000 MAUs |
| Amazon CloudWatch + SNS + EventBridge | 0.00 | 10 alarms, 1 dashboard, 7-day log retention, email alerts, and 1 scheduled rule — Free Tier |
| AWS Security Agent | 0.00 | Free trial (400 task-hours/month); 3.63 task-hours consumed |
| **Total Monthly Cost** | **18.70** | |
| **Total Annual Cost** | **224.40** | |

- **One-time hardware cost:** USD 0.
- **AWS Credits:** the account is currently on the AWS Free plan, with USD 197.14 in remaining credits as of November 8, 2026, expiring on February 28, 2027. This amount is sufficient to cover approximately 3.7 months from the reporting date (approximately USD 69) without out-of-pocket expenses, provided that usage does not increase significantly.
- **Cost Analysis:** AWS WAF accounts for USD 17.00 of the total monthly cost of USD 18.70. The remaining components account for approximately USD 1.70/month. Cost optimization must balance financial efficiency with the platform's security and operational requirements.

*Reference prices are based on AWS pricing for the ap-southeast-1 region in November 2026 and are subject to change. Usage-based costs will increase with traffic volume. The figures presented in this table represent the team's estimates at the time of preparing the Proposal.*

### 7. Risk Assessment

#### Risk Matrix

| Risk | Impact | Probability |
|---|---|---|
| R1. Different code versions deployed to the same Lambda function from two machines (occurred on 28/10/2026) | High | Medium |
| R2. AWS credit exhaustion or unexpected charges | High | Low |
| R3. Lambda/API throttling during sudden traffic increases | Medium | Medium |
| R4. Incorrect AI-based device identification or AI affected by inappropriate image content | Medium | Medium |
| R5. Security vulnerabilities in the public website or API | High | Low |
| R6. Buyers and sellers completing transactions outside the platform to avoid service fees | Medium | High |
| R7. Default Cognito email limits affecting user registration during demonstrations | Medium | Medium |

#### Mitigation Strategies

- **R1:** standardize the deployment workflow using committed source code, gradually transition to CI/CD, and assign individual access permissions to team members. Monitor deployment status and CloudTrail logs to ensure traceability.
- **R2:** evaluate the cost of each new feature against the remaining AWS credits before implementation. Apply AI usage limits and configure AWS Budgets alerts at 50% and 80%.
- **R3:** increase the Lambda concurrency quota to 1,000 and combine API Gateway throttling with AWS WAF rate-limiting mechanisms to control incoming traffic.
- **R4:** AI provides suggestions only, and sellers confirm each field. The system stores AI-generated suggestions alongside seller-confirmed values. Guardrails and Rekognition help detect inappropriate content, supported by the Admin moderation process.
- **R5:** implement AWS WAF, CSP, JWT authentication, resource access controls, and periodic security testing. The latest security assessment identified 7 findings, none rated high severity. Remediation updates were deployed on November 10, 2026.
- **R6:** apply service fee policies to transactions initiated through the platform, handle fee-related disputes through Finance Admins, and monitor repeated violations.
- **R7:** migrate Cognito email delivery to Amazon SES before opening the platform to trial users.

#### Contingency Plans

- **Deployment failure:** restore the most recent stable version from version-controlled source code. Use Terraform to review and control infrastructure changes before deployment.
- **Unexpected cost increases:** review service usage, adjust AI usage limits and non-essential workloads, and maintain necessary security measures.
- **Bedrock unavailable:** AI assistance is optional. Sellers can manually complete listing forms and continue publishing listings.
- **WAF or CSP incorrectly blocks legitimate users:** review logs, evaluate security configurations, and apply appropriate adjustments after testing.
- **Data incidents:** restore DynamoDB data using Point-in-Time Recovery (PITR), with a recovery window of up to 35 days. Sellers can re-upload images when necessary.

### 8. Expected Outcomes

#### Technical Improvements

- A complete Marketplace running on AWS, featuring 111 API routes, 3 user-facing portals, 182 automated tests, and a successful end-to-end smoke test on AWS.
- Infrastructure that can be fully reproduced using Terraform with a single command, at an estimated cost of USD 18.70/month. Usage-dependent components (Lambda, DynamoDB, API Gateway, and S3) account for less than USD 1/month at the current traffic volume.
- Security posture that can be evaluated through testing results: AWS WAF protection for key components, CSP implementation, and security testing with 7 findings, none rated high severity. Remediable issues have been addressed according to the team's recorded results.
- Response performance: an average Lambda execution duration of 144 ms, supported by CloudWatch Alarms for monitoring system latency.

#### Long-term Value

- Each listing stores AI-generated suggestions alongside seller-confirmed values. Over time, this creates a labeled dataset that can be used to evaluate or improve device identification capabilities.
- The adapter-based design (Table, ImageStore, Analyzer, AuthProvider) allows services to be replaced or extended, such as SES-based email delivery, WebSocket chat, or banking payment APIs, without modifying the core business logic.
- The same Terraform configuration and deployment scripts can be used to establish a production environment or expand the platform to another city. Geohash-based listing matching can operate across different geographical locations.
- Repair shops and recycling businesses gain an additional reliable channel for receiving used electronic devices from nearby sellers. This contributes to device reuse and formal recycling rather than allowing unused electronics to accumulate in households.
