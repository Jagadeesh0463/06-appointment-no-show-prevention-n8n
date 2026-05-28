# 🏥 ShowUp Engine — Predictive No-Show Prevention System

> An AI-powered appointment automation system that predicts cancellation risk, sends personalised reminders using Groq LLM, handles client confirmation via webhook, and triggers rebooking outreach — fully automated end to end using n8n.

---

## 📋 Table of Contents

- [The Problem](#the-problem)
- [My Solution](#my-solution)
- [Live Demo Flow](#live-demo-flow)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Workflow Nodes](#workflow-nodes)
- [Risk Scoring Engine](#risk-scoring-engine)
- [Google Sheets Structure](#google-sheets-structure)
- [Environment Variables](#environment-variables)
- [Setup Instructions](#setup-instructions)
- [How to Test](#how-to-test)
- [Production Deployment](#production-deployment)
- [Project Structure](#project-structure)
- [Known Limitations](#known-limitations)
- [Future Roadmap](#future-roadmap)
- [Author](#author)

---

## The Problem

**Appointment no-shows are costing small businesses thousands every month.**

According to MGMA 2025 data, 73% of medical practices reported no-show rates staying the same or worsening. The core reasons:

- Clients forget their appointments
- No personalised follow-up happens
- Businesses rely on manual reminders that are inconsistent
- There is no system to detect who is most likely to not show up
- When a client does not confirm, there is no automatic rebooking outreach

**This affects:**
- Medical clinics and GP practices
- Personal trainers and fitness coaches
- Tutors and educational consultants
- Salons and beauty professionals
- Business coaches and therapists

**The cost:**
- Lost revenue from empty appointment slots
- Wasted staff time preparing for clients who never arrive
- No way to fill last-minute cancellations proactively

---

## My Solution

**ShowUp Engine** is a fully automated no-show prevention workflow built on n8n that:

| Problem | Solution |
|---|---|
| Clients forget appointments | Sends immediate booking confirmation email with confirm link |
| No way to detect high-risk clients | Risk Scoring Engine with 7 behavioural signals (0–100 score) |
| One-size-fits-all reminders | Groq LLM generates personalised AI reminder per client |
| No confirmation tracking | Token-based webhook confirmation system |
| Reminder sent too early or too late | Dynamic wait time — 12h, 24h, or 48h based on risk level |
| No second reminder | Second check 2 hours after first reminder |
| No rebooking outreach | Automatic rebooking email if client never confirms |
| Manual status updates | Google Sheets updated automatically throughout entire journey |

---

## Live Demo Flow

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FLOW 1 — BOOKING TRIGGERED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

New row added to Bookings sheet
          ↓
Booking Trigger fires (polls every minute)
          ↓
Is New Booking? → checks is_processed column
          ↓ YES
Fetch Client History → filtered by client_id
          ↓
Risk Score Engine → calculates 0–100 risk score
          ↓
Commitment Message → sends appointment request email
          + confirm link with token, email, name, date
          ↓
Store Booking Record → writes to Appointment_Records
          + is_processed = TRUE (prevents loop)
          ↓
Wait Dynamic → pauses until X hours before appointment
          (48h = high risk | 24h = medium | 12h = low)
          ↓
Status Check → reads Appointment_Records by confirm_token
          ↓
Confirmed Check
          ↓ FALSE (not confirmed yet)
Reminder AI → Groq LLM generates personalised 2-sentence reminder
          ↓
Send Reminder Email → delivers AI reminder to client
          ↓
Wait 2 Hours → pauses for 2 more hours
          ↓
Status Check 2 → reads status again
          ↓
Confirmed Check 2
          ↓ FALSE (still not confirmed)
Send Rebooking Link → asks client to reschedule
          ↓
Update Booking Status → status = no_response

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FLOW 2 — CLIENT CLICKS CONFIRM LINK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Client clicks confirm link in email
          ↓
Webhook Confirm fires (GET /webhook/confirm?token=...)
          ↓
Update Confirmation Status → status = confirmed, outcome = confirmed
          ↓
Send Confirmation Reply → instant "Your appointment is confirmed" email
          ↓
Update Booking Status → final status written to sheet
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  FLOW 1 — Main Automation (Google Sheets Trigger)                    │
│                                                                      │
│  Bookings Sheet                                                      │
│       ↓                                                              │
│  Booking Trigger → Is New Booking → Fetch Client History            │
│       ↓                                                              │
│  Risk Score Engine (JS — 7 signals, score 0–100)                    │
│       ↓                                                              │
│  Commitment Message (Gmail — confirm link with token)               │
│       ↓                                                              │
│  Store Booking Record → Appointment_Records (is_processed=TRUE)     │
│       ↓                                                              │
│  Wait Dynamic (48h / 24h / 12h before appointment)                  │
│       ↓                                                              │
│  Status Check → Confirmed Check                                     │
│       ↓ FALSE                                                        │
│  Reminder AI (Groq LLM) → Send Reminder Email                      │
│       ↓                                                              │
│  Wait 2 Hours → Status Check 2 → Confirmed Check 2                 │
│       ↓ FALSE                                                        │
│  Send Rebooking Link → Update Booking Status                        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  FLOW 2 — Webhook Confirmation (Client Click Triggered)              │
│                                                                      │
│  Client clicks confirm link in email                                 │
│       ↓                                                              │
│  Webhook Confirm (GET /webhook/confirm?token=...)                   │
│       ↓                                                              │
│  Update Confirmation Status → Appointment_Records                   │
│       ↓                                                              │
│  Send Confirmation Reply (Gmail — instant confirmation)             │
│       ↓                                                              │
│  Update Booking Status                                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  GOOGLE SHEETS DATABASE                                              │
│                                                                      │
│  Client_History      ← read only — client behavioural data         │
│  Bookings            ← trigger watches this sheet for new rows      │
│  Appointment_Records ← workflow writes, updates, tracks here        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Tool | Version | Purpose |
|---|---|---|
| **n8n** | 2.17.7 | Workflow automation engine (self-hosted) |
| **Google Sheets** | — | Database for clients, bookings, records |
| **Google Sheets Trigger** | — | Detects new bookings every minute |
| **Gmail** | — | Sends all emails (confirmation, reminder, rebooking) |
| **Groq API** | — | LLM inference for AI-generated reminders |
| **Llama 3.1 8B Instant** | — | Model used for reminder generation |
| **n8n Webhook** | — | Receives client confirmation click |
| **JavaScript** | ES2020 | Risk scoring engine logic |
| **Luxon (DateTime)** | — | Dynamic wait time calculation |

---

## Workflow Nodes

| # | Node Name | Type | Purpose |
|---|---|---|---|
| 1 | Booking Trigger | Google Sheets Trigger | Polls Bookings sheet every minute |
| 2 | Is New Booking | IF | Prevents duplicate processing via is_processed |
| 3 | Fetch Client History | Google Sheets | Reads client data filtered by client_id |
| 4 | Risk Score Engine | Code (JavaScript) | Calculates risk score 0–100 from 7 signals |
| 5 | Commitment Message | Gmail | Sends appointment request with token-based confirm link |
| 6 | Store Booking Record | Google Sheets | Appends to Appointment_Records, sets is_processed=TRUE |
| 7 | Wait Dynamic | Wait | Waits until 12h / 24h / 48h before appointment |
| 8 | Status Check | Google Sheets | Reads current status by confirm_token |
| 9 | Confirmed Check | IF | Routes confirmed vs unconfirmed clients |
| 10 | Reminder AI | HTTP Request | Calls Groq API for personalised reminder |
| 11 | Send Reminder Email | Gmail | Delivers AI-generated reminder |
| 12 | Wait 2 Hours | Wait | Waits 2 more hours before final check |
| 13 | Status Check 2 | Google Sheets | Second status check after reminder |
| 14 | Confirmed Check 2 | IF | Final confirmed vs no-response routing |
| 15 | Send Rebooking Link | Gmail | Sends reschedule request to unconfirmed clients |
| 16 | Update Booking Status | Google Sheets | Final status update in Appointment_Records |
| 17 | Webhook Confirm | Webhook | Fires when client clicks confirm link |
| 18 | Update Confirmation Status | Google Sheets | Updates status=confirmed in sheet |
| 19 | Send Confirmation Reply | Gmail | Instant confirmation email to client |

---

## Risk Scoring Engine

A custom JavaScript node that calculates a risk score from 0 to 100 based on 7 behavioural signals.

### Signals

| Signal | Points | Trigger Condition |
|---|---|---|
| Booked too early | +20 | Appointment more than 30 days away |
| First-time client | +20 | total_bookings === 0 |
| Risky Monday slot | +15 | Monday before 10:00 AM |
| Risky Friday slot | +15 | Friday after 4:00 PM |
| Previous no-shows | +20 each | no_shows > 0 |
| High historical risk | +20 | risk_avg > 50 |
| Low attendance rate | +15 | attended / total_bookings < 0.8 |
| Multiple cancellations | +10 | cancelled >= 2 |

### Risk Levels and Wait Times

| Score | Risk Level | Wait Before Appointment | Strategy |
|---|---|---|---|
| 70 – 100 | 🔴 High | 48 hours | Waitlist pressure messaging |
| 40 – 69 | 🟡 Medium | 24 hours | Motivation-based messaging |
| 0 – 39 | 🟢 Low | 12 hours | Social proof messaging |

### Dominant Signals

| Signal | Meaning | Reminder Strategy |
|---|---|---|
| `waitlist` | Client has previous no-shows | Urgency — others want this slot |
| `motivation` | Booked very far in advance | Remind why they booked |
| `logistics` | First-time client | Help them prepare and find the place |
| `social_proof` | Low-risk regular client | Warm friendly reminder |

---

## Google Sheets Structure

### Sheet 1 — `Client_History`
Read-only by workflow. Stores historical client behaviour.

| Column | Type | Description |
|---|---|---|
| client_id | string | Unique client identifier e.g. CLI_001 |
| name | string | Client full name |
| phone | string | Phone number |
| total_bookings | number | Total bookings ever made |
| no_shows | number | Number of no-shows |
| attended | number | Number of attended appointments |
| cancelled | number | Number of cancellations |
| risk_avg | number | Historical average risk score |
| last_visit | date | Date of most recent visit |
| email | string | Client email address |

### Sheet 2 — `Bookings`
Trigger sheet. Add new rows here to start the workflow.

| Column | Type | Description |
|---|---|---|
| booking_id | string | Unique booking ID e.g. BKG_001 |
| client_id | string | Links to Client_History |
| appointment_date | datetime | Format: yyyy-MM-dd HH:mm |
| service_type | string | Type of appointment |
| risk_score | number | Filled by workflow after processing |
| status | string | pending / confirmed / no_response |
| confirm_token | string | Unique token for confirm link |
| message_sent | string | Dominant signal used for reminder |
| outcome | string | pending / confirmed / pending_followup |
| reason_for_booking | string | Client stated reason |
| workflow_version | string | v1 |
| is_processed | string | Empty on entry — TRUE after workflow runs |

### Sheet 3 — `Appointment_Records`
Written to by workflow. Separate from Bookings to prevent infinite trigger loop.

Same 12 columns as Bookings. All status updates happen here throughout the client journey.

> **Why separate sheets?**
> The Booking Trigger watches the Bookings sheet. If Store Booking Record wrote back to the same sheet it would re-trigger the workflow on every new row it writes — creating an infinite loop. Writing to a separate Appointment_Records sheet completely solves this.

---

## Environment Variables

Create these in n8n → Settings → Environment Variables:

| Key | Description |
|---|---|
| `GROQ_API_KEY` | Your Groq API key from console.groq.com |

In the Reminder AI node the Authorization header uses:
```
Bearer {{$env.GROQ_API_KEY}}
```

> ⚠️ Never hardcode your API key in the workflow JSON. The .gitignore in this repo excludes .env files. Always use environment variables.

---

## Setup Instructions

### Prerequisites

- n8n self-hosted v2.17.7 or above
- Google account with Sheets and Gmail access
- Groq API key — free at console.groq.com

### Step 1 — Clone the repo

```bash
git clone https://github.com/YOUR_USERNAME/showup-engine-n8n.git
cd showup-engine-n8n
```

### Step 2 — Set up Google Sheets

Create a new Google Spreadsheet with exactly 3 tabs:

```
Client_History
Bookings
Appointment_Records
```

Use the column structure from the Google Sheets Structure section above. Add at least 2 test clients to Client_History.

### Step 3 — Import workflow into n8n

1. Open n8n
2. Click **+** top left → **Import from file**
3. Select `predictive-noshow-prevention-system.json`
4. Click **Import**

### Step 4 — Replace placeholders

In the imported workflow find and replace all placeholders:

```
YOUR_GOOGLE_SHEET_URL          → your spreadsheet URL
YOUR_BOOKINGS_SHEET_URL        → Bookings tab URL
YOUR_CLIENT_HISTORY_SHEET_URL  → Client_History tab URL
YOUR_N8N_DOMAIN                → your n8n domain e.g. localhost:5678
YOUR_SHEETS_CREDENTIAL_ID      → your Google Sheets credential
YOUR_TRIGGER_CREDENTIAL_ID     → your Google Sheets Trigger credential
YOUR_GMAIL_CREDENTIAL_ID       → your Gmail credential
```

### Step 5 — Connect credentials

Inside n8n connect:

- **Google Sheets Trigger** → Google OAuth2
- **Google Sheets** → Google OAuth2
- **Gmail** → Google OAuth2

### Step 6 — Add environment variable

n8n → Settings → Environment Variables → Add:

```
Key:   GROQ_API_KEY
Value: your_groq_api_key_here
```

### Step 7 — Publish and activate

Click **Publish** in the top right of the n8n canvas to make the workflow active and register the webhook URL.

---

## How to Test

### Full confirmation flow test

1. Clean the Bookings and Appointment_Records sheets — keep only headers
2. Add a test row to Bookings sheet with `is_processed` column empty:

```
BKG_001 | CLI_001 | 2026-09-01 10:00 | General Consultation | (empty) | pending | (empty) | (empty) | (empty) | Routine checkup | v1 | (empty)
```

3. Wait up to 1 minute — commitment email arrives
4. Click the confirm link in the email immediately
5. Browser shows: `Appointment confirmation received successfully.`
6. Check Appointment_Records — status updates to `confirmed`
7. Confirmation reply email arrives instantly

### No-response / reminder flow test

1. Add a test row to Bookings sheet
2. Do NOT click the confirm link
3. Wait for Wait Dynamic to expire
4. AI-generated reminder email arrives
5. Wait 2 more hours (or test time)
6. If still not confirmed — rebooking email arrives
7. Appointment_Records updates to `no_response`

---

## Production Deployment

Before going live make these 3 changes:

### 1 — Update Wait Dynamic to production expression

Replace test expression with:

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

### 2 — Update all webhook URLs

Replace `localhost:5678` with your actual n8n domain in:
- Commitment Message confirm link
- Reminder AI prompt confirm link

### 3 — Add webhook authentication

In Webhook Confirm node → Authentication → Header Auth → add a secret key to prevent fake confirmations.

### 4 — Deploy n8n on a server

Host n8n on a VPS (DigitalOcean, Railway, Render) so the workflow runs 24/7 independently of your local machine.

---

## Project Structure

```
showup-engine-n8n/
│
├── predictive-noshow-prevention-system.json   ← n8n workflow export
├── .gitignore                                 ← excludes secrets and env files
└── README.md                                  ← this file
```

---

## Known Limitations

| Limitation | Detail | Fix in v2 |
|---|---|---|
| Wait Dynamic is in test mode | Uses `DateTime.now().plus()` for testing. Must change to production expression before going live. | Change before deployment |
| Webhook unauthenticated | No header auth on Webhook Confirm. Safe locally, insecure in production. | Add Header Auth in production |
| Single AI reminder only | One Groq reminder per booking. No multi-stage AI variation. | v2 roadmap |
| Google Sheets polling | Trigger polls every minute — not real-time. Small delay between booking and email. | Use Google Apps Script webhook for instant trigger in v2 |
| No retry on email failure | If Gmail send fails, workflow does not retry. | v2 roadmap |
| localhost webhook in emails | Confirm links use localhost — only works on same machine. | Change to production domain before deployment |

---

## Future Roadmap

### v2 — Smarter Reminders
- [ ] Multi-stage AI reminders — Day before + 2 hours before
- [ ] SMS reminders via Twilio
- [ ] WhatsApp reminders via Twilio WhatsApp API
- [ ] Webhook authentication with header secret key

### v2 — Smarter Tracking
- [ ] Auto-update Client_History after appointment outcome
- [ ] Track attendance rate per service type
- [ ] Risk score decay for clients who start attending

### v3 — Business Intelligence
- [ ] Dashboard for booking analytics
- [ ] No-show rate by day of week and time slot
- [ ] Revenue impact calculator
- [ ] Cancellation pattern detection

### v3 — Advanced Automation
- [ ] Google Calendar integration for real-time booking trigger
- [ ] Automatic waitlist filling when cancellation detected
- [ ] Slack/Teams notifications for clinic staff on high-risk bookings
- [ ] Multi-language reminder support

---

## Author

**S Jagadeesh**

Built as a capstone automation project demonstrating production-style n8n workflow design, AI integration with Groq LLM, webhook architecture, token-based confirmation, dynamic risk scoring, and multi-stage reminder logic.

---

## Skills Demonstrated

```
✅ n8n workflow architecture (multi-flow, webhook + trigger)
✅ JavaScript risk scoring engine with 7 behavioural signals
✅ Groq LLM API integration for personalised content generation
✅ Google Sheets as a structured database with read/write separation
✅ Token-based webhook confirmation system
✅ Dynamic wait time logic using Luxon DateTime
✅ Infinite loop prevention via sheet separation
✅ Gmail automation for transactional emails
✅ GitHub-ready secure JSON export with env variable pattern
✅ Production deployment considerations documented
```

---

> Built with n8n · Groq · Llama 3.1 · Google Sheets · Gmail · JavaScript · Luxon
