# 🏥 ShowUp Engine — Predictive No-Show Prevention System

> An AI-powered appointment automation system built with n8n that predicts cancellation risk, sends personalised reminders using Groq LLM, and handles client confirmation via webhook — fully automated end to end.

---

## 📌 Table of Contents

- [Overview](#overview)
- [Live Demo Flow](#live-demo-flow)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Workflow Nodes](#workflow-nodes)
- [Google Sheets Structure](#google-sheets-structure)
- [Risk Scoring Engine](#risk-scoring-engine)
- [Environment Variables](#environment-variables)
- [Setup Instructions](#setup-instructions)
- [How to Test](#how-to-test)
- [Production Deployment](#production-deployment)
- [Project Structure](#project-structure)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Author](#author)

---

## Overview

No-shows cost small businesses thousands every month. This system automatically:

- Detects new bookings in real time via Google Sheets
- Scores each client's cancellation risk (0–100) using 7 behavioural signals
- Sends a personalised appointment request email immediately
- Waits dynamically based on risk level (12h / 24h / 48h before appointment)
- Checks if the client confirmed via a unique token link
- Sends an AI-generated reminder via Groq LLM if not confirmed
- Sends an instant confirmation reply when the client clicks the confirm link
- Updates booking status in Google Sheets throughout the entire journey

---

## Live Demo Flow

```
New booking added to Google Sheets
        ↓
Booking Trigger fires (every minute poll)
        ↓
Is New Booking? (is_processed check)
        ↓
Fetch Client History (filter by client_id)
        ↓
Risk Score Engine (JavaScript — 7 signals)
        ↓
Commitment Message (Gmail — appointment request + confirm link)
        ↓
Store Booking Record (Appointment_Records sheet)
        ↓
Wait Dynamic (12h / 24h / 48h before appointment)
        ↓
Status Check (read Appointment_Records by confirm_token)
        ↓
Confirmed Check (IF status === confirmed)
   ↓ FALSE
Reminder AI (Groq LLM — personalised 2-sentence reminder)
   ↓
Send Reminder Email (Gmail)
   ↓
Update Booking Status (Appointment_Records)

--- PARALLEL FLOW ---

Client clicks confirm link in email
        ↓
Webhook Confirm (GET /webhook/confirm?token=...)
        ↓
Update Confirmation Status (status=confirmed, outcome=confirmed)
        ↓
Send Confirmation Reply (Gmail — instant "See you soon" email)
```

---

## Architecture

```
FLOW 1 — Main Automation (Booking Trigger based)
┌─────────────────────────────────────────────────────────────────┐
│  Booking Trigger                                                 │
│       → Is New Booking                                          │
│       → Fetch Client History                                    │
│       → Risk Score Engine                                       │
│       → Commitment Message                                      │
│       → Store Booking Record  ──→  Appointment_Records          │
│       → Wait Dynamic                                            │
│       → Status Check          ←──  Appointment_Records          │
│       → Confirmed Check                                         │
│            FALSE → Reminder AI                                  │
│                  → Send Reminder Email                          │
│                  → Update Booking Status                        │
└─────────────────────────────────────────────────────────────────┘

FLOW 2 — Webhook Confirmation (Client click based)
┌─────────────────────────────────────────────────────────────────┐
│  Webhook Confirm (GET /webhook/confirm?token=...)                │
│       → Update Confirmation Status  ──→  Appointment_Records    │
│       → Send Confirmation Reply                                  │
└─────────────────────────────────────────────────────────────────┘

GOOGLE SHEETS DATABASE
┌──────────────────────┐
│  Client_History      │  ← read only by workflow
│  Bookings            │  ← trigger watches this sheet
│  Appointment_Records │  ← workflow writes and updates here
└──────────────────────┘
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** v2.17.7 | Workflow automation engine (self-hosted) |
| **Google Sheets** | Database for clients, bookings, and records |
| **Google Sheets Trigger** | Polls Bookings sheet every minute for new rows |
| **Gmail** | Sends all emails (commitment, reminder, confirmation) |
| **Groq API** | LLM inference for AI-generated reminder messages |
| **Llama 3.1 8B Instant** | Model used for reminder generation |
| **n8n Webhook** | Receives client confirmation click |
| **JavaScript (Code node)** | Risk scoring engine with 7 behavioural signals |
| **Luxon (DateTime)** | Dynamic wait time calculation |

---

## Workflow Nodes

| # | Node Name | Type | Purpose |
|---|---|---|---|
| 1 | Booking Trigger | Google Sheets Trigger | Polls Bookings sheet every minute for new rows |
| 2 | Is New Booking | IF | Checks if is_processed is empty to prevent duplicates |
| 3 | Fetch Client History | Google Sheets | Reads client data filtered by client_id |
| 4 | Risk Score Engine | Code (JavaScript) | Calculates risk score 0–100 from 7 signals |
| 5 | Commitment Message | Gmail | Sends appointment request email with confirm link |
| 6 | Store Booking Record | Google Sheets | Appends booking to Appointment_Records with is_processed=TRUE |
| 7 | Wait Dynamic | Wait | Pauses execution until X hours before appointment |
| 8 | Status Check | Google Sheets | Reads current status by confirm_token |
| 9 | Confirmed Check | IF | Checks if status === confirmed |
| 10 | Reminder AI | HTTP Request | Calls Groq API to generate personalised reminder |
| 11 | Send Reminder Email | Gmail | Delivers AI-generated reminder to client |
| 12 | Update Booking Status | Google Sheets | Updates status and outcome in Appointment_Records |
| 13 | Webhook Confirm | Webhook | Receives GET request when client clicks confirm link |
| 14 | Update Confirmation Status | Google Sheets | Updates status=confirmed, outcome=confirmed |
| 15 | Send Confirmation Reply | Gmail | Sends instant "See you soon" email on confirmation |

---

## Google Sheets Structure

### Sheet 1 — `Client_History`
Stores historical client data used for risk scoring.

| Column | Type | Description |
|---|---|---|
| client_id | string | Unique client identifier (e.g. CLI_001) |
| name | string | Client full name |
| phone | string | Phone number |
| total_bookings | number | Total bookings ever made |
| no_shows | number | Number of times client did not show |
| attended | number | Number of times client attended |
| cancelled | number | Number of cancellations |
| risk_avg | number | Historical average risk score |
| last_visit | date | Date of most recent visit |
| email | string | Client email address |

### Sheet 2 — `Bookings`
Trigger sheet. New rows here start the entire workflow.

| Column | Type | Description |
|---|---|---|
| booking_id | string | Unique booking ID (e.g. BKG_001) |
| client_id | string | Links to Client_History |
| appointment_date | datetime | Format: yyyy-MM-dd HH:mm |
| service_type | string | Type of appointment |
| risk_score | number | Filled by workflow |
| status | string | pending / confirmed / cancelled |
| confirm_token | string | Unique token for confirm link |
| message_sent | string | Dominant signal used |
| outcome | string | pending / confirmed / pending_followup |
| reason_for_booking | string | Client's stated reason |
| workflow_version | string | v1 |
| is_processed | string | Empty on entry, TRUE after workflow runs |

### Sheet 3 — `Appointment_Records`
Processed records written by the workflow. Separate from Bookings to prevent infinite trigger loop.

Same columns as Bookings sheet. This is where all status updates happen.

---

## Risk Scoring Engine

The Risk Score Engine is a JavaScript Code node that calculates a risk score from 0 to 100.

### Signals

| Signal | Points | Condition |
|---|---|---|
| Booked too early | +20 | Appointment is more than 30 days away |
| First-time client | +20 | total_bookings === 0 |
| Risky Monday slot | +15 | Monday before 10 AM |
| Risky Friday slot | +15 | Friday after 4 PM |
| Previous no-shows | +20 each | no_shows > 0 |
| High historical risk | +20 | risk_avg > 50 |
| Low attendance rate | +15 | attended / total_bookings < 0.8 |
| Multiple cancellations | +10 | cancelled >= 2 |

### Risk Levels

| Score | Level | Wait Before Appointment |
|---|---|---|
| 70–100 | 🔴 High | 48 hours |
| 40–69 | 🟡 Medium | 24 hours |
| 0–39 | 🟢 Low | 12 hours |

### Dominant Signals (used for AI reminder strategy)

| Signal | Meaning |
|---|---|
| `waitlist` | Client has previous no-shows |
| `motivation` | Client booked very far in advance |
| `logistics` | First-time client |
| `social_proof` | Default for low-risk clients |

---

## Environment Variables

Create a `.env` file in your n8n environment:

```
GROQ_API_KEY=your_groq_api_key_here
```

Then in n8n → Settings → Environment Variables → add:

| Key | Value |
|---|---|
| GROQ_API_KEY | your Groq API key from console.groq.com |

In the Reminder AI node the Authorization header uses:
```
Bearer {{$env.GROQ_API_KEY}}
```

> ⚠️ Never commit your actual API key to GitHub. The .gitignore file in this repo excludes .env files.

---

## Setup Instructions

### Prerequisites

- n8n self-hosted (v2.17.7 or above)
- Google account with Sheets and Gmail access
- Groq API key (free at console.groq.com)

### Step 1 — Clone the repo

```bash
git clone https://github.com/YOUR_USERNAME/showup-engine-n8n.git
cd showup-engine-n8n
```

### Step 2 — Set up Google Sheets

Create a new Google Spreadsheet with these 3 tabs:

```
Client_History
Bookings
Appointment_Records
```

Use the column structure from the Google Sheets Structure section above.

### Step 3 — Import workflow into n8n

1. Open n8n → top left **+** → Import from file
2. Select `predictive-noshow-prevention-system.json`
3. Click Import

### Step 4 — Connect credentials

Inside n8n connect:

- **Google Sheets Trigger** — Google OAuth2
- **Google Sheets** — Google OAuth2
- **Gmail** — Google OAuth2
- **Groq API** — add via Environment Variable (see above)

### Step 5 — Update sheet URLs

In each Google Sheets node update the Document URL to your own spreadsheet URL.

### Step 6 — Add environment variable

In n8n → Settings → Environment Variables → add `GROQ_API_KEY`

### Step 7 — Activate workflow

Click **Publish** in the top right of the n8n canvas.

---

## How to Test

### Test the full flow

1. Open your `Bookings` sheet
2. Add a new row with `is_processed` column left empty:

```
BKG_001 | CLI_001 | 2026-09-01 10:00 | General Consultation | (empty) | pending | (empty) | (empty) | (empty) | Routine checkup | v1 | (empty)
```

3. Wait up to 1 minute — the Booking Trigger polls every minute
4. Check your email — commitment email should arrive
5. Click the confirm link in the email
6. Browser should return `{"message":"Workflow was started"}`
7. Check `Appointment_Records` — status should update to `confirmed`
8. Another email arrives instantly — "See you soon" confirmation reply

### Test the reminder flow

1. Add a new row to Bookings (same as above)
2. Do NOT click the confirm link
3. Wait for Wait Dynamic to expire
4. Groq AI reminder email arrives with personalised message

---

## Production Deployment

Before going live make these changes:

### 1 — Update Wait Dynamic expression

Change the Wait Dynamic node from test mode to production:

```javascript
={{
DateTime
.fromFormat(
$json.appointment_date,
'yyyy-MM-dd HH:mm'
)
.minus({
hours: $json.wait_hours
})
.toISO()
}}
```

### 2 — Update webhook URL in emails

Replace `localhost:5678` with your actual n8n domain in the Commitment Message and Reminder AI nodes:

```
https://your-n8n-domain.com/webhook/confirm?token=...
```

### 3 — Add webhook authentication

In Webhook Confirm node → Authentication → Header Auth → add a secret key to prevent fake confirmations.

### 4 — Set up n8n on a server

Deploy n8n on a VPS (DigitalOcean, Render, Railway) so the workflow runs 24/7 independently.

---

## Project Structure

```
showup-engine-n8n/
│
├── predictive-noshow-prevention-system.json   ← n8n workflow export
├── .gitignore                                 ← excludes .env and secrets
└── README.md                                  ← this file
```

---

## Known Limitations

| Limitation | Detail |
|---|---|
| Wait Dynamic is test mode | Using `DateTime.now().plus({minutes:1})` for testing. Must change to production expression before going live. |
| Webhook is unauthenticated | No header auth on Webhook Confirm. Safe for local testing, must secure for production. |
| Single reminder only | System sends one reminder per booking. No retry logic for v1. |
| Localhost webhook URL | Confirm link uses localhost. Must update to real domain for production. |
| Google Sheets polling | Trigger polls every minute, not real-time. Small delay between booking and email. |

---

## Roadmap

- [ ] v2 — Multi-reminder logic (Day-before + Day-of reminders)
- [ ] v2 — SMS reminders via Twilio
- [ ] v2 — Webhook authentication with header secret
- [ ] v2 — Auto-update Client_History after appointment outcome
- [ ] v3 — Dashboard for booking analytics
- [ ] v3 — WhatsApp reminders via Twilio WhatsApp API
- [ ] v3 — Cancellation handling flow

---

## Author

**S Jagadeesh**

Built as a capstone automation project demonstrating production-style n8n workflow design, AI integration, webhook architecture, and dynamic risk scoring.

---

> Built with n8n · Groq · Google Sheets · Gmail · JavaScript
