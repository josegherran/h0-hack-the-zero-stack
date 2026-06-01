# ContractLens — Detailed Product Document

> **Hackathon:** H01 (h01.devpost.com)  
> **Track:** Monetizable B2B App  
> **Type:** SaaS Web Application  
> **Date:** June 2026

---

## 1. Executive Summary

**ContractLens** is a SaaS web application that allows small and medium-sized businesses to upload contracts in PDF format and receive an intelligent analysis in seconds: identification of risky clauses, comparison against industry standards, an executive summary, and actionable recommendations — all without needing an in-house lawyer.

> **Problem:** 78% of SMBs sign contracts without proper legal review, exposing themselves to abusive clauses, hidden penalties, and unreasonable commitments. Hiring an external lawyer costs $200–$500 per contract and takes days.
>
> **Solution:** Instant, accessible, and affordable basic legal analysis for any company.

---

## 2. Value Proposition

| For whom | The problem | ContractLens offers |
|---|---|---|
| SMBs without in-house legal | Sign contracts without fully understanding them | Risk analysis in seconds |
| Entrepreneurs / Freelancers | Cannot afford legal advisory fees | Affordable basic review |
| Procurement / Sales teams | Complex vendor contracts | Benchmarking against industry standards |
| Growing startups | Many contracts, little time | Centralization and full traceability |

---

## 3. Core Features (MVP)

### 3.1 AI Contract Analysis

- **PDF Upload** — drag and drop or file picker.
- **Text extraction** — PDF parsing with structure detection.
- **AI Analysis** — using LLM (OpenAI GPT-4o) to identify:
  - Limited or unlimited liability clauses.
  - Breach penalties.
  - Exclusivity or non-compete clauses.
  - Contract termination conditions.
  - Payment timelines and commercial terms.
  - Governing law and jurisdiction.
  - Auto-renewal and exit conditions.
- **Overall risk level** — Low / Medium / High / Critical (color-coded with a 0–100 score).
- **Executive summary** — plain-language paragraph explaining the contract.
- **Recommendations** — prioritized list of points to negotiate or reject.

### 3.2 Contracts Dashboard

- List view of all analyzed contracts.
- Filters by: date, contract type, risk level, tag.
- Search by name, company, or clause.
- Summary stats: total contracts, risk distribution, contracts expiring soon.

### 3.3 Analysis Detail View

- Integrated PDF viewer with overlaid annotations.
- Side panel with findings organized by category.
- Each finding includes: contract excerpt, explanation, risk level, and suggestion.
- Ability to mark findings as "reviewed" or "accepted".
- Export analysis report as PDF.

### 3.4 Standards Comparison

- Database of standard clauses by contract type (NDA, SaaS, Vendor, Employment, Lease).
- For each identified clause, shows whether it falls within or outside typical market range.
- Badges: "Market Standard" / "Uncommon" / "Unusually Restrictive".

### 3.5 Version History

- Save multiple versions of the same contract (v1, v2 negotiated, etc.).
- Compare changes between versions with a visual diff.
- Notes and comments per version.

### 3.6 Alerts & Reminders

- Key dates automatically extracted: expiration, renewal, payment milestones.
- Email notifications with configurable advance notice (30 / 15 / 7 days).
- Calendar view showing all active milestones.

### 3.7 Teams & Collaboration

- Per-company workspaces with multiple users.
- Roles: Admin, Editor, Viewer.
- Comments on findings with @mentions.
- Activity log per contract.

---

## 4. Wow Features (Differentiators)

| Feature | Description | Jury impact |
|---|---|---|
| **Animated Risk Score** | Visual gauge that fills from 0 to 100 as the analysis loads | Best Design |
| **Dangerous clause highlight** | Hovering a finding scrolls the PDF and highlights the exact clause | Best Technical |
| **Chat with the contract** | Ask in plain language: "When does this contract expire?" | Most Original |
| **Industry benchmark** | Compare your contracts against your sector's standards | Most Impactful |
| **Negotiation mode** | Suggests alternative wording for problematic clauses | Most Original |

---

## 5. Technical Architecture

### 5.1 Stack

| Layer | Technology |
|---|---|
| **Frontend** | React + Vite + TypeScript |
| **UI Components** | shadcn/ui + Tailwind CSS |
| **Backend** | Express 5 + Node.js 24 |
| **Database** | AWS Aurora PostgreSQL *(required by hackathon)* |
| **ORM** | Drizzle ORM |
| **Validation** | Zod v4 |
| **AI / LLM** | OpenAI GPT-4o (via Replit AI Integrations) |
| **PDF parsing** | pdf-parse + pdf-lib |
| **File storage** | Replit Object Storage (PDF files) |
| **Frontend deployment** | Vercel *(required by hackathon)* |
| **Authentication** | Clerk |
| **Email** | Resend |
| **API codegen** | Orval (OpenAPI → React Query hooks) |

### 5.2 Analysis Flow Diagram

```
User uploads PDF
       ↓
Frontend → POST /api/contracts (multipart/form-data)
       ↓
Backend saves PDF to Object Storage
       ↓
Backend extracts text (pdf-parse)
       ↓
Backend calls OpenAI GPT-4o with specialized prompt
       ↓
LLM returns structured JSON with findings
       ↓
Backend persists analysis in Aurora PostgreSQL
       ↓
Backend returns analysis to frontend
       ↓
Frontend renders Risk Score + findings + PDF viewer
```

### 5.3 Data Model (Aurora PostgreSQL)

```sql
-- Organizations (company workspace)
organizations
  id              UUID PK
  name            TEXT NOT NULL
  plan            TEXT DEFAULT 'free'   -- free | starter | pro
  created_at      TIMESTAMPTZ

-- Users
users
  id              UUID PK
  organization_id UUID FK → organizations
  email           TEXT UNIQUE
  name            TEXT
  role            TEXT   -- admin | editor | viewer
  created_at      TIMESTAMPTZ

-- Contracts
contracts
  id              UUID PK
  organization_id UUID FK → organizations
  uploaded_by     UUID FK → users
  name            TEXT NOT NULL
  counterparty    TEXT              -- name of the other party
  contract_type   TEXT              -- NDA | SaaS | Vendor | Employment | Other
  file_url        TEXT              -- URL in Object Storage
  file_name       TEXT
  file_size_bytes INT
  status          TEXT DEFAULT 'pending'  -- pending | analyzing | done | error
  created_at      TIMESTAMPTZ
  updated_at      TIMESTAMPTZ

-- Analyses (LLM output)
analyses
  id              UUID PK
  contract_id     UUID FK → contracts
  version         INT DEFAULT 1
  risk_score      INT               -- 0-100
  risk_level      TEXT              -- low | medium | high | critical
  summary         TEXT              -- executive summary
  raw_llm_output  JSONB             -- raw LLM response
  model_used      TEXT
  tokens_used     INT
  duration_ms     INT
  created_at      TIMESTAMPTZ

-- Individual findings
findings
  id              UUID PK
  analysis_id     UUID FK → analyses
  category        TEXT   -- liability | payment | termination | exclusivity | jurisdiction | renewal | other
  title           TEXT
  excerpt         TEXT   -- contract fragment
  explanation     TEXT   -- plain-language explanation
  risk_level      TEXT   -- low | medium | high | critical
  suggestion      TEXT   -- recommendation
  page_number     INT
  char_offset     INT    -- text position for highlight
  is_standard     BOOL   -- is this a market-standard clause?
  status          TEXT DEFAULT 'open'  -- open | reviewed | accepted
  created_at      TIMESTAMPTZ

-- Extracted key dates
key_dates
  id              UUID PK
  contract_id     UUID FK → contracts
  label           TEXT   -- "Expiration date", "Auto-renewal", etc.
  date            DATE
  notified_30d    BOOL DEFAULT false
  notified_7d     BOOL DEFAULT false
  created_at      TIMESTAMPTZ

-- Comments
comments
  id              UUID PK
  finding_id      UUID FK → findings
  user_id         UUID FK → users
  body            TEXT
  created_at      TIMESTAMPTZ
```

---

## 6. App Screens / Pages

### 6.1 Route Map

| Route | Screen | Description |
|---|---|---|
| `/` | Landing Page | Public marketing page |
| `/login` | Login | Authentication with Clerk |
| `/register` | Register | New organization onboarding |
| `/dashboard` | Dashboard | Overview of all contracts |
| `/contracts/new` | Upload | Upload a new contract |
| `/contracts/:id` | Detail | Full contract analysis view |
| `/contracts/:id/versions` | Versions | Version history and comparison |
| `/calendar` | Calendar | Key date milestones and alerts |
| `/settings` | Settings | Profile, team, plan, and billing |

### 6.2 Screen Descriptions

#### Landing Page (`/`)
- Hero with value proposition and CTA "Analyze your first contract for free".
- Animated demo showing the analysis flow.
- Key features section.
- Pricing plans.
- Testimonials / use cases.
- Legal footer.

#### Dashboard (`/dashboard`)
- Top bar with global search.
- Stats cards: total contracts, high-risk contracts, contracts expiring in 30 days, analyses this month.
- Contracts table with columns: Name, Counterparty, Type, Risk, Upload Date, Actions.
- Quick filters by risk and type.
- Prominent "New Analysis" button.

#### Upload (`/contracts/new`)
- Drag & drop area for PDF files.
- Optional fields: contract name, counterparty, contract type, tags.
- Progress bar during analysis with status messages ("Extracting text...", "Analyzing clauses...", "Generating report...").
- Automatically redirects to the detail view when complete.

#### Analysis Detail (`/contracts/:id`)
- Split layout: PDF viewer (left) + findings panel (right).
- Header with: contract name, type, counterparty, Risk Score gauge, analysis date.
- Tabs in right panel: Summary | Findings | Key Dates | Benchmark | Chat.
- Each finding: risk icon, title, excerpt, explanation, suggestion, "Mark as reviewed" button.
- Clicking a finding → PDF scrolls to and highlights the corresponding clause.

#### Calendar (`/calendar`)
- Monthly view with milestones color-coded by urgency.
- List of upcoming 30 days in a side panel.
- Badge for contracts with auto-renewal within the next 30 days.

---

## 7. API Endpoints (OpenAPI)

```yaml
POST   /api/contracts                  # Upload and create contract
GET    /api/contracts                  # List organization's contracts
GET    /api/contracts/:id              # Get contract with analysis
DELETE /api/contracts/:id              # Delete contract

POST   /api/contracts/:id/analyze      # Re-analyze (new version)
GET    /api/contracts/:id/analyses     # Analysis version history

PATCH  /api/findings/:id               # Update finding status
POST   /api/findings/:id/comments      # Add comment
GET    /api/findings/:id/comments      # List comments

GET    /api/key-dates                  # Key dates across all contracts
GET    /api/key-dates/upcoming         # Upcoming within the next N days

GET    /api/dashboard/stats            # Summary stats for dashboard
GET    /api/dashboard/risk-distribution # Risk distribution (for chart)

GET    /api/organizations/me           # Current organization data
PATCH  /api/organizations/me           # Update organization settings
GET    /api/organizations/me/members   # List members
POST   /api/organizations/me/members   # Invite member
DELETE /api/organizations/me/members/:userId  # Remove member
```

---

## 8. Analysis Prompt (AI)

The analysis is performed with a specialized **system prompt** that instructs the LLM to return structured JSON:

```
You are a lawyer specializing in commercial contracts for businesses.
Analyze the following contract and return a JSON with:

{
  "risk_score": 0-100,
  "risk_level": "low|medium|high|critical",
  "summary": "Executive summary in 2-3 sentences",
  "contract_type": "NDA|SaaS|Vendor|Employment|Lease|Other",
  "findings": [
    {
      "category": "liability|payment|termination|exclusivity|jurisdiction|renewal|other",
      "title": "Short finding title",
      "excerpt": "Literal fragment from the contract",
      "explanation": "Plain-language explanation for a non-lawyer",
      "risk_level": "low|medium|high|critical",
      "suggestion": "What to do about it",
      "is_standard": true|false
    }
  ],
  "key_dates": [
    {
      "label": "Date name",
      "date": "YYYY-MM-DD or null if undeterminable",
      "context": "Clause from which it was extracted"
    }
  ],
  "negotiation_points": ["point 1", "point 2"]
}

Focus on: risk clauses for the signing company, not the counterparty.
Be conservative: when in doubt, flag as medium or high risk.
```

---

## 9. Monetization Model

### SaaS Plans

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

### Revenue Projections (6 months post-hackathon)

| Scenario | Starter Users | Pro Users | MRR |
|---|---|---|---|
| Conservative | 50 | 10 | $2,240 |
| Base | 200 | 50 | $9,750 |
| Optimistic | 500 | 150 | $26,350 |

---

## 10. Hackathon Criteria — How ContractLens meets them

| Criterion | How it's met | Expected score |
|---|---|---|
| **Technical Implementation** | Aurora PostgreSQL with a robust relational model, OpenAPI-first architecture, async analysis with error handling, PDF viewer with clause highlights | High |
| **Design** | Animated Risk Score gauge, PDF+analysis split view, risk-level color coding, clarity-focused UX | High |
| **Impact & Real-world Applicability** | Real problem with demonstrable ROI, clear business model, ready to monetize from day one | Very high |
| **Originality** | Chat with the contract, synchronized PDF↔finding highlight, negotiation mode with alternative wording suggestions | High |
| **Bonus** | Publish blog + demo video with #H0Hackathon | +0.6 pts |

**Potential special category prizes:**
- 🏆 **Most Impactful** — direct impact on reducing legal risk for SMBs
- 🥇 **Best Technical Implementation** — Aurora PostgreSQL + LLM + PDF parsing integration

---

## 11. Hackathon Demo Video

**Recommended duration:** 2–3 minutes

**Script:**

1. **(0:00–0:20)** — Present the problem: "How many SMBs sign contracts without legal review?"
2. **(0:20–0:45)** — Show uploading a real contract (NDA or SaaS agreement).
3. **(0:45–1:30)** — Watch the live analysis: Risk Score appears, findings are listed, click one and the PDF scrolls to the exact clause.
4. **(1:30–2:00)** — Show the "Chat with the contract": ask "Are there any auto-renewal clauses?" and see the response.
5. **(2:00–2:30)** — Dashboard with multiple contracts, risk distribution chart, key dates calendar.
6. **(2:30–3:00)** — Show the Aurora PostgreSQL database with stored data and close with a CTA.

---

## 12. Development Plan (5 days)

### Day 1 — Infrastructure + Auth
- [ ] Create React + Vite project on Vercel
- [ ] Configure Aurora PostgreSQL + Drizzle ORM
- [ ] Implement authentication with Clerk
- [ ] Define and apply database schema
- [ ] Set up Object Storage for PDFs

### Day 2 — Backend Core
- [ ] Contract upload endpoint (multipart)
- [ ] pdf-parse integration for text extraction
- [ ] Analysis prompt + OpenAI GPT-4o integration
- [ ] Persist analysis and findings in Aurora PostgreSQL
- [ ] CRUD endpoints for contracts and analyses

### Day 3 — Frontend Core
- [ ] General layout + navigation
- [ ] Upload page with drag & drop + progress bar
- [ ] Dashboard with contract list and stats
- [ ] Detail view: PDF viewer + findings panel

### Day 4 — Wow Features
- [ ] Animated Risk Score gauge
- [ ] Synchronized PDF ↔ finding highlight
- [ ] Chat with the contract (streaming responses)
- [ ] Key dates calendar
- [ ] Export PDF report

### Day 5 — Polish + Deploy
- [ ] Negotiation mode (alternative clause wording)
- [ ] Team invitation system
- [ ] Email notifications (Resend)
- [ ] Public landing page
- [ ] Final deploy on Vercel + configure Aurora in production
- [ ] Record demo video

---

## 13. Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Scanned PDF (image, not text) | Medium | High | Warn the user; future OCR integration |
| Slow LLM analysis (+15 sec) | Medium | Medium | Streaming responses + animated progress UI |
| LLM hallucinations | Medium | High | Always show the original excerpt; disclaimer that it does not replace legal counsel |
| Token limit (long contracts) | Low | High | Chunk text by section before sending to the LLM |
| OpenAI cost during demo | Low | Medium | Cache analyses; use a pre-analyzed set of demo contracts |

---

## 14. Legal Disclaimer (required in the app)

> ContractLens is an AI-powered assistance tool designed to help identify potential areas of attention in contracts. **It does not constitute legal advice** and does not replace consultation with a qualified attorney. Analyses are indicative and may contain errors. Always consult a legal professional before signing important documents.

---

*Generated for H01 Hackathon (h01.devpost.com) — June 2026*