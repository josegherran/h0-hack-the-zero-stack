# ContractLens — Hackathon Submission Materials

> **Hackathon:** H01 (h01.devpost.com)  
> **Track:** Monetizable B2B App  
> **Date:** June 2026

---

## Part 1 — AWS Database: Text Description

### Which AWS Database we used and why

**ContractLens uses AWS Aurora PostgreSQL** as its primary database.

---

### Why Aurora PostgreSQL

Contract analysis is a fundamentally relational problem. A single uploaded document triggers a cascade of structured, interconnected records: the contract itself, the AI-generated analysis, dozens of individual findings, extracted key dates, user comments, and team activity logs. These entities have strict relationships and referential integrity requirements — an analysis cannot exist without a contract, a finding cannot exist without an analysis, a comment cannot exist without a finding. Aurora PostgreSQL enforces these constraints at the database level, giving us confidence that the data is always consistent regardless of concurrency or partial failures.

We evaluated all three options — Aurora PostgreSQL, Aurora DSQL, and DynamoDB — and chose Aurora PostgreSQL for the following specific reasons:

**1. Relational integrity for complex contract data**
The ContractLens data model has six interconnected tables with foreign key relationships. Relational databases handle this natively and efficiently with JOIN queries. Fetching a complete contract analysis — including its findings, key dates, and comments — is a single well-structured query in PostgreSQL. In DynamoDB, this would require multiple round-trips or denormalized, duplicated data structures that would make the codebase significantly harder to maintain.

**2. JSONB for flexible LLM output**
The raw output from the OpenAI GPT-4o model is stored in a `JSONB` column on the `analyses` table. Aurora PostgreSQL's native JSONB support lets us store the full LLM response without a fixed schema, while still being able to index and query specific fields inside the JSON when needed. This is a hybrid approach — strict schema where the data is well-defined, flexible schema where it is not — that is uniquely well-suited to Aurora PostgreSQL.

**3. Full-text search across contract analyses**
Users need to search across all their contracts by clause content, counterparty name, finding explanation, or recommendation text. Aurora PostgreSQL's built-in full-text search (`tsvector`, `tsquery`) handles this natively without an external search service, keeping the architecture simple and the latency low for the hackathon timeline.

**4. Transactional safety for multi-step analysis pipeline**
When the AI analysis completes, ContractLens performs a multi-step write: create the `analyses` record, then insert N `findings` records, then insert M `key_dates` records — all atomically. If any step fails, the entire transaction rolls back and the contract is marked with an `error` status for retry. Aurora PostgreSQL's ACID transactions make this safe and simple. A partial write that leaves a contract in an inconsistent state would be a critical bug for a legal-adjacent application where users are making decisions based on the data.

**5. Scalability path with Aurora's architecture**
Aurora PostgreSQL's serverless v2 configuration allows the database to scale compute capacity automatically based on load — from a fraction of an ACU during off-peak hours to many ACUs during analysis bursts. For a B2B SaaS with unpredictable usage patterns (teams analyze contracts in batches before board meetings, deal closings, or contract renewal seasons), this auto-scaling behavior provides both cost efficiency and performance without manual provisioning.

---

### How Aurora PostgreSQL is used in ContractLens

| Table | Purpose | Key Aurora feature used |
|---|---|---|
| `organizations` | Multi-tenant workspace per company | Row-level filtering by org ID on every query |
| `users` | Team members with roles | FK constraints + unique email enforcement |
| `contracts` | Uploaded contract metadata + status | Indexed queries by status, type, risk level |
| `analyses` | AI analysis results per contract version | JSONB column for raw LLM output |
| `findings` | Individual clause findings from AI | Full-text search on explanation + excerpt |
| `key_dates` | Auto-extracted important dates | Date range queries for upcoming alerts |
| `comments` | Team collaboration on findings | Cascading deletes tied to parent finding |

**Example query — Dashboard stats (single Aurora PostgreSQL query):**
```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'done')           AS total_analyzed,
  COUNT(*) FILTER (WHERE risk_level = 'high'
                      OR risk_level = 'critical')   AS high_risk_count,
  COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '30 days') AS analyzed_this_month
FROM contracts
JOIN analyses ON analyses.contract_id = contracts.id
WHERE contracts.organization_id = $1;
```

This single query replaces what would be multiple DynamoDB scans with filtering in application code — demonstrating the practical advantage of the relational model for analytics workloads.

---

## Part 2 — Video Storyline

### Demo Video Script — ContractLens
**Format:** Screen recording + voiceover  
**Target duration:** 3–5 minutes  
**Platform:** YouTube (unlisted or public)  
**Hashtag:** #H0Hackathon

---

### Full Script with Timecodes

---

#### [0:00 – 0:30] — Hook: The Problem

**[Scene: Plain slide or animated text on screen]**

**Voiceover:**
> "Every week, thousands of small businesses sign contracts they don't fully understand. NDAs with unlimited liability clauses. Vendor agreements with automatic renewals that are nearly impossible to exit. SaaS subscriptions with data ownership clauses buried on page 12. Most small companies can't afford a lawyer to review every contract. So they sign anyway — and hope for the best."

**[Scene: Cut to a frustrated person scrolling through a dense 15-page PDF contract]**

> "We built ContractLens to fix that."

---

#### [0:30 – 1:00] — Who it's for and why we chose this problem

**[Scene: Simple graphic showing SMB owners, freelancers, startup founders]**

**Voiceover:**
> "ContractLens is built for small and medium-sized businesses — the 30-person SaaS company reviewing a new enterprise customer agreement, the freelance designer receiving a client contract, the startup founder signing a vendor deal. People who are smart enough to know contracts matter, but who don't have a lawyer on speed dial."

> "We chose this problem because the risk is real and the cost is measurable. A single bad clause — an uncapped liability provision, a non-compete that's too broad, an auto-renewal you missed — can cost a small business tens of thousands of dollars. The solution shouldn't require a $400-per-hour attorney. It should take 30 seconds."

---

#### [1:00 – 1:30] — Product walkthrough: Upload

**[Scene: Live screen recording of the ContractLens web app]**

**Voiceover:**
> "Here's ContractLens in action. From the dashboard, I'll click 'New Analysis' and upload a real vendor agreement — a standard SaaS subscription contract."

**[Action: Drag and drop a PDF onto the upload area. Show the contract name and type fields being filled in.]**

> "I can optionally label the contract type and name the counterparty. Then I hit Analyze."

**[Action: Click Analyze. Show the animated progress bar with status messages: 'Extracting text... Analyzing clauses... Generating report...']**

> "In about 15 seconds, the AI has read every clause in this contract."

---

#### [1:30 – 2:30] — Product walkthrough: Analysis results

**[Scene: The analysis detail view loads. The Risk Score gauge animates from 0 to 74 — 'High Risk'.]**

**Voiceover:**
> "The risk score lands at 74 — High. ContractLens found 8 issues worth your attention."

**[Action: Scroll through the findings panel on the right side.]**

> "Each finding shows you the exact clause from the contract, explains it in plain English, tells you why it's a risk, and suggests what to do about it."

**[Action: Click on a finding — "Uncapped Liability Clause". The PDF viewer on the left scrolls automatically to page 4 and highlights the exact sentence in yellow.]**

> "When I click a finding, the PDF jumps directly to the clause in question and highlights it. No more hunting through 15 pages of legalese."

**[Action: Hover over the finding details. Show the badge: 'Unusually Restrictive — not market standard'.]**

> "ContractLens also benchmarks each clause against what's typical in the market. This liability clause is flagged as unusually restrictive — most vendor contracts in this category cap liability at the contract value."

---

#### [2:30 – 3:00] — Wow feature: Chat with the contract

**[Scene: Click on the 'Chat' tab in the right panel.]**

**Voiceover:**
> "One of our favorite features: you can talk to your contract. I'll ask it a plain-language question."

**[Action: Type in the chat: "What happens if I want to cancel before the contract ends?"]**

**[Scene: Streaming response appears, quoting the relevant clause and explaining the early termination fee in plain English.]**

> "The AI finds the termination clause, quotes it directly, and explains exactly what it means — including the 90-day notice requirement and the 25% early exit fee. No searching, no decoding legal language."

---

#### [3:00 – 3:30] — Dashboard & Calendar

**[Scene: Navigate back to the main dashboard.]**

**Voiceover:**
> "Back on the dashboard, I can see all my organization's contracts at a glance — risk levels, types, upcoming expirations."

**[Action: Show the stats cards: 12 total contracts, 3 high-risk, 2 expiring in 30 days.]**

**[Action: Navigate to the Calendar view.]**

> "The calendar view shows every key date ContractLens extracted automatically — renewal deadlines, payment milestones, notice periods. You can set email alerts so you never miss a critical date again."

**[Scene: Show a colored milestone on the calendar: "Auto-renewal — Acme Corp — 14 days".]**

---

#### [3:30 – 4:00] — AWS Aurora PostgreSQL

**[Scene: Simple diagram or terminal/database UI showing the Aurora PostgreSQL instance]**

**Voiceover:**
> "Under the hood, ContractLens runs on AWS Aurora PostgreSQL. Every contract, every AI-generated finding, every key date, and every team comment is stored in a fully relational Aurora database."

> "We chose Aurora PostgreSQL specifically because contract data is deeply relational — analyses belong to contracts, findings belong to analyses, comments belong to findings. Aurora's transactional guarantees mean that when the AI finishes analyzing a contract, all 20-plus findings are written atomically. If anything fails mid-write, the whole transaction rolls back. For a tool people use to make legal decisions, data consistency isn't optional."

**[Action: Briefly show a dashboard query returning stats or a findings list — demonstrating the app is backed by real data.]**

> "Aurora's JSONB support also lets us store the raw LLM output alongside our structured schema — the best of both worlds."

---

#### [4:00 – 4:20] — Closing & CTA

**[Scene: Back to the ContractLens landing page or app dashboard]**

**Voiceover:**
> "ContractLens is live, deployed on Vercel, backed by Aurora PostgreSQL, and ready for real users. Our free plan lets any small business analyze three contracts per month — no credit card required."

> "We built this in five days for H01 Hackathon. The problem is real. The solution works. And we're just getting started."

**[Scene: Show the URL / app link on screen]**

> "Try it at [your-vercel-url]. Thanks for watching — and if you found this useful, check out the GitHub repo linked below."

**[Fade out with #H0Hackathon on screen]**

---

### Video Production Notes

| Element | Recommendation |
|---|---|
| **Recording tool** | Loom, OBS, or QuickTime (Mac) |
| **Resolution** | 1920×1080 minimum |
| **Voiceover** | Record separately for clean audio; use a decent mic |
| **PDF to use** | Use a real (anonymized) NDA or SaaS vendor contract for authenticity |
| **Captions** | Add auto-captions in YouTube for accessibility |
| **Thumbnail** | Screenshot of the Risk Score gauge at "High Risk" — visually striking |
| **Description** | Include app URL, GitHub link, tech stack, and #H0Hackathon |
| **Title suggestion** | `ContractLens — AI Contract Analysis for SMBs | H01 Hackathon Demo` |

---

### Submission Checklist

- [ ] Video uploaded to YouTube (public or unlisted)
- [ ] Video covers: problem, target user, why this problem, working app demo, AWS database explanation
- [ ] GitHub repository is public
- [ ] Vercel project is live and accessible
- [ ] Storage configuration screenshot ready (Aurora PostgreSQL connection proof)
- [ ] Vercel Team ID noted
- [ ] Blog post or article published with #H0Hackathon (bonus +0.2 pts)
- [ ] Podcast or audio content published with #H0Hackathon (bonus +0.2 pts)
- [ ] Additional public content with #H0Hackathon (bonus +0.2 pts)

---

*Generated for H01 Hackathon (h01.devpost.com) — June 2026*