# ContractLens — Product Specification

> **Version:** 1.1.0
> **Status:** Draft
> **Hackathon:** H01 (h01.devpost.com) — June 2026
> **Track:** Monetizable B2B App

---

## Table of Contents

1. [Overview](#1-overview)
2. [Business Value](#2-business-value)
3. [Functional Requirements](#3-functional-requirements)
4. [Non-Functional Requirements](#4-non-functional-requirements)
5. [Tech Stack](#5-tech-stack)
6. [Architecture](#6-architecture)
7. [Data Model](#7-data-model)
8. [API Contract](#8-api-contract)
9. [Future Features](#9-future-features)
10. [Changelog](#10-changelog)
11. [Contributing](#11-contributing)
12. [License](#12-license)
13. [Conclusion](#13-conclusion)

---

## 1. Overview

**ContractLens** is a SaaS web application that enables small and medium-sized businesses to upload contracts in PDF format and receive an AI-powered analysis in seconds. The system identifies risky clauses, benchmarks them against industry standards, generates an executive summary, and provides actionable negotiation recommendations — without requiring in-house legal counsel.

### Problem Statement

78% of SMBs sign contracts without proper legal review, exposing themselves to abusive clauses, hidden penalties, and unreasonable commitments. Hiring an external lawyer costs $200–$500 per contract and takes days. The gap between "sign now" pressure and "understand fully" need is where ContractLens operates.

### Solution

An instant, accessible, and affordable contract analysis tool that gives any business the clarity of a legal review in under 30 seconds.

---

## 2. Business Value

### Target Users

| Segment | Pain Point | Value Delivered |
|---|---|---|
| SMBs without in-house legal | Sign contracts without fully understanding them | Risk analysis in seconds |
| Entrepreneurs / Freelancers | Cannot afford legal advisory fees | Affordable basic review |
| Procurement / Sales teams | Complex vendor contracts | Benchmarking against industry standards |
| Growing startups | Many contracts, little time | Centralization and full traceability |

### Monetization

| Feature | Free | Starter ($29/mo) | Pro ($79/mo) |
|---|---|---|---|
| Contracts / month | 3 | 20 | Unlimited |
| Versions per contract | 1 | 3 | Unlimited |
| Users | 1 | 5 | 15 |
| Chat with contract | ❌ | ✅ | ✅ |
| Negotiation mode | ❌ | ❌ | ✅ |
| Export PDF report | ❌ | ✅ | ✅ |
| Email alerts | ❌ | ✅ | ✅ |
| Support | — | Email | Priority |

### Revenue Projections (6 months post-launch)

| Scenario | Starter Users | Pro Users | MRR |
|---|---|---|---|
| Conservative | 50 | 10 | $2,240 |
| Base | 200 | 50 | $9,750 |
| Optimistic | 500 | 150 | $26,350 |

### ROI for Users

A single avoided bad clause (e.g., uncapped liability, missed auto-renewal) can save an SMB $10,000–$50,000. At $29/month, the payback period is effectively one avoided incident per year.

---

## 3. Functional Requirements

### FR-01 — Contract Upload

| ID | Requirement | Priority |
|---|---|---|
| FR-01.1 | Users can upload PDF contracts via drag & drop or file picker | Must Have |
| FR-01.2 | Upload goes directly from browser to S3 via presigned URL (no Vercel proxy) | Must Have |
| FR-01.3 | Maximum file size: 50 MB | Must Have |
| FR-01.4 | Only PDF format is accepted; other formats are rejected with a clear error | Must Have |
| FR-01.5 | Users can optionally provide: contract name, counterparty name, contract type, tags | Should Have |
| FR-01.6 | Upload progress is shown with a progress bar and status messages | Should Have |

### FR-02 — AI Analysis

| ID | Requirement | Priority |
|---|---|---|
| FR-02.1 | System extracts text from uploaded PDF server-side (Lambda) | Must Have |
| FR-02.2 | System sends extracted text to Anthropic Claude 3.5 Sonnet via Amazon Bedrock with a structured prompt (tool use) | Must Have |
| FR-02.3 | Analysis returns: risk score (0–100), risk level, executive summary, findings, key dates, negotiation points | Must Have |
| FR-02.4 | Analysis runs asynchronously; frontend polls for completion | Must Have |
| FR-02.5 | Contract status transitions: `pending → analyzing → done` (or `error`) | Must Have |
| FR-02.6 | On failure, contract is marked `error` and user is notified | Must Have |
| FR-02.7 | Long contracts (>100k tokens) are chunked by section before LLM submission | Should Have |
| FR-02.8 | Users can trigger a re-analysis to create a new version | Should Have |

### FR-03 — Analysis Display

| ID | Requirement | Priority |
|---|---|---|
| FR-03.1 | Risk Score is displayed as an animated gauge (0–100) with color coding | Must Have |
| FR-03.2 | Executive summary is shown in plain language | Must Have |
| FR-03.3 | Findings are listed by category with: excerpt, explanation, risk level, suggestion | Must Have |
| FR-03.4 | Clicking a finding scrolls the PDF viewer to the exact clause and highlights it | Must Have |
| FR-03.5 | Each finding can be marked as "reviewed" or "accepted" | Should Have |
| FR-03.6 | Analysis report can be exported as a PDF | Should Have |

### FR-04 — Standards Comparison

| ID | Requirement | Priority |
|---|---|---|
| FR-04.1 | Each finding is tagged with a market benchmark badge: "Market Standard" / "Uncommon" / "Unusually Restrictive" | Must Have |
| FR-04.2 | Benchmark data covers: NDA, SaaS, Vendor, Employment, Lease contract types | Should Have |

### FR-05 — Chat with Contract

| ID | Requirement | Priority |
|---|---|---|
| FR-05.1 | Users can ask plain-language questions about the contract | Must Have |
| FR-05.2 | Responses stream in real time via Vercel AI SDK + `@ai-sdk/amazon-bedrock` | Must Have |
| FR-05.3 | Responses cite the relevant clause from the contract | Should Have |
| FR-05.4 | Chat is available on Starter and Pro plans only | Must Have |

### FR-06 — Version History

| ID | Requirement | Priority |
|---|---|---|
| FR-06.1 | Multiple analysis versions can be saved per contract | Should Have |
| FR-06.2 | Users can compare changes between versions with a visual diff | Should Have |
| FR-06.3 | Each version supports notes and comments | Could Have |

### FR-07 — Key Dates & Alerts

| ID | Requirement | Priority |
|---|---|---|
| FR-07.1 | Key dates are automatically extracted from the contract (expiration, renewal, payment milestones) | Must Have |
| FR-07.2 | Email notifications are sent at configurable intervals: 30 / 15 / 7 days before each date | Should Have |
| FR-07.3 | A calendar view shows all active milestones across contracts | Should Have |
| FR-07.4 | Contracts with auto-renewal within 30 days are badged prominently | Should Have |

### FR-08 — Dashboard

| ID | Requirement | Priority |
|---|---|---|
| FR-08.1 | Dashboard shows: total contracts, high-risk count, expiring in 30 days, analyses this month | Must Have |
| FR-08.2 | Contracts table with columns: Name, Counterparty, Type, Risk, Upload Date, Actions | Must Have |
| FR-08.3 | Filters by: risk level, contract type, date range | Should Have |
| FR-08.4 | Full-text search by contract name, counterparty, or clause content | Should Have |

### FR-09 — Teams & Collaboration

| ID | Requirement | Priority |
|---|---|---|
| FR-09.1 | Each organization has an isolated workspace | Must Have |
| FR-09.2 | Roles: Admin, Editor, Viewer with appropriate permission scopes | Must Have |
| FR-09.3 | Admins can invite members by email | Should Have |
| FR-09.4 | Users can comment on findings with @mentions | Should Have |
| FR-09.5 | Activity log is maintained per contract | Could Have |

### FR-10 — Authentication & Multi-tenancy

| ID | Requirement | Priority |
|---|---|---|
| FR-10.1 | Authentication via Clerk (email/password + OAuth) | Must Have |
| FR-10.2 | Every data query is scoped to the user's organization | Must Have |
| FR-10.3 | New users are onboarded into a new organization by default | Must Have |
| FR-10.4 | Users can belong to only one organization (MVP) | Must Have |

### FR-11 — Negotiation Mode

| ID | Requirement | Priority |
|---|---|---|
| FR-11.1 | For high-risk findings, the system suggests alternative clause wording | Should Have |
| FR-11.2 | Negotiation mode is available on Pro plan only | Must Have |

---

## 4. Non-Functional Requirements

### NFR-01 — Performance

| ID | Requirement | Target |
|---|---|---|
| NFR-01.1 | PDF upload (presigned URL request + S3 PUT) | < 3s for files up to 10 MB |
| NFR-01.2 | Contract analysis end-to-end (SQS → Lambda → Aurora) | < 30s for contracts up to 30 pages |
| NFR-01.3 | Dashboard initial load (API + render) | < 1.5s |
| NFR-01.4 | Polling interval for analysis status | Every 2s |
| NFR-01.5 | Chat first token latency (Vercel AI SDK + Bedrock streaming) | < 1s |

### NFR-02 — Scalability

| ID | Requirement |
|---|---|
| NFR-02.1 | Aurora PostgreSQL Serverless v2 auto-scales from 0.5 ACU to 16 ACU based on load |
| NFR-02.2 | Lambda concurrency scales automatically with SQS queue depth |
| NFR-02.3 | S3 provides effectively unlimited storage for PDF files |
| NFR-02.4 | Vercel Edge Network handles frontend CDN distribution globally |

### NFR-03 — Reliability

| ID | Requirement |
|---|---|
| NFR-03.1 | SQS dead-letter queue captures failed analysis jobs after 3 retries |
| NFR-03.2 | Lambda analysis writes to Aurora in a single ACID transaction; partial writes roll back |
| NFR-03.3 | Contract status is always consistent: a contract is never `done` without a complete analysis |
| NFR-03.4 | Vercel API Routes are stateless and horizontally scalable |

### NFR-04 — Security

| ID | Requirement |
|---|---|
| NFR-04.1 | All API routes validate a Clerk JWT on every request |
| NFR-04.2 | Every database query filters by `organization_id` to enforce tenant isolation |
| NFR-04.3 | S3 presigned URLs expire after 5 minutes |
| NFR-04.4 | S3 bucket is private; no public access |
| NFR-04.5 | Lambda IAM role follows least-privilege: S3 read, SQS consume, Aurora write, SES send, Bedrock InvokeModel |
| NFR-04.6 | All secrets (DB credentials) are stored in environment variables; Bedrock access uses Lambda IAM role — no API key needed |
| NFR-04.7 | HTTPS enforced on all endpoints (Vercel + AWS API Gateway) |

### NFR-05 — Observability

| ID | Requirement |
|---|---|
| NFR-05.1 | Vercel built-in logs and error tracking for API Routes |
| NFR-05.2 | Lambda structured logs to CloudWatch with contract ID and duration |
| NFR-05.3 | Aurora slow query log enabled |
| NFR-05.4 | SQS CloudWatch metrics: queue depth, age of oldest message, DLQ count |

### NFR-06 — Accessibility

| ID | Requirement |
|---|---|
| NFR-06.1 | UI meets WCAG 2.1 AA contrast ratios |
| NFR-06.2 | All interactive elements are keyboard navigable |
| NFR-06.3 | Risk level is communicated via both color and text label (not color alone) |

### NFR-07 — Legal & Compliance

| ID | Requirement |
|---|---|
| NFR-07.1 | Legal disclaimer displayed on every analysis result |
| NFR-07.2 | Contract PDFs are stored in S3 with server-side encryption (SSE-S3) |
| NFR-07.3 | Users can delete their contracts and all associated data |

---

## 5. Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Frontend** | React + Vite + TypeScript | Fast DX, strong typing, Vercel-native |
| **UI Components** | shadcn/ui + Tailwind CSS | Accessible, unstyled primitives with full design control |
| **PDF Viewer** | react-pdf (pdfjs-dist) | In-browser rendering with programmatic scroll + highlight |
| **Backend (sync)** | Vercel API Routes (Node.js 22) | Auth, CRUD, upload trigger, polling — short-lived operations |
| **Backend (async)** | AWS Lambda (Node.js 22) | LLM analysis pipeline — up to 15-min timeout, auto-scaling |
| **Job Queue** | AWS SQS | Durable async decoupling; built-in retry + DLQ |
| **Database** | AWS Aurora PostgreSQL (Serverless v2) | Relational integrity, JSONB, full-text search, auto-scaling |
| **ORM** | Drizzle ORM | Type-safe, lightweight, Edge-compatible |
| **Validation** | Zod v4 | Runtime schema validation on all API boundaries |
| **File Storage** | AWS S3 + presigned URLs | Direct browser upload, no Vercel body size limits |
| **AI / LLM** | Anthropic Claude 3.5 Sonnet via Amazon Bedrock | Fully AWS-native; IAM role auth, no external API key; tool use for structured JSON output |
| **PDF Parsing** | pdf-parse (Lambda) | Server-side text extraction |
| **Authentication** | Clerk | Multi-tenant org support, JWT middleware, OAuth |
| **Email** | AWS SES | Same AWS account — no extra vendor or credentials |
| **Real-time / Streaming** | Vercel AI SDK + `@ai-sdk/amazon-bedrock` | Native Claude streaming via Bedrock, works within Vercel's function model |
| **API Codegen** | Orval (OpenAPI → React Query) | Type-safe client hooks generated from spec |
| **Frontend Deploy** | Vercel | Required by hackathon; CDN, preview deployments, CI/CD |

---

## 6. Architecture

### System Diagram

```mermaid
graph TD
    subgraph Browser["Browser (Vercel CDN)"]
        UI["React + Vite\nshadcn/ui + Tailwind"]
        PDFViewer["react-pdf\nPDF Viewer"]
        ReactQuery["React Query\n(Orval generated)"]
    end

    subgraph Vercel["Vercel (API Routes — Node.js 22)"]
        APIRoutes["API Routes\n/api/*"]
        AIStream["AI SDK\nStreaming endpoint\n/api/chat"]
        Clerk["Clerk\nAuth middleware"]
    end

    subgraph AWS["AWS"]
        S3["S3\nPDF File Storage"]
        SQS["SQS\nAnalysis Job Queue"]

        subgraph Lambda["Lambda (Node.js 22)"]
            LambdaFn["Analysis Worker\npdf-parse → Bedrock Claude\n→ Aurora write"]
        end

        Aurora["Aurora PostgreSQL\norganizations · users\ncontracts · analyses\nfindings · key_dates\ncomments"]
        SES["SES\nEmail Alerts"]
        Bedrock["Amazon Bedrock\nClaude 3.5 Sonnet"]
    end

    UI -->|"1 · Request presigned URL"| APIRoutes
    APIRoutes -->|"2 · Return presigned PUT URL"| UI
    UI -->|"3 · PUT PDF directly"| S3
    UI -->|"4 · POST /api/contracts"| APIRoutes
    APIRoutes -->|"5 · INSERT contract (pending)"| Aurora
    APIRoutes -->|"6 · SQS.sendMessage"| SQS

    SQS -->|"7 · Trigger"| LambdaFn
    LambdaFn -->|"8 · GetObject"| S3
    LambdaFn -->|"9 · pdf-parse"| LambdaFn
    LambdaFn -->|"10 · InvokeModel (tool use)"| Bedrock
    Bedrock -->|"11 · findings JSON"| LambdaFn
    LambdaFn -->|"12 · INSERT analyses + findings\nUPDATE status=done"| Aurora
    LambdaFn -->|"13 · SendEmail"| SES

    ReactQuery -->|"14 · GET /api/contracts/:id (poll)"| APIRoutes
    APIRoutes -->|"15 · SELECT + JOIN"| Aurora
    Aurora -->|"16 · Full payload"| APIRoutes
    APIRoutes -->|"17 · JSON response"| ReactQuery
    ReactQuery --> UI
    ReactQuery --> PDFViewer

    UI -->|"Chat question"| AIStream
    AIStream -->|"InvokeModelWithResponseStream"| Bedrock
    Bedrock -->|"Token stream"| AIStream
    AIStream -->|"SSE stream"| UI

    Clerk -.->|"JWT validation"| APIRoutes

    classDef aws fill:#FF9900,color:#000,stroke:#c47400
    classDef vercel fill:#000,color:#fff,stroke:#333
    classDef browser fill:#1a73e8,color:#fff,stroke:#1557b0

    class S3,SQS,LambdaFn,Aurora,SES,Bedrock aws
    class APIRoutes,AIStream,Clerk vercel
    class UI,PDFViewer,ReactQuery browser
```

### Key Architecture Decisions

| Decision | Rationale |
|---|---|
| **S3 presigned URLs** | Bypasses Vercel's 4.5 MB body limit; PDFs go directly browser → S3 |
| **SQS + Lambda** | LLM calls take 15–30s; Lambda supports up to 15 min vs Vercel's 60s max |
| **Polling over WebSocket** | Simpler on Vercel; 2s polling is acceptable for a 30s analysis |
| **Amazon Bedrock for LLM** | Claude 3.5 Sonnet accessed via Bedrock uses IAM role auth — no external API key, stays fully within AWS |
| **Vercel AI SDK** | Native streaming with `@ai-sdk/amazon-bedrock` provider; works within Vercel's serverless model |
| **Drizzle ORM** | Edge-compatible, type-safe, works in both Vercel and Lambda |

---

## 7. Data Model

```sql
-- Organizations (multi-tenant workspace)
organizations
  id              UUID PK
  name            TEXT NOT NULL
  plan            TEXT DEFAULT 'free'        -- free | starter | pro
  created_at      TIMESTAMPTZ

-- Users
users
  id              UUID PK
  organization_id UUID FK → organizations
  email           TEXT UNIQUE NOT NULL
  name            TEXT
  role            TEXT                       -- admin | editor | viewer
  created_at      TIMESTAMPTZ

-- Contracts
contracts
  id              UUID PK
  organization_id UUID FK → organizations
  uploaded_by     UUID FK → users
  name            TEXT NOT NULL
  counterparty    TEXT
  contract_type   TEXT                       -- NDA | SaaS | Vendor | Employment | Other
  file_url        TEXT                       -- S3 object URL
  file_name       TEXT
  file_size_bytes INT
  status          TEXT DEFAULT 'pending'     -- pending | analyzing | done | error
  created_at      TIMESTAMPTZ
  updated_at      TIMESTAMPTZ

-- Analyses (LLM output, versioned)
analyses
  id              UUID PK
  contract_id     UUID FK → contracts
  version         INT DEFAULT 1
  risk_score      INT                        -- 0–100
  risk_level      TEXT                       -- low | medium | high | critical
  summary         TEXT
  raw_llm_output  JSONB                      -- full LLM response
  model_used      TEXT
  tokens_used     INT
  duration_ms     INT
  created_at      TIMESTAMPTZ

-- Findings (individual clause findings)
findings
  id              UUID PK
  analysis_id     UUID FK → analyses
  category        TEXT   -- liability | payment | termination | exclusivity | jurisdiction | renewal | other
  title           TEXT
  excerpt         TEXT                       -- literal contract fragment
  explanation     TEXT                       -- plain-language explanation
  risk_level      TEXT                       -- low | medium | high | critical
  suggestion      TEXT
  page_number     INT
  char_offset     INT                        -- text position for PDF highlight
  is_standard     BOOL
  status          TEXT DEFAULT 'open'        -- open | reviewed | accepted
  created_at      TIMESTAMPTZ

-- Key dates (auto-extracted)
key_dates
  id              UUID PK
  contract_id     UUID FK → contracts
  label           TEXT                       -- "Expiration date", "Auto-renewal", etc.
  date            DATE
  notified_30d    BOOL DEFAULT false
  notified_7d     BOOL DEFAULT false
  created_at      TIMESTAMPTZ

-- Comments (team collaboration)
comments
  id              UUID PK
  finding_id      UUID FK → findings
  user_id         UUID FK → users
  body            TEXT
  created_at      TIMESTAMPTZ
```

---

## 8. API Contract

```yaml
# Upload & Contracts
POST   /api/contracts                        # Create contract record + return presigned S3 URL
GET    /api/contracts                        # List organization's contracts
GET    /api/contracts/:id                    # Get contract + latest analysis (polling endpoint)
DELETE /api/contracts/:id                    # Delete contract and all associated data

# Analysis
POST   /api/contracts/:id/analyze            # Re-analyze (enqueues new SQS job)
GET    /api/contracts/:id/analyses           # List all analysis versions

# Findings
PATCH  /api/findings/:id                     # Update finding status (reviewed / accepted)
POST   /api/findings/:id/comments            # Add comment to a finding
GET    /api/findings/:id/comments            # List comments on a finding

# Chat (streaming)
POST   /api/chat                             # Stream LLM response for a contract question

# Key Dates
GET    /api/key-dates                        # All key dates for the organization
GET    /api/key-dates/upcoming               # Upcoming within the next N days

# Dashboard
GET    /api/dashboard/stats                  # Aggregate stats (total, high-risk, expiring)
GET    /api/dashboard/risk-distribution      # Risk level distribution for chart

# Organization & Team
GET    /api/organizations/me                 # Current organization
PATCH  /api/organizations/me                 # Update organization settings
GET    /api/organizations/me/members         # List members
POST   /api/organizations/me/members         # Invite member by email
DELETE /api/organizations/me/members/:userId # Remove member
```

All endpoints require a valid Clerk JWT in the `Authorization: Bearer <token>` header. All responses are `application/json`. The `/api/chat` endpoint returns `text/event-stream`.

---

## 9. Future Features

These are out of scope for the MVP but represent the natural product roadmap post-hackathon.

### Near-term (1–3 months)

| Feature | Description |
|---|---|
| **OCR for scanned PDFs** | Integrate AWS Textract to handle image-based PDFs, not just text-based ones |
| **Clause library** | User-curated library of approved/rejected clause templates for their organization |
| **Bulk upload** | Upload and analyze multiple contracts in a single batch operation |
| **Slack / Teams integration** | Push key date alerts and high-risk findings to team channels |
| **Webhook support** | Notify external systems when an analysis completes or a key date approaches |

### Medium-term (3–6 months)

| Feature | Description |
|---|---|
| **Contract templates** | Generate contract drafts from templates with AI-assisted clause selection |
| **Counterparty risk scoring** | Aggregate risk profile per counterparty across all contracts |
| **Audit trail** | Immutable log of all actions per contract for compliance purposes |
| **SSO / SAML** | Enterprise authentication for larger organizations |
| **API access** | Public REST API for programmatic contract submission and result retrieval |

### Long-term (6–12 months)

| Feature | Description |
|---|---|
| **Multi-language support** | Analyze contracts in Spanish, French, German, Portuguese |
| **Jurisdiction-aware analysis** | Tailor risk assessment to the governing law of the contract |
| **E-signature integration** | Connect with DocuSign / HelloSign to analyze before signing in one flow |
| **Legal network marketplace** | Connect users with vetted lawyers for contracts flagged as critical risk |
| **Mobile app** | iOS / Android app for reviewing contracts on the go |

---

## 10. Changelog

### v1.1.0 — June 2026

- Replaced OpenAI GPT-4o with Anthropic Claude 3.5 Sonnet via Amazon Bedrock
- Lambda analysis worker now authenticates to Bedrock via IAM role — no external API key
- Chat streaming updated to use `@ai-sdk/amazon-bedrock` provider with Vercel AI SDK
- Structured output migrated from OpenAI function calling to Claude tool use
- Removed `OPENAI_API_KEY` from required environment variables
- Added `bedrock:InvokeModel` to Lambda IAM execution role

### v1.0.0 — June 2026 (Hackathon MVP)

- Initial release for H01 Hackathon
- PDF upload via S3 presigned URLs
- Async AI analysis pipeline: SQS → Lambda → Anthropic Claude 3.5 Sonnet (Amazon Bedrock) → Aurora PostgreSQL
- Risk Score gauge (0–100) with animated display
- Findings panel with PDF clause highlighting
- Chat with the contract (Vercel AI SDK + `@ai-sdk/amazon-bedrock` streaming)
- Key dates extraction and calendar view
- Dashboard with stats and contract list
- Multi-tenant workspaces with Clerk authentication
- AWS SES email alerts for key dates
- Export analysis as PDF report

---

## 11. Contributing

This project was built for H01 Hackathon. Post-hackathon contributions are welcome.

### Getting Started

```bash
# Clone the repository
git clone https://github.com/<org>/contractlens.git
cd contractlens

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env.local
# Fill in: CLERK_*, DATABASE_URL, AWS_*, S3_BUCKET

# Run database migrations
npm run db:migrate

# Start the development server
npm run dev
```

### Environment Variables

| Variable | Description |
|---|---|
| `CLERK_SECRET_KEY` | Clerk backend secret key |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk frontend publishable key |
| `DATABASE_URL` | Aurora PostgreSQL connection string |
| `AWS_REGION` | AWS region (e.g., `us-east-1`) |
| `AWS_ACCESS_KEY_ID` | AWS access key (S3, SQS, SES, Bedrock — used by Vercel API Routes) |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key (used by Vercel API Routes) |
| `S3_BUCKET_NAME` | S3 bucket for PDF storage |
| `SQS_QUEUE_URL` | SQS queue URL for analysis jobs |
| `SES_FROM_EMAIL` | Verified SES sender address |
| `BEDROCK_MODEL_ID` | Bedrock model ID (e.g., `anthropic.claude-3-5-sonnet-20241022-v2:0`) |

> **Note:** Lambda→Bedrock calls use the Lambda IAM execution role (`bedrock:InvokeModel`). No `OPENAI_API_KEY` or separate Bedrock API key is required.

### Branch Strategy

- `main` — production-ready code, deployed to Vercel
- `dev` — integration branch for feature work
- `feature/<name>` — individual feature branches

### Pull Request Guidelines

- Keep PRs focused on a single feature or fix
- Include a description of what changed and why
- All API changes must update the OpenAPI spec
- Run `npm run lint` and `npm run typecheck` before opening a PR

---

## 12. License

MIT License

Copyright (c) 2026 ContractLens

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

## 13. Conclusion

ContractLens addresses a real, measurable problem: the legal exposure gap that affects the majority of small and medium-sized businesses. By combining a Vercel-hosted frontend with an AWS-native async processing pipeline (S3 → SQS → Lambda → Aurora PostgreSQL), the system delivers a production-grade architecture that is scalable, cost-efficient, and resilient — not just a hackathon prototype.

The product is designed to be monetizable from day one, with a clear free-to-paid conversion path, a pricing model grounded in the value delivered (one avoided bad clause pays for years of subscription), and a roadmap that extends naturally into enterprise features, multi-language support, and legal marketplace integrations.

The technical choices — presigned S3 uploads, async Lambda analysis, Vercel AI SDK streaming with Amazon Bedrock, and Aurora PostgreSQL's relational + JSONB hybrid model — are each justified by concrete constraints and tradeoffs, not convenience. Using Anthropic Claude 3.5 Sonnet via Amazon Bedrock keeps the entire AI pipeline within AWS: Lambda authenticates via IAM role, no external API key is needed, and all LLM traffic stays on AWS's private network. The result is a system that can handle real contract volumes, real team collaboration, and real business decisions.

> **Legal Disclaimer:** ContractLens is an AI-powered assistance tool. It does not constitute legal advice and does not replace consultation with a qualified attorney. Always consult a legal professional before signing important documents.

---

*ContractLens SPEC v1.1.0 — H01 Hackathon (h01.devpost.com) — June 2026*
