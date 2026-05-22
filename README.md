# Identity Verification System – Production Architecture

![Architecture Diagram](./architecture.png)

> **AWS Certified Solutions Architect** – A production‑ready identity verification system with liveness detection, OCR, facial matching, and GDPR compliance.

---

## Overview

This project presents a cloud-native Identity Verification System designed using AWS services and modern full-stack engineering practices. The system enables users to securely verify their identity by uploading a government-issued ID document and capturing a live selfie for facial verification.

The architecture focuses on:

- Secure identity verification workflows
- Face comparison and liveness detection
- OCR extraction from ID documents
- Scalable backend APIs
- Cloud-native serverless orchestration
- Security, compliance, observability, and auditability

The solution is designed as a conceptual production-ready architecture suitable for KYC (Know Your Customer), onboarding, fraud prevention, and secure authentication workflows.

### Architecture Goals

**Functional Goals**
- User registration and login
- ID document upload
- Live selfie capture
- Face matching between selfie and ID
- OCR extraction from identity documents
- Verification status tracking
- Existing user facial authentication
- Notification delivery

**Non-Functional Goals**
- Scalability
- Reliability
- Security
- Compliance readiness
- Observability
- Fault tolerance
- Low operational overhead
- Fast response time

---

## High‑Level User Flow

1. **User registers / logs in** – via React frontend.
2. **ID document upload** – image captured or uploaded.
3. **Live selfie capture** – with real‑time liveness guidance.
4. **Secure transmission** – frontend sends data to backend API over HTTPS.
5. **Direct S3 upload via pre‑signed URL** – backend generates URL; frontend uploads directly to S3 (reduces cost & latency).
6. **Step Functions orchestration** – manages the verification workflow:
   - Textract extracts ID data (OCR)
   - Rekognition performs face comparison + liveness detection
7. **Results persisted** – status saved to database, user notified via SES/SNS.
8. **User sees verification result** – approved / rejected / retry.

---

## Component Breakdown

### 1. Frontend (Web)

- **React** application hosted on **AWS Amplify** (or any static hosting).
- Features:
  - User registration / login
  - ID capture / upload (camera or file)
  - Live selfie capture with liveness guidance
  - Real‑time verification status polling

### 2. Backend API (Django)

- **Django REST Framework** – RESTful API endpoints.
- **JWT authentication** – stateless, secure.
- **RBAC / ABAC** – role‑based access control.
- Endpoints:
  - `POST /generate-presigned-url` – returns S3 upload URL
  - `POST /verify` – triggers verification workflow
  - `GET /status/{session_id}` – returns verification result
- Orchestrates the verification workflow (calls Step Functions, updates DB).

**Why Django?**
- **Rapid API development** – built‑in admin, serializers, viewset conventions
- **Strong authentication ecosystem** – JWT, OAuth, session auth ready to use
- **Mature ORM** – complex audit joins across `users`, `verification_requests`, `audit_logs`
- **Django REST Framework** – production‑ready API pagination, filtering, versioning
- **Faster MVP** – proven prototype in 2 weeks before cloud migration

### 2a. Orchestration Layer – AWS Step Functions (Production)

For production scale, the verification workflow is managed by **AWS Step Functions**:

- State machine with built‑in retries, error handling, and parallel execution
- States:
  - `ExtractID` – calls Textract for OCR
  - `CheckLiveness` – calls Rekognition Liveness (anti‑spoofing)
  - `CompareFaces` – calls Rekognition CompareFaces (ID vs selfie)
  - `SearchDuplicateFaces` – checks existing face collection
  - `StoreResult` – writes to database
- Separates business logic from Lambda/Django code
- Visual debugging and execution history

### 3. Database – PostgreSQL

- Tables:
  - `users` – authentication & profile data
  - `roles_permissions` – RBAC
  - `verification_requests` – session tracking
  - `verification_results` – match scores, liveness outcome
  - `audit_logs` – immutable action history
- Encryption at rest (AWS KMS or application‑level)

### 4. Cache / Session Store – Amazon ElastiCache (Redis)

- Session management
- OTP / temporary data storage
- Rate limiting counters

### 5. Notification Service – Amazon SES + SNS

- **SES** – email notifications (verification status, alerts)
- **SNS** – SMS alerts for high‑priority events

### 6. AI / Analysis Services – Amazon Rekognition & Amazon Textract

| Service | Feature | Purpose |
|---------|---------|---------|
| Rekognition | `CompareFaces` | ID photo vs selfie → similarity score (%) |
| Rekognition | `Face Liveness` | Anti‑spoofing (photo, video replay, mask detection) |
| Rekognition | `IndexFaces` | Store face vectors for returning users |
| Rekognition | `SearchFacesByImage` | Duplicate check / existing user login |
| Textract | `AnalyzeDocument` | OCR for ID number, full name, date of birth, expiry |

> **Note:** Document authenticity (tamper detection, holograms, watermarks) is **not** performed by Rekognition alone. A production system would add third‑party KYC providers (Onfido, Jumio) or custom fraud checks.

### 7. Storage Layer – Amazon S3 (with pre‑signed URLs)

- **Encrypted** with AWS KMS (AES‑256)
- **Upload pattern:** Backend generates a pre‑signed URL; frontend uploads **directly to S3**
- **Why:** Avoids routing large image files through application servers (reduces cost & latency, improves reliability)
- **Security:** URL expires in 60 seconds; IAM role only allows PUT to user‑specific prefix
- **Lifecycle policy:** Auto‑delete images after 48 hours (GDPR compliance)
- **Access:** No public access – IAM roles only
- Stores raw ID documents, selfies, thumbnails, and audit logs

### 8. Security & Compliance

| Layer | Implementation |
|-------|----------------|
| In transit | HTTPS / TLS 1.3 (enforced) |
| Authentication | JWT (Cognito or custom) |
| Access control | RBAC/ABAC + IAM least privilege |
| Input validation | API Gateway & Django |
| Data at rest | AWS KMS (S3, RDS, ElastiCache) |
| Audit | CloudTrail + DynamoDB audit logs |
| Data retention | 48‑hour auto‑deletion (S3 lifecycle + DB TTL) |
| Compliance | GDPR / POPIA ready (eu‑west‑1 data residency) |

### 9. Monitoring & Logging

- **CloudWatch** – metrics, logs, dashboards, alarms
- **CloudTrail** – API activity logging (who accessed which S3 object)
- **X‑Ray** – distributed tracing across Step Functions → Lambda → Rekognition → DB
- Alarms on verification failure rate >10%, Lambda errors, etc.

### 10. CI/CD Pipeline

- **CodeBuild** – compile, test, lint
- **CodeDeploy** – deploy to AWS (ECS / Lambda)
- **GitHub Actions** – trigger on push to `main`

### 11. Infrastructure as Code (IaC)

- **Terraform** (or CloudFormation)
- Reproducible environments (dev, staging, prod)
- Version controlled – all infrastructure changes peer‑reviewed

---

## Security & Compliance Highlights

- **All data encrypted** at rest and in transit
- **Auto‑deletion** after 48 hours – supports GDPR "right to erasure"
- **No public S3 access** – pre‑signed URLs for uploads
- **Audit logs** immutable – every verification action recorded
- **Least privilege IAM** – Lambda can only access its own S3 prefix
- **Data residency** – choose AWS region (e.g. `eu‑west‑1` for GDPR)

---

## Scalability & Reliability

| Service | How it scales |
|---------|----------------|
| Frontend | Amplify / S3 + CloudFront – CDN, auto‑scaling |
| API | API Gateway + Lambda – serverless, pay‑per‑request |
| Orchestration | Step Functions – automatic retries, parallel execution |
| Database | RDS read replicas / DynamoDB on‑demand |
| AI | Rekognition & Textract – AWS managed, automatic scaling |
| Storage | S3 – 11 nines durability, auto‑tiering |

---

## Cost Estimation (example – 10k verifications/month)

| Service | Monthly Cost |
|---------|--------------|
| API Gateway | ~$3.50 |
| Lambda | ~$0.50 |
| Step Functions | ~$0.25 |
| Rekognition (CompareFaces + Liveness + Search) | ~$125.00 |
| Textract | ~$75.00 |
| S3 | ~$1.15 |
| RDS (PostgreSQL, smallest instance) | ~$15.00 |
| ElastiCache (Redis, t3.micro) | ~$13.00 |
| SNS + SES | ~$3.00 |
| CloudWatch, X‑Ray, CloudTrail | ~$5.00 |
| **Total (without Macie)** | **~$241** |

*Use AWS Free Tier for initial development. Liveness detection is the primary cost driver (~$0.025/check).*

---

## Deployment Overview

1. **Infrastructure** – `terraform apply` (VPC, RDS, S3, ElastiCache, Lambda, Step Functions)
2. **Backend** – deploy Django container to ECS / Lambda (or keep as EC2)
3. **Frontend** – `npm run build` → upload to S3 / Amplify
4. **CI/CD** – GitHub Actions runs tests then Terraform + deployment
5. **Monitoring** – CloudWatch dashboard auto‑created

---

## Trade‑offs & Design Decisions

### Why no Amazon Macie in this architecture?

- **Cost** – Macie starts at ~$5,000/month. For 10k verifications/month, that's 30x the cost of all other services combined.
- **Alternatives used** – S3 bucket policies (block public access), IAM least privilege, CloudWatch anomaly detection on S3 access patterns, and manual classification of PII fields (ID number, name, DOB) in the application layer.
- **When to add Macie** – If the system grows to >100k verifications/day and stores sensitive documents for longer than 48 hours.

### Why PostgreSQL instead of DynamoDB?

- **Need for complex joins** – audit logs, user‑role permissions, and verification history queries across multiple tables.
- **Familiarity** – Django ORM works seamlessly with PostgreSQL.
- **Future proof** – read replicas and connection pooling scale well.

### Why not fully serverless (Lambda + API Gateway) from day one?

- **Working prototype first** – Django + React proved the business logic in 2 weeks.
- **Migration path** – The API boundaries are identical; moving to Lambda would take <1 week.
- **Development velocity** – Django's admin panel and ORM accelerate feature iteration.

### Why pre‑signed URLs instead of backend proxy uploads?

- **Cost** – Routing images through Lambda adds compute cost per request.
- **Latency** – Direct upload to S3 is faster.
- **Scalability** – S3 handles high throughput natively.

### Why Step Functions instead of a single Lambda orchestrator?

- **Visibility** – Visual workflow debugging in AWS console.
- **Resilience** – Built‑in retries, error handling, and state persistence.
- **Long‑running workflows** – Can wait for external callbacks or human review.

---

## Current Limitations

- **Multi‑region deployment** – not yet optimized; all services in a single AWS region (eu‑west‑1)
- **Rekognition accuracy** – depends on image quality (low‑resolution or obscured faces reduce match confidence)
- **Offline verification** – limited; requires internet connection for API calls and AI services
- **Fraud scoring** – no advanced engine (e.g., device fingerprinting, behavioural analysis, document forgery detection)
- **Document authenticity** – no third‑party KYC provider integration (e.g., Onfido, Jumio) – relies on basic OCR + face match only
- **Rate limiting** – prototype uses simple IP‑based limits; no adaptive or token‑bucket algorithm yet
- **Pre‑signed URL expiry** – currently 60 seconds; may need adjustment for poor network conditions

These limitations are acceptable for MVP / pilot scale (<10k verifications/day) and will be addressed in production hardening.

---

## Next Steps for Production

- Enable **AWS WAF** for additional DDoS / SQL injection protection
- Add **Amazon Macie** for advanced PII discovery (if budget allows)
- Integrate **third‑party KYC provider** for document authenticity checks (holograms, watermarks, tamper detection)
- Implement **AWS Step Functions** for complex retry logic (if not already)
- Set up **Canary deployments** with CodeDeploy
- Add **device fingerprinting** for fraud detection
- Implement **adaptive rate limiting** based on user behaviour

---

## Related Repositories

- **Working Prototype** (Django + React + vanilla JS + face‑api.js) – proves end‑to‑end coding ability
  → [Link to prototype repo]
- **Terraform modules** – infrastructure code for this architecture
  → [Link to IaC repo]

---

**Document version:** 1.1  
**Last updated:** May 2026  
**Author:** Damaris Chege, AWS Certified Solutions Architect  
**License:** MIT