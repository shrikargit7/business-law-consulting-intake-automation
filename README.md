# Parr Law Intake Automation

A two-stage automated client intake system for law firms, combining a simple HTML form, ScoreApp readiness quiz, and AI-powered triage to streamline client onboarding and internal case preparation.

## Overview

This project automates the end-to-end intake process for business law clients:

1. **Stage 1 (Parr1)**: Client submits a basic intake form → data flows to a Google Sheet CRM
2. **Stage 2 (Parr2)**: Client completes a readiness quiz → AI generates structured intake summary, flags, and next steps

The result is a single enriched CRM row per client with all the information lawyers need before the first consultation.

---

## Architecture

```
┌─────────────────┐         ┌──────────────┐         ┌─────────────────┐
│  HTML Form      │────────▶│   Parr1 Zap  │────────▶│  Google Sheets  │
│  (index.html)   │ webhook │              │ add row │   (PARR_CRM)    │
└─────────────────┘         └──────────────┘         └─────────────────┘
                                                              │
                                                              │ lookup by email
                                                              ▼
┌─────────────────┐         ┌──────────────┐         ┌─────────────────┐
│  ScoreApp Quiz  │────────▶│   Parr2 Zap  │────────▶│  Google Sheets  │
│  (readiness)    │ trigger │  + AI model  │ update  │  (enriched row) │
└─────────────────┘         └──────────────┘         └─────────────────┘
```

---

## Parr1 – Initial Intake Form

### What it does
Captures basic client information and marketing attribution from a lightweight HTML form embedded on the firm's website.

### Form fields (`index.html`)
- **First name** / **Last name**
- **Email** (used as unique identifier across stages)
- **Phone**
- **Support for**: Myself / My company / Another person or company
- **Support type**: Business / Estate and Wills / Probate / Other
- **How did you hear about us?**: Professional advisor, referral, Google, social media, etc.

### Workflow (`parr1.json`)
1. **Trigger**: Zapier Catch Webhook receives form POST
2. **Action**: Create new row in `PARR_CRM` Google Sheet with:
   - Client contact info
   - Marketing source
   - Timestamp
   - Stage set to `New`

### Technical details
- Form validation ensures all required fields are filled
- JavaScript `fetch()` sends JSON to Zapier webhook
- No external dependencies or frameworks required

---

## Parr2 – Readiness Quiz + AI Triage

### What it does
Enriches the initial intake row with business readiness data from ScoreApp and generates AI-powered internal summaries.

### ScoreApp quiz questions
The quiz assesses:
- Business structure (sole proprietor, partnership, corporation, etc.)
- Number of founders/owners
- Existing legal agreements (shareholders, partnership, contracts)
- Jurisdiction and location
- Contract confidence level
- Main legal priority
- Desired timeline
- Overall readiness score and tier

### Workflow (`parr2.json`)
1. **Trigger**: ScoreApp quiz completion (for specific scorecard)
2. **Lookup**: Find matching row in `PARR_CRM` by email
3. **Update row**: Write quiz answers and readiness score to sheet
4. **AI step**: Generate structured outputs using Google Gemini 2.5 Flash
5. **Final update**: Write AI outputs back to the same row

### AI outputs

The AI model receives all quiz answers in a `Readiness` field and produces three structured outputs:

#### 1. `intake_summary`
A markdown-formatted summary with five sections:
- **Client & matter overview**: Goal, timeline, current structure
- **Business snapshot**: Stage, ownership, jurisdiction
- **Key risks and gaps**: Missing agreements, low contract confidence, etc.
- **Client's goals and success vision**: Priorities and success criteria
- **Suggested next steps (internal)**: Immediate actions for the firm

#### 2. `flags`
Comma-separated tags for filtering and triage:

**Structure/readiness**  
`foundations-needed`, `mostly-sorted`, `high-readiness`

**Ownership**  
`single-founder`, `multi-founder`, `no-owners-agreement`, `old-owners-agreement`

**Contracts**  
`contracts-needed`, `few-contracts`, `contracts-in-place`, `low-contract-confidence`

**Jurisdiction**  
`bc-jurisdiction`, `canada-outside-bc`, `out-of-canada`

**Timing**  
`tight-deadline`, `short-deadline`, `medium-deadline`, `no-deadline`

**Budget**  
`price-sensitive`, `value-focused`, `flexible-budget`

#### 3. `next_steps`
A 2–3 sentence paragraph describing the most important immediate actions (e.g., book consult, send checklists, confirm jurisdiction/conflicts).

### Final Google Sheets columns

After both Parr1 and Parr2 run, each row contains:

| Stage 1 (Parr1) | Stage 2 (Parr2 - Quiz) | Stage 2 (Parr2 - AI) |
|-----------------|------------------------|----------------------|
| first_name      | business_structure     | ai_intake_summary    |
| last_name       | business_location      | ai_flags             |
| email           | founder_count          | next_step            |
| phone           | timeline               |                      |
| support_for     | main_need              |                      |
| support_type    | score                  |                      |
| hear_about_us   | tier/segment           |                      |
| stage           |                        |                      |
| created_at      |                        |                      |

---

## Files in this repository

- **`index.html`** – Client-facing intake form (embed on website)
- **`parr1.json`** – Zapier export for Stage 1 (form → Sheets)
- **`parr2.json`** – Zapier export for Stage 2 (quiz → AI → Sheets)
- **`README.md`** – This documentation

---

## Setup instructions

### Prerequisites
- Zapier account (free or paid tier)
- Google account with Google Sheets
- ScoreApp account with a configured readiness quiz
- Web hosting for `index.html` (or embed in existing site)

### Step 1: Set up Google Sheet
1. Create a new Google Sheet named `PARR_CRM`
2. Ensure `Sheet1` exists (or update Zap configurations to match your sheet name)
3. Connect Google Sheets to your Zapier account

### Step 2: Import Parr1 Zap
1. In Zapier, go to **My Zaps** → **Create** → **Import**
2. Upload `parr1.json`
3. Configure the webhook trigger and note the webhook URL
4. Update `index.html` line 68 with your webhook URL
5. Connect your Google Sheets account
6. Map form fields to sheet columns (or use existing mappings)
7. Test and turn on the Zap

### Step 3: Deploy intake form
1. Host `index.html` on your website or embed the form HTML
2. Test by submitting the form and verifying a new row appears in `PARR_CRM`

### Step 4: Set up ScoreApp quiz
1. Create a quiz in ScoreApp that covers:
   - Business structure
   - Ownership details
   - Contract/agreement status
   - Legal priorities and timeline
2. Note your quiz/scorecard ID
3. Connect ScoreApp to your Zapier account

### Step 5: Import Parr2 Zap
1. In Zapier, import `parr2.json`
2. Configure the ScoreApp trigger with your scorecard ID
3. Update the Google Sheets lookup and update steps to match your sheet structure
4. Configure AI step:
   - Provider: Google
   - Model: Gemini 2.5 Flash (or equivalent)
   - Input fields: Map quiz answers to `Readiness` field
   - Output schema: `intake_summary`, `flags`, `next_steps`
5. Test with a sample quiz submission
6. Turn on the Zap

### Step 6: Test end-to-end
1. Fill out the intake form (`index.html`)
2. Complete the ScoreApp quiz using the same email
3. Check `PARR_CRM` sheet for a fully enriched row with AI outputs

---
