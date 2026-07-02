# 🤖 AI SDR — Autonomous Sales Development Representative

> An end-to-end AI-powered sales automation system built with **n8n**, **Groq (Llama 3.1)**, and **100% free APIs**.
> Discovers leads, researches companies, scores prospects, and sends personalized outreach — fully automatically, at **$0/month**.

Built as a portfolio project demonstrating **production-grade agentic AI workflow design**, fault-tolerant automation, API cost optimization, and real business value delivery using free-tier infrastructure.

---

## 📋 Table of Contents

- [What It Does](#what-it-does)
- [Business Value](#business-value)
- [Tech Stack](#tech-stack)
- [Architecture & Design Decisions](#architecture--design-decisions)
- [Workflow Structure](#workflow-structure)
- [Setup Guide](#setup-guide)
- [Sample Output](#sample-output)
- [Limitations](#limitations)
- [Skills Demonstrated](#skills-demonstrated)
- [Author](#author)

---

## What It Does

A sales manager fills **one form**. Everything else is automatic.

```
Form Submission (industry + location + job title + your service)
        ↓
LinkedIn Profile Discovery     [Serper API — Google search]
        ↓
Lead Extraction + Identity     [unique leadId per person]
        ↓
Sequential Processing          [Split In Batches — 1 at a time]
        ↓
Company Domain Discovery       [Serper — targeted web search]
        ↓
Email Discovery                [Serper — public email extraction]
        ↓
Company Research               [Serper — funding, news, growth signals]
        ↓
AI Scoring + Message Draft     [Groq / Llama 3.1 — single optimized call]
        ↓
Score >= 7?
  YES → Email found? → Send via Gmail
         No email?   → Log LinkedIn draft
  NO  → Append to Review Queue
        ↓
Log to Google Sheets CRM
```

**Input:** One Tally form with 5 fields
**Output:** Personalized outreach sent, CRM updated, low-quality leads queued for human review

---

## Business Value

| Manual SDR Process | This System |
|---|---|
| 30–45 min per lead (research + write + send) | **Under 2 minutes** per lead |
| $200–$500/month (Apollo.io, Clay, Hunter.io) | **$0/month** |
| Generic emails with no personalization | Emails referencing **real company facts** (funding, products, hiring signals) |
| Manual one-by-one processing | **Batch processing** from a single form submission |
| No automatic quality filtering | AI **scores every lead 1–10**, routes low-quality to human review |
| Human error — wrong email to wrong person | **Zero data contamination** via sequential processing architecture |

### 💰 ROI Calculation

A sales team processing **20 leads/day** saves **8–12 hours of manual work daily**.
At $25/hr, that's **$50,000+ per year** in recovered time per sales rep.
Replacing tools like Apollo.io or Clay saves an additional **$2,400–$6,000/year** in software costs.

---

## Tech Stack

| Component | Tool | Monthly Cost |
|---|---|---|
| Workflow orchestration | n8n (self-hosted) | Free |
| LinkedIn profile discovery | Serper.dev | Free (2,500 searches/mo) |
| Company domain research | Serper.dev | Free |
| Email discovery | Serper.dev (public email extraction) | Free |
| Lead scoring + message drafting | Groq API — Llama 3.1-8b-instant | Free |
| Email delivery | Gmail API (OAuth2) | Free |
| CRM / logging | Google Sheets API | Free |
| Form intake | Tally.so | Free |

### Total monthly cost: $0

---

## Architecture & Design Decisions

### 1. Sequential Processing via Split In Batches

`Split In Batches (size=1)` processes one lead at a time through the entire pipeline before moving to the next. No parallel branches, no index-based lookups, zero cross-contamination between leads.

### 2. Lead Identity Preservation via JavaScript Spread

Every node carries the **complete lead object forward** using JavaScript spread:

```javascript
return [{ json: { ...lead, newField: newValue } }];
```

No node ever reaches back to a previous node by index. The lead object is the **single source of truth** from extraction to CRM logging.

### 3. Single Groq Call — 50% LLM Cost Reduction

Scoring and message drafting are merged into a single structured Groq prompt that returns one JSON object with both outputs:

```json
{
  "score": 9,
  "hooks": ["$44M Goldman Sachs funding", "expanding enterprise partnerships"],
  "reasoning": "Strong budget signals and growth trajectory",
  "subject": "Congrats on the Goldman raise, Nav",
  "body": "Hi Nav, saw Goldman Sachs led your $44M round..."
}
```

**One API call. One structured response. 50% fewer tokens.**



### 4. Fault Tolerance — No Single Point of Failure

Every external API node has **Continue on Error** enabled. Graceful fallbacks at every stage:

| Failure Scenario | Fallback Behavior |
|---|---|
| Serper returns no results | `research = "No research available"` → pipeline continues |
| Groq returns malformed JSON | `try/catch` returns safe defaults (score=5) → pipeline continues |
| Email not found | `outreachMethod = "linkedin"` → LinkedIn draft logged instead |
| Invalid or missing domain | Validation check skips email search → no API error thrown |

No single failure stops the pipeline. Every lead is processed to completion.

### 5. Smart Lead Routing with AI Scoring

Leads are not treated equally. The scoring prompt uses explicit, deterministic criteria:

- **8–10:** Tech company with budget signals (funding rounds, hiring, growth trajectory)
- **5–7:** Unclear fit or limited signals — queued for human review
- **1–4:** Local business, no tech or budget signals — deprioritized

**Score >= 7 → automated outreach. Score < 7 → human review queue.**

The system makes routing decisions autonomously — it doesn't just trigger actions, it exercises judgment.

### 6. Domain Validation Before API Calls

Before any email discovery attempt, the extracted domain is validated:

```javascript
const validDomain = domain && domain.includes('.') && domain.length > 4
  && !domain.includes(' ') && !domain.includes('/');
```

Invalid domains skip the email search stage entirely, preventing wasted API calls and downstream errors.

---

## Workflow Structure

```
Tally Form Webhook
    → Parse Form Data
    → Find LinkedIn Profiles        [Serper — Google search]
    → Extract Leads                 [assign unique leadId, split into items]
    → Process One at a Time         [Split In Batches, size=1]
    → Build Domain Search
    → Find Company Domain           [Serper]
    → Extract Domain                [with validation]
    → Build Email Search
    → Has Domain?
        YES → Find Email via Serper
                → Merge Email Result
        NO  → No Domain Path (outreachMethod = "linkedin")
    → Build Research Query
    → Research Company              [Serper — funding, news, signals]
    → Build Groq Prompt             [score + draft in one call]
    → Score and Draft               [Groq / Llama 3.1-8b-instant]
    → Parse Groq Response           [try/catch with safe defaults]
    → Score >= 7?
        YES → Email or LinkedIn?
                → Email    → Send Email [Gmail] → Log to Leads Sheet
                → LinkedIn → Log to Leads Sheet (draft logged)
        NO  → Append to Review Queue
    → Loop back to Split In Batches
```

**Total nodes: 19 | External APIs: 3 (Serper, Groq, Gmail/Sheets) | Conditional branches: 3**

---

## Setup Guide

### Prerequisites

- [n8n](https://n8n.io/) installed locally (`npm install -g n8n`) or self-hosted
- Google account (for Gmail + Google Sheets)
- Free API accounts:
  - [Serper.dev](https://serper.dev) — Google Search API
  - [Groq](https://console.groq.com) — LLM inference (Llama 3.1)

### Step 1 — Import the Workflow

Download `AI SDR - Production Ready.json` and import into n8n:

```
Open n8n → New Workflow → ⋮ menu → Import from file
```

### Step 2 — Set API Keys

Replace the placeholder keys in the following nodes:

| Node | Field | Value |
|---|---|---|
| Find LinkedIn Profiles | `X-API-KEY` header | Your Serper key |
| Find Company Domain | `X-API-KEY` header | Your Serper key |
| Find Email via Serper | `X-API-KEY` header | Your Serper key |
| Research Company | `X-API-KEY` header | Your Serper key |
| Score and Draft (Groq) | `Authorization` header | `Bearer your-groq-key` |
| Build Email Search | jsCode line 8 | Your Serper key |

### Step 3 — Configure OAuth2 Credentials

| Node | Credential Type |
|---|---|
| Log to Leads Sheet | Google Sheets OAuth2 |
| Append to Review Queue | Google Sheets OAuth2 |
| Send Email | Gmail OAuth2 |

### Step 4 — Create Google Sheet

Create a spreadsheet named **Lead Agent** with two tabs:

**Leads tab** — columns in order:
```
A: name | B: company | C: website | D: email | E: status | F: score | G: hooks_found | H: processed_at
```

**Review Queue tab** — columns in order:
```
A: name | B: company | C: email | D: score | E: hooks | F: reasoning | G: added_at
```

### Step 5 — Connect Tally Form

1. Create a form at [tally.so](https://tally.so) with these 5 fields:
   - Industry
   - Location
   - Job title to target
   - Number of leads
   - Your service or product

2. Go to **Integrate → Webhooks** → paste your n8n webhook URL:
   ```
   https://your-n8n-instance.com/webhook/ai-sdr
   ```

### Step 6 — Activate and Test

1. Toggle the workflow to **Active** in n8n
2. Submit the Tally form with a test query
3. Check your Google Sheet — leads should populate within 2–3 minutes
4. Check your Gmail sent folder for dispatched outreach

---

## Sample Output

### Leads Sheet

| Name | Company | LinkedIn | Email | Status | Score | Hooks Found |
|---|---|---|---|---|---|---|
| Harry Thomsen | aareon.com | /in/harry-thomsen | last@aareon.com | email | 9 | $28.1M IT spend, ERP for property industry |
| Nav Sandhu | — | /in/navsandhu | — | linkedin | 9 | Goldman Sachs $44M funding |

### Email Sent to Harry

```
Subject: Aareon's $28.1M IT roadmap — quick question

Hi Harry, I came across Aareon's projected $28.1M IT spend this year and was
impressed by your organization's investment in ERP infrastructure. Managing
enterprise property portfolios at that scale comes with real operational complexity.
[Your service] has helped similar organizations reduce manual overhead by 40%.
Would a 15-minute call make sense to explore if there's a fit?
```

### Review Queue

Low-scoring leads are written to a separate sheet tab with the AI's reasoning, allowing a human to decide whether to pursue them manually.

---

## Limitations

| Limitation | Detail |
|---|---|
| Email discovery hit rate | ~20–30% (only publicly listed emails surfaced) |
| Serper free tier | 2,500 searches/month (~125 leads/month at ~20 searches per lead) |
| Groq free tier | Generous limits; sufficient for portfolio and demo volume |
| n8n branding | Appears in emails on the self-hosted free version |
| LinkedIn scraping | Indirect via Google search; no direct LinkedIn API used |

---

## Skills Demonstrated

- **Agentic AI workflow design** with n8n — multi-step, decision-making pipelines
- **LLM integration** (Groq / Llama 3.1) for automated scoring and message generation
- **Production architecture** — fault tolerance, graceful degradation, error handling at every node
- **API cost optimization** — single LLM call replacing two; Hunter.io replaced with free Serper extraction
- **Data integrity engineering** — lead identity preservation via JavaScript object spread
- **Real business problem framing** — mapped to SDR cost reduction and productivity gain
- **JavaScript** — n8n Code nodes for data transformation, validation, and fallback logic
- **Google Sheets as lightweight CRM** — automated append with structured schema
- **OAuth2 credential management** — Gmail and Google Sheets API integration
- **Webhook-driven automation** — Tally form → n8n → multi-service orchestration

---

## Author

**Ammara Tariq** 

[LinkedIn](https://linkedin.com/in/ammara) · [GitHub](https://github.com/ammara) · ammaratariq443@gmail.com

---

*Built with n8n · Groq · Serper · Gmail API · Google Sheets API · Tally.so*
