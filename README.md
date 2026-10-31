# Reputation & Feedback Intelligence Engine

An end-to-end customer feedback loop for a **multi-location car-care business**, built with three connected **n8n** workflows and a single **Airtable** base.

A visit is recorded → a feedback request goes out automatically → the customer's reply is scored by AI for **sentiment and severity** → every result is routed, stored and actioned in one dashboard. Positive reviews land in a "ready to post" queue, complaints get an AI-drafted reply, and repeat or severe negatives escalate to the location manager.

**TS ACADEMY Capstone Project — Group 77** · Saheed Oyewole

---

## The problem

Feedback from customers kept getting scattered. Nothing told the business how people actually felt, and bad reviews or repeated complaints could slip through silently — no follow-up, no escalation, no record.

## The solution

| Piece | What it does |
|---|---|
| **Workflow 01** | Branch staff submit "Mark Visit Complete"; `VIS###` / `CUS###` IDs are auto-generated and a pre-filled feedback request is emailed |
| **Workflow 02** | Public feedback form; identity fields hidden and pre-filled from the email link |
| **Workflow 03** | Webhook intake → validation → duplicate check → AI scoring → routing → Airtable + alerts |
| **Airtable base** | Single source of truth: Visits, Feedback, Responses, Alerts, Customers, Locations |
| **Airtable Interfaces** | Dashboard with overview charts, feedback grid, ready-to-post queue, drafts and an alert board |

**Stack:** n8n Cloud · Airtable (base + Interfaces) · OpenAI GPT-4o-mini · Gmail

---

## System flow

1. Branch staff fill the **Mark Visit Complete** form — no IDs are typed; Visit and Customer IDs are generated in-workflow.
2. The visit and customer records are created (`VIS001`, `CUS001`, …) and a feedback-request email is sent with a pre-filled survey link.
3. The customer submits the public feedback form.
4. The submission is posted to an intake webhook, where the AI scores sentiment, severity and confidence.
5. A routing rule decides the outcome; negative feedback gets an AI-drafted reply; repeat or severe negatives escalate to the location manager by email.
6. Everything lands in one Airtable base, surfaced in an Interfaces dashboard filterable by location.

All workflows are published and running on n8n Cloud.

---

## Workflow 01 — Visit Complete → Feedback Request

Triggered by the internal form `…/form/mark-visit-complete`.

| # | Node | Work at this stage |
|---|---|---|
| 1 | **Form Trigger** | Renders the internal form: Customer Name, Customer Email, Location, Service. No ID fields — those are generated downstream. |
| 2 | **Airtable — Search** | Lists every Visit ID so the next sequential `VIS` number can be computed. Empty-safe. |
| 3 | **Code — Compute Next Visit ID** | Scans the highest `VIS###` and emits `next_visit_id` (`VIS001`, `VIS002`, …) merged with the form data. |
| 4 | **Airtable — Search** | Lists all customers so returning customers are matched by name. |
| 5 | **Code — Compute Customer Plan** | Decides `next_customer_id`: a new number for a new customer, or their existing ID when the name already exists. |
| 6 | **Airtable — Upsert** | Upserts the Customers table on Customer Name and writes the Customer ID. |
| 7 | **Code — Merge Customer Link** | Merges `customer_record_id`, Customer ID and Visit ID into one output item. |
| 8 | **Airtable — Create** | Writes the Visit row: completion time (Africa/Lagos), linked customer and location, service, Visit ID, `n8n Trigger ID` = `VIS###`, `Feedback Request Sent = false`. |
| 9 | **Gmail — Send** | Emails the customer. The body carries a pre-filled link encoding `customer_id`, `customer_name`, `customer_email`, `visit_id` and `location`. |
| 10 | **Airtable — Update** | Marks the visit `Feedback Request Sent = true` with a timestamp. |

## Workflow 02 — Customer Feedback Form

Triggered by the public form `…/form/feedback`.

| # | Node | Work at this stage |
|---|---|---|
| 1 | **Form Trigger** | Renders the survey. Identity fields are hidden and pre-filled from the email link; "How did your visit go?" is the only required visible field. |
| 2 | **Code — Build Intake Payload** | Normalises the submission into one clean object with exactly six keys, defaulting blanks to empty strings. |
| 3 | **HTTP Request — POST** | Posts the payload as JSON to the Workflow 03 intake webhook. |

## Workflow 03 — Feedback Intake, Scoring & Routing

Triggered by `POST …/webhook/feedback-intake` (JSON body with the six keys).

| # | Node | Work at this stage |
|---|---|---|
| 1 | **Webhook Trigger** | Receives the JSON body and answers immediately, so the caller gets a fast response while the pipeline continues. |
| 2 | **Code — Validate Input** | Checks the six fields, records which are missing, and generates the anti-duplicate token `FB-<timestamp>-<random>`. |
| 3 | **Airtable — Search + Code** | Reads existing Feedback IDs and emits the next `FID###`. Empty-safe. |
| 4 | **If — Input Valid?** | Complete submissions go to history and scoring; incomplete ones go to the human-review fallback. |
| 5 | **Set + Airtable — Create** | Tags `Route = Human Review`, `Status = Needs Human Review`, stores the submission — weak or blank feedback is never scored. |
| 6 | **Airtable — Search** | Reads every prior Feedback row for this customer, so repeat complaints can be detected. |
| 7 | **Code — Summarize History** | Computes `repeat_negative`, `prior_count`, `is_duplicate`. |
| 8 | **If — Is Duplicate?** | Duplicates go to the skip branch; genuine reviews continue to AI scoring. |
| 9 | **Set — Skip Duplicate** | `Status = Skipped - Duplicate`, so re-posted submissions are never double-scored or double-alerted. |
| 10 | **OpenAI — Analyse Feedback** | GPT-4o-mini with a scoring rubric and structured outputs. Returns exactly `sentiment`, `sentiment_score`, `severity`, `severity_score`, `confidence_score`, `reason`. Temperature `0.2`. |
| 11 | **Code — Validate AI Output** | Parses and validates the reply, converts scores to numbers, sets `ai_valid = false` if unparseable — bad AI output can never corrupt routing. |
| 12 | **Code — Determine Route** | Applies the routing matrix, picks the manager email, sets `needs_draft`, `status`, `route`, `alert_reason`. |
| 13 | **Airtable — Search** | Finds the Visit whose `n8n Trigger ID` equals the submitted `visit_id` (fail-soft if missing). |
| 14 | **Airtable — Create** | Stores the scored record in Feedback with `FID###`, linking customer, location and visit. |
| 15 | **Airtable — Update** | Sets `Feedback Collection Triggered = true` on the visit. |
| 16 | **If — Needs AI Draft?** | Only negative feedback needs a draft; positive and neutral skip this branch. |
| 17 | **OpenAI — Generate AI Response** | Drafts a personalised reply naming the customer and their issue. Policy: no liability, no promises, no claims of resolution; acknowledges repeat complaints. |
| 18 | **Airtable — Search + Code** | Emits the next `RES###`. Empty-safe. |
| 19 | **Airtable — Create** | Stores the draft in Responses with `Human Review Status = Pending Review` — a human approves before anything is sent. |
| 20 | **If — Is Manager Alert?** | Checks whether the route is `Manager Alert`. |
| 21 | **Airtable — Search + Code** | Emits the next `ALT###`. Empty-safe. |
| 22 | **Airtable — Create** | Creates the escalation record: draft as Alert Message, reason, severity score, linked feedback, `Alert Status = Open`. |
| 23 | **Airtable — Search + Gmail** | Looks up the location's Manager Email and emails the manager with customer, scores, reason, original feedback and the drafted reply. Fail-soft. |

---

## Routing rules

| Condition | Route | Status |
|---|---|---|
| Missing data, or AI invalid / confidence < 75 | Human Review | Needs Human Review |
| Positive | Ready to Post | Pending Human Review |
| Negative → customer has complained before | Manager Alert | Escalated — Repeat Negative |
| Negative → severity score ≥ 70 | Manager Alert | Escalated — Severe |
| Negative (first time, moderate) | Private Queue | Pending Human Review |
| Neutral | Private Queue | Pending Human Review |

---

## Data model (single Airtable base)

| Table | Contents |
|---|---|
| **Visits** | Each completed visit: `VIS###`, `n8n Trigger ID` (the match key), feedback-request flags, `Feedback Collection Triggered` |
| **Feedback** | Every submission with AI scores, sentiment, severity, confidence, route, status, reason, linked visit/customer/location |
| **Responses** | AI-drafted reply for each negative item, pending human review |
| **Alerts** | Escalation records for repeat or severe negatives |
| **Customers** / **Locations** | Records referenced by the link fields above |

### ID scheme

- **`VIS###`** — generated in Workflow 01 from the highest existing number, stored as both the Visit ID and the `n8n Trigger ID` match key.
- **`CUS###`** — generated the same way; a returning customer (same name) keeps their existing number.
- **`FID###`** — generated in Workflow 03 for both the scored and human-review branches.
- **`RES###`** — generated whenever a negative draft reply is created.
- **`ALT###`** — generated whenever an escalation record is created.

Primary ID fields are plain text (converted from Auto Number) so the workflows can read and compute them. Every lookup is configured to still emit when the table is empty, so the first submission on an empty base works.

## Dashboard (Airtable Interfaces)

| Panel | Purpose |
|---|---|
| **Overview** | Charts for feedback volume by location and the sentiment split |
| **Every feedback** | Grid of all submissions with Customer, Location, Sentiment Score, Severity Score, Route, Status |
| **Ready to Post queue** | Copy-and-paste list of positive reviews awaiting publication |
| **Negative feedback + drafts** | Each complaint beside its AI-drafted reply and a Sent checkbox |
| **Open alerts** | Escalation board with editable Alert Status (Open / Investigating / Resolved / Archived) |
| **Location filter** | Interactive record picker that filters every panel to a single branch |

![Airtable dashboard](Airtable%20Dashboard.png)

---

## How to run / test

**Live:** staff submit the "Mark Visit Complete" form (IDs are auto-generated); the customer then completes the pre-filled feedback form. The pipeline runs automatically.

| Path | Expected outcome |
|---|---|
| Positive feedback | `Ready to Post` → dashboard queue, no alert |
| First-time moderate negative | `Private Queue` → AI draft saved as `Pending Review` |
| Severe or repeat negative | `Manager Alert` → `ALT###` record + email to the location manager |
| Missing / blank submission | `Needs Human Review` → stored but never scored |

## Configuration notes

- **Credentials:** Airtable (OAuth) and Gmail (OAuth) connected; AI uses OpenAI **GPT-4o-mini** with a structured JSON schema at temperature `0.2`.
- **Manager emails** are looked up from the Locations table. If a location has no email set, the workflow falls back to the default address — edit the Locations table to control escalation recipients.
- **Sequential IDs** use a read-then-write pattern that is empty-table safe.
- **Anti-duplicate guard:** the `FB-<timestamp>-<random>` token; the branded sequential IDs are what appear in Airtable and the dashboard.
- **Time zone:** visit timestamps use `Africa/Lagos`.

---

## Repository contents

| File | Description |
|---|---|
| `Reputation_Feedback_Engine_Documentation.pdf` / `.docx` | Full automation documentation — every node, stage by stage |
| `Explanation.docx` | Demo video script and walkthrough narration |
| `Workflow 1 - Auto-generate Visit & Customer IDs.png` | Workflow 01 canvas |
| `Workflow 2 - Customer Feedback Form.png` | Workflow 02 canvas |
| `Workflow 3A- Feedback Intake, Scoring & Routing.png` | Workflow 03 canvas (scoring and routing) |
| `Workflow 3B- Feedback Intake, Scoring & Routing.png` | Workflow 03 canvas (drafting and alerts) |
| `Airtable Dashboard.png` | Interfaces dashboard |
| `Airtable fields.png` | Base schema |
| `*_TS ACADEMY _ CAPSTONE BRIEF….pdf` | Original capstone brief |

> **Demo video:** the recorded walkthrough is ~419 MB, which exceeds GitHub's 100 MB per-file limit, so it is **not** included here. It is submitted separately via the capstone submission form. Ask if you'd like it hosted (YouTube/Drive) and linked here.

## Screenshots

| Workflow 01 | Workflow 02 |
|---|---|
| ![Workflow 1](Workflow%201%20-%20Auto-generate%20Visit%20%26%20Customer%20IDs.png) | ![Workflow 2](Workflow%202%20-%20Customer%20Feedback%20Form.png) |

| Workflow 03 — A | Workflow 03 — B |
|---|---|
| ![Workflow 3A](Workflow%203A-%20Feedback%20Intake%2C%20Scoring%20%26%20Routing.png) | ![Workflow 3B](Workflow%203B-%20Feedback%20Intake%2C%20Scoring%20%26%20Routing.png) |

---

## Limitations & next steps

- **Credentials are not in this repo** — the live n8n instance and Airtable base are private and not exported here.
- **No JSON exports included** — these workflows run on n8n Cloud; exports could be added for reproducibility.
- **Escalation is email-based** — SMS or WhatsApp would suit the Nigerian market better.
- **The "Sent" flag is manual** — moving from human approval to tracked delivery (e.g. a Gmail send node on approval) would close the loop.

## Author

**Saheed Oyewole** — Group 77
TS ACADEMY AI & Automation Capstone Project
