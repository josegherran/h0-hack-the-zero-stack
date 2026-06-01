# ContractLens — Engineering Improvement Plan

> **Version:** 1.0.0  
> **Baseline:** SPEC v1.1.0 · ARCHITECTURE v1.1.0  
> **Date:** June 2026  
> **Scope:** Post-hackathon path to production-grade SaaS  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State & Gaps](#2-current-state--gaps)
3. [Quality Characteristics Matrix](#3-quality-characteristics-matrix)
4. [Improvement Waves](#4-improvement-waves)
   - [Wave 0 — Pre-Launch Foundations](#wave-0--pre-launch-foundations-days-1-7)
   - [Wave 1 — Quick Wins](#wave-1--quick-wins-weeks-1-4)
   - [Wave 2 — High-Impact Investments](#wave-2--high-impact-investments-months-1-3)
   - [Wave 3 — Scale & Enterprise](#wave-3--scale--enterprise-months-3-6)
5. [Wave Dependency Map](#5-wave-dependency-map)
6. [Follow-Up Metrics](#6-follow-up-metrics)
7. [Risks & Mitigations](#7-risks--mitigations)
8. [Conclusion](#8-conclusion)

---

## 1. Executive Summary

ContractLens enters post-hackathon production with a well-reasoned hybrid architecture: Vercel for the synchronous user-facing layer, AWS for durable async processing. The core design decisions — presigned S3 uploads, SQS-decoupled Lambda analysis, Aurora Serverless v2, and IAM-native Bedrock access — are sound.

However, a gap analysis across eight quality attributes reveals **twenty-one concrete improvements** spanning security, observability, cost, performance, scalability, maintainability, availability, and portability. Left unaddressed, the highest-risk items (long-lived IAM credentials in Vercel, polling-induced DB cost, no distributed tracing, no rate enforcement) will surface as incidents within the first 30 days of real user traffic.

This plan organizes those improvements into four phased waves, ordered by **business value vs. implementation effort**. Wave 0 must be completed before any public traffic. Waves 1–3 extend the system progressively toward enterprise-readiness.

**Priority summary:**

| Wave | Theme | Items | Effort | Target |
|---|---|---|---|---|
| 0 | Pre-launch safety net | 6 | Low–Medium | Before launch |
| 1 | Quick wins | 6 | Low | Weeks 1–4 |
| 2 | High-impact investments | 6 | Medium–High | Months 1–3 |
| 3 | Scale & enterprise | 3 | High | Months 3–6 |

---

## 2. Current State & Gaps

### 2.1 What Works Well

| Strength | Evidence |
|---|---|
| **Async decoupling** | SQS + Lambda correctly isolates long-running Bedrock calls from Vercel's 60s timeout ceiling |
| **IAM-native LLM auth** | Lambda → Bedrock via execution role eliminates a secret from the Lambda environment |
| **Atomic analysis writes** | Single Aurora transaction prevents partial `analyses + findings + key_dates` records |
| **Direct S3 upload** | Presigned URLs bypass Vercel's 4.5 MB body limit cleanly |
| **Multi-tenant scoping** | `organization_id` filter on every query is the correct pattern for shared-infrastructure SaaS |
| **Type-safe boundaries** | Zod at API edges + Drizzle ORM + Orval codegen creates end-to-end type safety |

### 2.2 Gap Inventory

The following gaps were identified by applying ISO 25010 quality characteristics to the documented design.

#### Security Gaps

| ID | Gap | Severity |
|---|---|---|
| SEC-01 | Vercel API Routes use long-lived IAM user credentials (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) — if these leak, an attacker gains S3, SQS, SES, and Bedrock access with no expiry | **Critical** |
| SEC-02 | No rate limiting or quota enforcement at the API layer — the Free plan allows 3 contracts/month but nothing prevents a script from submitting 300, each triggering a Bedrock call | **High** |
| SEC-03 | Uploaded PDFs are stored with no malware or content-type validation beyond client-side MIME check — a crafted file could exploit pdf-parse parsing | **High** |
| SEC-04 | Chat endpoint injects full contract text into the system prompt with no token budget guard — a large contract could exhaust Bedrock's context window or be used for prompt injection | **Medium** |
| SEC-05 | No Content Security Policy (CSP) headers defined — react-pdf loads pdfjs workers dynamically, which requires careful CSP configuration to avoid XSS vectors | **Medium** |
| SEC-06 | No audit log for compliance-sensitive events (contract viewed, contract deleted, member removed) | **Medium** |

#### Observability Gaps

| ID | Gap | Severity |
|---|---|---|
| OBS-01 | No correlation ID threaded from the Vercel API Route → SQS message → Lambda execution — a failing analysis cannot be traced end-to-end | **High** |
| OBS-02 | No Dead Letter Queue depth alerting — failed jobs accumulate silently with no on-call notification | **High** |
| OBS-03 | No application-level error tracking (Sentry or equivalent) — Vercel logs capture stdout but not aggregated error rates or stack traces | **Medium** |
| OBS-04 | No SLO/SLA definition and no p50/p95/p99 latency metrics for the analysis pipeline — no baseline to detect regressions | **Medium** |

#### Cost Gaps

| ID | Gap | Severity |
|---|---|---|
| CST-01 | 2-second polling creates O(N_active_uploads) Aurora Data API calls per second — at 100 concurrent uploads this is 50 req/s of paid Data API calls even when zero analyses have completed | **High** |
| CST-02 | `raw_llm_output` JSONB stored indefinitely for every analysis version — at scale this dominates storage costs with no archival or TTL strategy | **Medium** |
| CST-03 | No AWS cost allocation tags — impossible to attribute spending to feature areas (upload, analysis, chat, email) | **Low** |

#### Performance Gaps

| ID | Gap | Severity |
|---|---|---|
| PRF-01 | No Aurora connection pooling — every Lambda invocation opens a fresh Aurora Data API connection; under concurrent load (>50 Lambda instances) this risks connection exhaustion | **High** |
| PRF-02 | `GET /api/contracts` and `GET /api/dashboard/stats` have no documented pagination or result limit — returning all records for a large organization will degrade with data growth | **Medium** |
| PRF-03 | react-pdf / pdfjs-dist ships a ~3 MB worker bundle — no documented code splitting or lazy loading strategy | **Medium** |
| PRF-04 | Chat endpoint re-fetches and re-injects the full contract text on every message — no server-side caching of extracted contract text | **Low** |

#### Availability Gaps

| ID | Gap | Severity |
|---|---|---|
| AVL-01 | Aurora Serverless v2 cold start latency of 1–2 seconds after idle periods is not accounted for in the 1.5s dashboard load NFR | **Medium** |
| AVL-02 | SQS visibility timeout of 5 minutes may expire before Lambda completes a slow analysis — the message would be re-queued, triggering a duplicate analysis | **Medium** |
| AVL-03 | No health check endpoint defined — load balancers and uptime monitors have no canonical signal | **Low** |

#### Maintainability Gaps

| ID | Gap | Severity |
|---|---|---|
| MNT-01 | No testing strategy — no unit tests for the Lambda analysis worker, no integration tests for Aurora writes, no end-to-end tests for the upload→analysis→display flow | **High** |
| MNT-02 | No local development environment for Lambda — developers must deploy to AWS to test Lambda changes, slowing the feedback loop | **Medium** |
| MNT-03 | Database migration strategy is "a dedicated script" — no zero-downtime migration pattern for columns added to high-traffic tables | **Medium** |

---

## 3. Quality Characteristics Matrix

Each quality attribute is scored on current maturity (1–5) and its business criticality for a B2B SaaS handling legal documents.

| Quality Attribute | Current Score | Target Score | Business Criticality | Top Gap |
|---|---|---|---|---|
| **Security** | 3/5 | 5/5 | 🔴 Critical | Long-lived IAM keys in Vercel (SEC-01) |
| **Observability** | 2/5 | 4/5 | 🔴 Critical | No correlation ID or DLQ alerting |
| **Availability** | 3/5 | 4/5 | 🟠 High | Aurora cold start vs. load NFR |
| **Performance** | 3/5 | 4/5 | 🟠 High | No connection pooling; unbounded polling cost |
| **Cost Efficiency** | 3/5 | 4/5 | 🟠 High | Polling-driven DB query volume |
| **Maintainability** | 2/5 | 4/5 | 🟠 High | No test coverage at any layer |
| **Scalability** | 3/5 | 4/5 | 🟡 Medium | No read replicas; polling amplification |
| **Portability** | 2/5 | 3/5 | 🟡 Medium | Hard Vercel + Clerk + Bedrock coupling |

**Scoring rubric:** 1 = absent/broken, 2 = partial, 3 = functional but gaps, 4 = production-ready, 5 = industry-leading.

---

## 4. Improvement Waves

### Wave 0 — Pre-Launch Foundations (Days 1–7)

> **Goal:** Eliminate blocking risks before the first real user accesses the system. Every item in this wave has high severity and low-to-medium implementation effort.

---

#### W0-1: Replace Long-Lived IAM Keys with OIDC Federation

**Addresses:** SEC-01  
**Quality:** Security  
**Business value:** Critical — a leaked key enables unbounded Bedrock usage billed to your account and full S3 access to customer contracts.

**Current state:** `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are stored as Vercel environment variables for the chat and presigned URL endpoints.

**Target state:** Configure AWS IAM Identity Center (or a dedicated OIDC provider) to issue short-lived credentials to Vercel's deployment environment via GitHub Actions OIDC, then use an IAM role assumed at runtime.

**Implementation approach:**
1. Create an IAM role (`contractlens-vercel-api-role`) with the scoped permissions currently held by the IAM user: `s3:PutObject` (presigned), `sqs:SendMessage`, `bedrock:InvokeModelWithResponseStream`.
2. Configure a trust policy allowing Vercel's OIDC provider to assume the role.
3. In Vercel API Routes, use `@aws-sdk/credential-providers` with `fromTokenFile()` or `fromWebIdentity()` to acquire temporary credentials at request time.
4. Remove `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from Vercel environment variables.

**Immediate fallback:** If OIDC setup is not feasible before launch, rotate the IAM user keys to 90-day expiry and enable CloudTrail alerts on unusual API call patterns.

**Metrics:** Zero long-lived IAM user keys in Vercel environment after wave completion.

---

#### W0-2: Implement API Rate Limiting & Plan Quota Enforcement

**Addresses:** SEC-02  
**Quality:** Security, Cost  
**Business value:** High — without enforcement, the Free plan's 3 contract/month limit is a billing comment, not a constraint. One abusive user can generate thousands of dollars in Bedrock costs.

**Current state:** Plan limits are documented in the pricing table but no enforcement mechanism exists in the API layer.

**Target state:** Middleware in every Vercel API Route checks the organization's current-month usage against the plan limit before processing the request.

**Implementation approach:**
1. Add a `usage` column or `usage_summary` table tracking `contracts_analyzed_this_month` per organization, updated atomically when a contract record is created.
2. In the `POST /api/contracts` route, query the counter before inserting — reject with HTTP 429 and a human-readable error if the limit is reached.
3. Add a secondary check in the Lambda worker to guard against race conditions (two concurrent uploads that both passed the API check before either incremented the counter). The Lambda should check and reject before invoking Bedrock.
4. Expose `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers on all routes.

**Metrics:** Zero unauthorized plan-limit overages in first 30 days.

---

#### W0-3: Add a Correlation ID Across the Full Pipeline

**Addresses:** OBS-01  
**Quality:** Observability  
**Business value:** High — without a correlation ID, debugging a failed analysis requires manually correlating Vercel logs, SQS message IDs, Lambda execution IDs, and Aurora records. Mean time to diagnose (MTTD) for production incidents will be hours, not minutes.

**Current state:** Each service layer logs independently. The SQS message contains `contractId` but Lambda logs do not consistently emit it as a structured field.

**Target state:** Every log line from Vercel → SQS → Lambda shares the same `correlationId` (the `contractId` UUID is the natural candidate).

**Implementation approach:**
1. In `POST /api/contracts`, include `contractId` in the SQS message body and as a structured log field.
2. In the Lambda handler, extract `contractId` from the SQS event and inject it into every log call: `console.log(JSON.stringify({ correlationId: contractId, event: 'analysis_started' }))`.
3. In Vercel API Routes, log `contractId` on every polling response.
4. Create a CloudWatch Logs Insights query template that filters by `correlationId` across Lambda and Aurora slow query log groups.

**Metrics:** Any failed analysis traceable from Vercel request to Lambda DLQ within 5 minutes using only the `contractId`.

---

#### W0-4: DLQ Depth Alerting

**Addresses:** OBS-02  
**Quality:** Observability, Availability  
**Business value:** High — without alerting, failed analyses accumulate in the DLQ while users see contracts stuck at `analyzing` forever. The first signal is a user support ticket.

**Current state:** SQS DLQ exists and captures failed jobs. No CloudWatch alarm or notification is configured.

**Target state:** A CloudWatch Alarm on `NumberOfMessagesSent` for the DLQ triggers a notification (email or Slack) when the first message lands.

**Implementation approach:**
1. Create a CloudWatch Alarm: metric `AWS/SQS > NumberOfMessagesSent`, threshold `>= 1`, period `60s`, evaluation periods `1`.
2. Attach an SNS topic → email or Slack webhook for notification.
3. Document a runbook for DLQ inspection: replay procedure using the AWS Console or a Lambda replay function.
4. Add a DLQ message count to the internal admin dashboard if one exists.

**Metrics:** Notification received within 2 minutes of the first DLQ message landing. MTTD for analysis failures < 5 minutes.

---

#### W0-5: Fix SQS Visibility Timeout vs. Lambda Duration Mismatch

**Addresses:** AVL-02  
**Quality:** Availability  
**Business value:** High — if a slow analysis (large contract, Bedrock latency spike) takes longer than the 5-minute visibility timeout, SQS re-queues the message. Lambda picks it up again and creates a duplicate analysis record for the same contract. The status transitions `pending → analyzing → done → analyzing → done` would confuse users and pollute the analyses table.

**Current state:** SQS visibility timeout = 5 minutes (per ARCHITECTURE.md). Lambda max timeout = 15 minutes.

**Target state:** Visibility timeout ≥ Lambda function timeout + 10% buffer.

**Implementation approach:**
1. Set SQS visibility timeout to `15 minutes + 2 minutes buffer = 17 minutes` (maximum allowed is 12 hours).
2. Alternatively, have the Lambda heartbeat the message visibility using `ChangeMessageVisibility` every 4 minutes during long analyses — this is the more robust solution as it extends visibility only for in-flight messages.
3. Add a Lambda-level idempotency check: before beginning analysis, verify the contract is still in `pending` or `analyzing` state (not already `done`) to guard against any duplicate execution.

**Metrics:** Zero duplicate analysis records in Aurora for any single contract.

---

#### W0-6: Define Health Check Endpoint

**Addresses:** AVL-03  
**Quality:** Availability  
**Business value:** Medium — uptime monitors, Vercel deployments, and future load balancers need a canonical signal.

**Target state:** `GET /api/health` returns `{ status: "ok", db: "ok", timestamp: "..." }` after verifying Aurora connectivity with a `SELECT 1`.

**Implementation approach:**
1. Add `GET /api/health` to the Vercel API Routes — protected only by internal auth or no auth (public endpoint returning minimal info).
2. The handler executes `SELECT 1` against Aurora via Drizzle ORM. If the query fails, return HTTP 503.
3. Configure an uptime monitor (Vercel's built-in, Better Uptime, or similar) to check this endpoint every 30 seconds.
4. Do not include sensitive information in the response (no version strings, no stack traces).

**Metrics:** Health endpoint responds within 500ms at the 99th percentile. Uptime monitor triggers within 2 minutes of Aurora becoming unreachable.

---

### Wave 1 — Quick Wins (Weeks 1–4)

> **Goal:** Improve developer productivity, reduce operational cost, and harden the system with changes that each take less than a day to implement.

---

#### W1-1: Application Error Tracking (Sentry)

**Addresses:** OBS-03  
**Quality:** Observability, Maintainability  

Integrate Sentry (or equivalent) in both the Vercel API layer and the Lambda worker. Key configuration:
- Source maps uploaded at build time so stack traces point to TypeScript source lines.
- `contractId` and `organizationId` attached to every error event as context tags.
- Separate Sentry DSN for Lambda vs. Vercel (different environments, different alert routing).
- Alert thresholds: >5 new unique errors/hour triggers a notification.

**Metrics:** Mean time to detect (MTTD) for new error types < 10 minutes.

---

#### W1-2: Add Pagination to List Endpoints

**Addresses:** PRF-02  
**Quality:** Performance, Scalability  

`GET /api/contracts` must support cursor-based pagination before any organization accumulates more than a few hundred contracts. Cursor pagination (using `created_at + id` as the cursor) is preferable over offset-based because it remains stable under concurrent inserts.

**Implementation:**
- Add `?limit=25&cursor=<base64_encoded_cursor>` query parameters.
- Validate with Zod. Default limit = 25, max = 100.
- Return `{ data: [...], nextCursor: "...", hasMore: bool }`.
- Apply the same pattern to `GET /api/contracts/:id/analyses` and `GET /api/findings/:id/comments`.

**Metrics:** P99 response time for `GET /api/contracts` stays under 200ms regardless of organization contract count.

---

#### W1-3: PDF Bundle Lazy Loading

**Addresses:** PRF-03  
**Quality:** Performance  

`pdfjs-dist` is ~3 MB of JavaScript that is only needed on the contract detail page (`/contracts/:id`). Loading it on the dashboard hurts Time to Interactive globally.

**Implementation:**
- Use React's `lazy()` + `Suspense` to dynamically import the `PDFViewer` component only when the detail route is active.
- Configure Vite's `build.rollupOptions.output.manualChunks` to isolate the pdfjs bundle into a dedicated chunk named `pdf-worker`.
- Serve the pdfjs worker file directly from the Vercel CDN (not `blob:` URLs) by setting `pdfjs.GlobalWorkerOptions.workerSrc` to the CDN path.

**Metrics:** Dashboard Lighthouse performance score ≥ 90. PDF viewer load time < 1.5s on a simulated 4G connection.

---

#### W1-4: AWS Cost Allocation Tags

**Addresses:** CST-03  
**Quality:** Cost  

Apply consistent resource tags across all AWS resources so spending can be attributed to features and environments.

**Tag schema:**

```
Project:     contractlens
Environment: production | staging | dev
Feature:     upload | analysis | chat | email | database
```

**Apply to:** S3 bucket, SQS queues (main + DLQ), Lambda function, Aurora cluster, SES configuration set, CloudWatch log groups.

**Metrics:** 100% of AWS monthly spend attributable to a feature tag within the Cost Explorer dashboard.

---

#### W1-5: Extracted Text Caching for Chat

**Addresses:** PRF-04  
**Quality:** Performance, Cost  

The `/api/chat` route currently (by design) injects the contract's full extracted text into the system prompt on every message. For a 30-page contract this is ~15,000 tokens re-sent to Bedrock on each turn.

**Implementation options (in order of effort):**
1. **Low effort:** Store the extracted text in an `analyses.extracted_text TEXT` column after the Lambda worker's pdf-parse step. The chat route reads this column instead of re-fetching the PDF from S3.
2. **Higher effort:** Use Bedrock's prompt caching feature (available for Claude 3.5 Sonnet) to mark the system prompt as cacheable — Bedrock charges ~10% of normal input token cost for cache hits.

Option 1 eliminates the S3 fetch. Option 2 reduces token billing for multi-turn conversations.

**Metrics:** Chat endpoint p50 time-to-first-token < 600ms (vs. 1s target today). Bedrock token cost per chat session reduced by >50% for conversations with >2 turns.

---

#### W1-6: CloudFront Distribution for PDF Delivery

**Addresses:** PRF-03 (partially), availability  
**Quality:** Performance, Availability  

Currently, the PDF viewer fetches the PDF directly from S3 (via the Lambda-read IAM path or a presigned GET URL). S3 transfer latency is regional; users outside `us-east-1` will experience slow PDF loads.

**Implementation:**
1. Create a CloudFront distribution in front of the S3 bucket.
2. Use CloudFront signed URLs (not S3 presigned URLs) for authenticated PDF access — shorter TTL, origin access control (OAC) keeps S3 private.
3. Enable CloudFront's automatic compression for PDFs (though PDFs are already compressed, enabling this costs nothing).
4. Set a cache TTL of 1 hour for PDFs (they are immutable after upload).

**Metrics:** PDF first-byte latency < 200ms for users in EU and APAC (vs. ~800ms with direct S3 in us-east-1).

---

### Wave 2 — High-Impact Investments (Months 1–3)

> **Goal:** Address systemic architectural gaps that will limit the system at scale. Each item requires a week or more of focused engineering effort.

---

#### W2-1: Replace 2-Second Polling with SSE Push

**Addresses:** CST-01, PRF-01 (partially)  
**Quality:** Cost, Performance, Scalability  
**Business value:** Very high — this is the single largest cost driver at scale.

**Current state:** Every browser tab with an active upload calls `GET /api/contracts/:id` every 2 seconds. At 100 concurrent uploads: 50 Aurora Data API requests/second. At 1,000: 500 req/s just for polling. Aurora Data API costs ~$0.35 per million requests — at 500 req/s that is ~$1,000/month before a single useful query.

**Target state:** When a Lambda worker finishes an analysis, it publishes a completion event. The browser receives a single push notification and stops polling.

**Implementation approach (lowest complexity on Vercel):**

1. **Vercel KV (Redis) as a pub/sub channel:** When Lambda completes an analysis, it calls a Vercel API Route (or publishes directly to a Vercel KV pub/sub channel) with `{ contractId, status: 'done' }`.
2. **SSE endpoint:** A new Vercel API Route `GET /api/contracts/:id/status-stream` holds an open SSE connection. It subscribes to the Vercel KV channel for the contract's key. When the message arrives, it emits `data: { status: "done" }` and closes the stream.
3. **Frontend:** Replace the React Query polling interval with `useEventSource` — the browser opens one SSE connection per active upload and handles the push.
4. **Fallback:** Keep a 30-second polling fallback in case the SSE connection drops.

**Alternative (API Gateway WebSocket):** More robust but adds a third compute boundary. Recommended for Wave 3 if SSE proves insufficient.

**Metrics:**
- Aurora Data API calls during analysis: reduced from O(N × analysis_duration_seconds / 2) to O(1) per analysis.
- Estimated cost reduction: >90% of current polling cost at 100+ concurrent uploads.

---

#### W2-2: Aurora Connection Pooling (RDS Proxy)

**Addresses:** PRF-01  
**Quality:** Performance, Scalability, Availability  

**Current state:** Aurora Data API is used by Lambda for simplicity — no VPC required. However, the Data API is stateless: each call is a new HTTP request to an AWS-managed endpoint that opens a short-lived connection to Aurora. Under Lambda concurrency spikes (>50 simultaneous workers), Aurora's max_connections limit (tied to ACU) can be saturated.

**Target state:** Add RDS Proxy between Lambda and Aurora. RDS Proxy maintains a warm connection pool and multiplexes Lambda connections onto a smaller set of Aurora connections.

**Implementation approach:**
1. Create an RDS Proxy for the Aurora cluster, pointing at the writer endpoint.
2. Move the Lambda function into a VPC with private subnets (required for RDS Proxy connectivity). Add a VPC endpoint for SQS and Bedrock to keep traffic off the public internet.
3. Update the Lambda `DATABASE_URL` to point at the RDS Proxy endpoint.
4. Update Vercel's `DATABASE_URL` — Vercel API Routes can reach RDS Proxy only via a NAT gateway or by keeping the Data API for Vercel and RDS Proxy only for Lambda.

**Note:** If moving Lambda to a VPC is too disruptive in Wave 2, an interim mitigation is to use Aurora Data API but limit Lambda concurrency to 30 reserved concurrent executions (below Aurora's connection limit at 1 ACU).

**Metrics:** Aurora `DatabaseConnections` CloudWatch metric stays below 80% of max capacity under 100 concurrent Lambda invocations.

---

#### W2-3: Distributed Tracing with AWS X-Ray

**Addresses:** OBS-04  
**Quality:** Observability  

Add AWS X-Ray instrumentation to create a unified trace from the Vercel API Route → SQS → Lambda → Bedrock → Aurora for every analysis request.

**Implementation:**
1. Instrument the Lambda handler with the X-Ray SDK: wrap the SQS handler, S3 GetObject call, Bedrock InvokeModel call, and Aurora write in X-Ray subsegments.
2. Add the `X-Amzn-Trace-Id` header to the SQS message attributes so the trace is linked across the queue boundary.
3. Enable X-Ray active tracing on the Lambda function and SQS queue.
4. For Vercel, emit trace context in API Route logs using the OpenTelemetry format — Vercel does not natively support X-Ray on the frontend, but the correlation ID (W0-3) bridges the two trace contexts.
5. Create X-Ray Service Map and latency percentile alerts in CloudWatch.

**Metrics:** Full analysis pipeline visible as a single X-Ray trace. P95 Lambda analysis duration monitored with a CloudWatch alarm at >45s.

---

#### W2-4: Automated Testing Strategy

**Addresses:** MNT-01  
**Quality:** Maintainability  

No test coverage is the most compounding technical debt for a product with this complexity. The priority order is:

**Layer 1 — Lambda unit tests (highest value, lowest effort):**
- Test the analysis prompt construction and Zod schema validation against mock Bedrock responses.
- Test the Aurora write transaction with an in-memory PostgreSQL instance (pg-mem or a real Postgres in Docker).
- Target: >80% coverage of the Lambda handler's business logic branches (happy path, error path, chunking path).

**Layer 2 — Vercel API Route integration tests:**
- Use Vitest + supertest against a test Aurora database.
- Cover: contract creation, plan limit enforcement (W0-2), tenant isolation (verify organization A cannot read organization B's contracts).
- Target: All `organization_id` scoping paths have a test that asserts the isolation property.

**Layer 3 — End-to-end tests (highest business value, highest effort):**
- Use Playwright to test the full upload → analysis → display flow against a staging environment with a pre-seeded contract.
- Run on every pull request via GitHub Actions.
- Target: The critical path (upload → done → findings visible) has a green E2E test as a merge gate.

**Metrics:** Pull request merge gate requires green unit tests + integration tests. Zero regressions in the multi-tenant isolation property.

---

#### W2-5: raw_llm_output Archival Strategy

**Addresses:** CST-02  
**Quality:** Cost, Maintainability  

`analyses.raw_llm_output JSONB` stores the full Bedrock response for every analysis version. A 30-page contract analysis response is ~20–50 KB of JSON. At the base-case scenario (200 Starter users × 20 contracts/month = 4,000 analyses/month), that is 200 MB/month of JSONB growth — manageable. At pro scale (500 users × unlimited), this compounds fast.

**Implementation:**
1. After a successful analysis write, move `raw_llm_output` to S3 under `llm-outputs/{analysisId}.json.gz` (gzip reduces size by ~70%).
2. Store only the S3 key in the `analyses` table: `raw_llm_output_s3_key TEXT`.
3. Expose the raw output only in a staff/debug API route that pre-signs a temporary read URL.
4. Apply an S3 lifecycle rule: move `llm-outputs/` to S3 Glacier Instant Retrieval after 90 days.

**Metrics:** Aurora storage growth rate reduced by >60%. S3 Glacier cost for raw outputs < $2/month at base-case volume.

---

#### W2-6: Zero-Downtime Database Migration Pattern

**Addresses:** MNT-03  
**Quality:** Maintainability, Availability  

The current migration strategy ("a dedicated script") is sufficient for a hackathon but not for a live product where users may be mid-session during a deploy.

**Pattern:** Expand-and-contract migrations.

1. **Expand:** Add the new column as nullable. Deploy application code that writes to both old and new columns.
2. **Backfill:** Run a background job to populate the new column for existing rows.
3. **Contract:** Once all rows are populated, drop the old column in a subsequent deployment.

**Tooling:**
- Drizzle Kit's `drizzle-kit push` is fine for development but not for production. Switch to explicit migration files checked into the repository.
- Add a `migrations/` directory with timestamped SQL files generated by `drizzle-kit generate`.
- Run migrations in a pre-deploy step (Vercel's build hook) before the new code goes live.

**Metrics:** Zero downtime during schema migrations. Migration execution time < 30 seconds for any single migration (enforced by testing against a production-size data clone).

---

### Wave 3 — Scale & Enterprise (Months 3–6)

> **Goal:** Unlock enterprise customer segments (SSO, data residency, compliance) and handle 10× current projected load without architectural changes.

---

#### W3-1: Step Functions for the Analysis Pipeline

**Addresses:** MNT (monolithic Lambda), Scalability, Availability  
**Quality:** Maintainability, Scalability  

**Current state:** The Lambda analysis worker is a monolithic 10-step function. As the pipeline adds OCR (Textract), multi-language support, and fine-tuned model variants, this function will become unmaintainable.

**Target state:** Replace the single Lambda with an AWS Step Functions Express Workflow with discrete states:

```
START
  → ValidateContract      (check org limits, verify S3 key exists)
  → DetectContentType     (text-based PDF vs. scanned image)
  → [Branch]
      → ExtractText       (pdf-parse — text PDFs)
      → OCRExtractText    (AWS Textract — scanned PDFs)
  → ChunkIfLarge          (split contracts > 100k tokens)
  → InvokeModel           (Bedrock Claude — with retry/catch)
  → WriteResults          (Aurora transaction)
  → SendNotifications     (SES + future Slack/webhook)
END
```

**Benefits:**
- Each state is an independent Lambda with a single responsibility.
- Step Functions handles retry logic, error states, and branching natively.
- Visual execution history in the AWS console for debugging.
- Future states (fine-tune model, multi-language, clause library matching) are additive changes, not rewrites.

**Metrics:** Pipeline is modifiable (new state added, existing state changed) without touching unrelated logic. Mean time to add a new pipeline stage < 1 day.

---

#### W3-2: Aurora Read Replica for Dashboard & Search

**Addresses:** Scalability, Performance  
**Quality:** Scalability, Performance  

Dashboard queries (stats aggregations, full-text search, contract listing) and write operations (analysis results) currently share the same Aurora writer endpoint. Under load, reporting queries compete with the critical analysis write path.

**Implementation:**
1. Add an Aurora read replica (Aurora Serverless v2 supports reader instances that scale independently).
2. Route `GET /api/dashboard/*`, `GET /api/contracts`, and `GET /api/key-dates` to the reader endpoint.
3. Route all write operations and `GET /api/contracts/:id` (polling, which requires read-your-writes consistency) to the writer endpoint.
4. In Drizzle ORM, configure a `readClient` and `writeClient` via Drizzle's built-in read replica support.

**Metrics:** Writer instance CPU stays below 60% during dashboard load tests at 10× base-case query volume. Analysis write P99 latency unaffected by dashboard queries.

---

#### W3-3: Multi-Region Data Residency for EU Enterprise Customers

**Addresses:** Portability, Security  
**Quality:** Portability, Security  

Enterprise customers in the EU increasingly require GDPR-compliant data residency: contract PDFs and analysis results must reside in the EU.

**Implementation:**
1. Deploy a second Aurora Global Database secondary in `eu-west-1` (Ireland).
2. Create a second S3 bucket in `eu-west-1` with the same private configuration.
3. Add an `organization.preferred_region TEXT` column (`us-east-1` | `eu-west-1`).
4. Route Vercel API Routes and Lambda workers to the appropriate region based on the authenticated organization's preferred region.
5. Update Bedrock model selection — confirm Claude 3.5 Sonnet is available in `eu-west-1`; fall back to `eu-central-1` (Frankfurt) if not.

**Metrics:** EU-resident organizations' contract data never leaves the AWS EU region. Achieves GDPR Art. 46 adequacy for enterprise prospects.

---

## 5. Wave Dependency Map

Dependencies between improvements (an arrow means "should be completed before"):

```
W0-3 (Correlation ID)
  └─→ W2-3 (X-Ray Tracing)

W0-2 (Rate Limiting)
  └─→ no downstream (standalone safeguard)

W0-5 (Visibility Timeout Fix)
  └─→ W2-1 (SSE Push) [SSE eliminates the polling root cause]

W1-5 (Extracted Text Caching)
  └─→ W2-1 (SSE Push) [both reduce unnecessary calls; text cache reduces Bedrock cost per chat session]

W0-6 (Health Check)
  └─→ W3-2 (Read Replica) [health check should verify both writer and reader endpoints]

W2-2 (RDS Proxy)
  └─→ W3-1 (Step Functions) [Lambda-in-VPC is required for both]

W2-4 (Testing)
  └─→ W2-6 (Zero-Downtime Migrations) [migration tests run in the same test infrastructure]

W2-5 (LLM Output Archival)
  └─→ W3-3 (Multi-Region) [archival bucket lifecycle rules must be replicated per region]

W3-1 (Step Functions)
  └─→ W3-3 (Multi-Region) [per-region Lambda deployment is simpler with discrete state functions]
```

**Critical path to production readiness:**

```
W0-1 → W0-2 → W0-3 → W0-4 → W0-5 → W0-6
  ↓
W1-1 → W1-2 → W2-4 → W2-6
  ↓
W2-1 → W2-2 → W3-1 → W3-2 → W3-3
```

All Wave 0 items are parallel-executable. Within Wave 1, W1-1 through W1-4 are independent.

---

## 6. Follow-Up Metrics

After each wave, the following metrics define success. Baselines should be measured before Wave 0 begins.

### Availability Metrics

| Metric | Baseline | Wave 0 Target | Wave 2 Target |
|---|---|---|---|
| Uptime (health endpoint) | Unmeasured | 99.5% | 99.9% |
| Analysis pipeline success rate | Unmeasured | > 95% | > 99% |
| Mean time to detect (MTTD) analysis failure | Hours | < 5 min | < 2 min |
| DLQ message age at detection | Unknown | < 2 min | < 1 min |

### Cost Metrics

| Metric | Baseline | Wave 1 Target | Wave 2 Target |
|---|---|---|---|
| Aurora Data API calls / analysis | ~15 (2s × 30s) | ~15 (unchanged) | ~1 (SSE) |
| Bedrock input tokens / chat turn | Full contract (~15k) | ~7.5k (cache hit) | ~7.5k |
| S3 storage growth / month | Unmeasured | Tracked | < $5/month with archival |
| AWS spend attributable by feature | 0% | 100% | 100% |

### Performance Metrics

| Metric | NFR Target | Wave 0 Baseline | Wave 2 Target |
|---|---|---|---|
| Dashboard load (p95) | < 1.5s | Measured after W0-6 | < 1.2s |
| Analysis end-to-end (p95) | < 30s | Measured after W0-3 | < 25s |
| Chat first token (p95) | < 1s | Measured after W1-5 | < 600ms |
| PDF viewer interactive (p95) | Not specified | Measured after W1-3 | < 2s |

### Security Metrics

| Metric | Current | Wave 0 Target | Wave 2 Target |
|---|---|---|---|
| Long-lived IAM keys in Vercel env | 2 (KEY_ID + SECRET) | 0 | 0 |
| Plan limit bypass rate | Unmeasured | 0% | 0% |
| Mean key rotation frequency | Never (static) | N/A (OIDC — no keys) | N/A |

### Maintainability Metrics

| Metric | Current | Wave 2 Target |
|---|---|---|
| Lambda handler unit test coverage | 0% | > 80% |
| Multi-tenant isolation test cases | 0 | ≥ 5 per table with org scoping |
| E2E critical path coverage | 0 | Upload → analysis → findings display: 100% |
| Deployment-induced downtime | Unknown | 0s (expand-and-contract) |

---

## 7. Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| **OIDC configuration complexity delays W0-1** | Medium | High | Use a time-boxed interim: rotate IAM user keys + CloudTrail anomaly alert while OIDC is configured in parallel |
| **SSE push (W2-1) introduces complexity that breaks the fallback** | Medium | Medium | Keep the polling fallback active for 30 days post-deployment; disable only after SSE stability is confirmed in production |
| **RDS Proxy adds VPC latency to Lambda cold starts** | Low | Medium | Measure Lambda cold start before/after VPC migration; use provisioned concurrency for the analysis Lambda if cold starts exceed 2s |
| **Step Functions state machine becomes over-engineered for MVP** | Medium | Low | Gate W3-1 on actual pipeline complexity hitting the monolithic Lambda; if the pipeline stays at 10 steps with no branching, delay until OCR is required |
| **Aurora read replica replication lag causes stale polling responses** | Low | Medium | Route `GET /api/contracts/:id` (polling) exclusively to the writer endpoint; only dashboard/list queries use the reader (W3-2) |
| **Multi-region Bedrock availability** | Low | High | Before committing to W3-3, verify Claude 3.5 Sonnet model access is approved in `eu-west-1` or `eu-central-1`; have a fallback to cross-region inference if needed |
| **Lean team cannot absorb Wave 2 scope in parallel with product development** | High | Medium | Treat W2-1 (SSE push) and W2-4 (testing) as fixed scope; deprioritize W2-5 and W2-6 until the team has capacity |

---

## 8. Conclusion

ContractLens is built on a sound architectural foundation. The SQS-decoupled async pipeline, IAM-native Bedrock access, and Aurora ACID transactions reflect production-grade thinking at the design stage. The improvements in this plan are refinements of a working architecture, not replacements of it.

The highest-leverage actions are concentrated in Wave 0 and Wave 2:

- **Wave 0** eliminates the two highest-severity risks before any user data touches the system: long-lived IAM credential exposure and uncontrolled plan quota bypass.
- **Wave 2** addresses the structural cost and scalability ceiling: the polling-driven Aurora call volume that scales linearly with user growth, and the absence of any automated test coverage.

The phased structure ensures that each wave produces independently useful, deployable improvements. No wave requires a rewrite or forces the team to abandon current patterns — the Step Functions refactor (W3-1) is an evolution of the existing Lambda, not a replacement of the overall event-driven design.

Executing Waves 0 and 1 within the first 30 days post-launch puts ContractLens in a defensible production position. Waves 2 and 3 convert it into a system capable of serving enterprise customers with regulatory compliance requirements and 10× the initial projected user volume.

---

*ContractLens IMPROVEMENT_PLAN v1.0.0 — H01 Hackathon (h01.devpost.com) — June 2026*
