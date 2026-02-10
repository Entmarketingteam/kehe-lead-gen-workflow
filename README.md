# KeHE Spotlight → Lead Gen → Influencer Outreach

n8n workflow that automatically processes KeHE Distributors "Spotlight Brands" emails, finds decision makers at those brands, and runs personalized cold outreach for influencer marketing partnerships.

Built for **ENT Agency**.

## Workflow Overview

```
KeHE Email → Parse Brands → Enrich Companies → Find Decision Makers → Find & Verify Emails → Dedup → Personalized Outreach → Follow-Up Sequence
```

### 8 Phases, 25 Nodes

| Phase | What it does | Tools |
|-------|-------------|-------|
| 0. Secrets Fetch | Pulls all API keys from Doppler at runtime | Doppler |
| 1. Email Ingestion | Watches inbox for KeHE emails, extracts brand names & descriptions | IMAP |
| 2. Company Enrichment | Finds brand website, enriches company data, filters by ICP | SerpAPI, Apollo.io |
| 3. Find Decision Makers | Searches LinkedIn for marketing/partnerships contacts | PhantomBuster + LinkedIn Sales Navigator |
| 4. Email Finding | Finds and verifies contact email addresses | FindyMail |
| 5. CRM & Dedup | Checks for duplicates, logs new leads | Google Sheets |
| 6. Personalized Outreach | Generates 3 personalized emails, adds lead to campaign | Smartlead.ai |
| 7. Follow-Up Sequence | 3-step sequence with reply detection | Smartlead.ai, Google Sheets |

## Email Templates

**Email 1 (Initial):** KeHE Spotlight congratulations + influencer marketing pitch

**Email 2 (Follow-Up, Day 3):** Category-specific case study with concrete results

**Email 3 (Final, Day 7):** FOMO/scarcity angle with Calendly link

## Setup

### 1. Import into n8n
Workflows → Import from File → select `kehe-lead-gen-workflow.json`

### 2. Doppler Secrets Management
All API keys are fetched at runtime from Doppler. Only **one** n8n environment variable is needed:

| Variable | Source |
|----------|--------|
| `DOPPLER_SERVICE_TOKEN` | Doppler → Project → Config → Service Tokens |

All other secrets live in Doppler (`ent-agency-automation` project, `dev` config):

| Doppler Secret | Source |
|----------------|--------|
| `SERP_API_KEY` | serpapi.com |
| `APOLLO_API_KEY` | app.apollo.io → Settings → API |
| `PHANTOMBUSTER_API_KEY` | phantombuster.com → Settings |
| `PHANTOMBUSTER_LINKEDIN_AGENT_ID` | Your LinkedIn Search agent ID |
| `FINDYMAIL_API_KEY` | app.findymail.com → Settings |
| `SMARTLEAD_API_KEY` | app.smartlead.ai → Settings → API |
| `SMARTLEAD_CAMPAIGN_ID` | From your Smartlead campaign URL |
| `SMTP_FROM_EMAIL` | Your outreach sender email |
| `SMTP_REPLY_TO` | Reply-to email address |

The workflow's "Fetch Secrets from Doppler" node calls `GET /v3/configs/config/secrets/download` at the start of each execution, making all secrets available to downstream nodes.

### 3. n8n Credentials
- **IMAP Account** — `marketingteam@nickient.com` inbox
- **Google Sheets OAuth2** — for Lead Tracker access
- **SMTP Account** — optional, for direct follow-up sending

### 4. Google Sheet
Spreadsheet: [Lead Tracker](https://docs.google.com/spreadsheets/d/1tnv0RcwQ-kLCEBI8YQckNRVcaHnOeavaMDuOfufg5D4/edit)

Tab name: `Lead Tracker`

Columns:
```
brand_name | brand_description | brand_category | kehe_feature_date | contact_name | contact_title | contact_email | contact_linkedin | company_website | employee_count | outreach_status | outreach_date | last_updated
```

### 5. Smartlead Campaign Setup
1. Create a campaign in Smartlead
2. Reference custom fields in your email templates: `{{brand_name}}`, `{{contact_title}}`, `{{brand_category}}`, `{{email_1_body}}`, etc.
3. Set up 3-step subsequence with 3-day and 4-day delays
4. Grab Campaign ID from the Smartlead dashboard URL

## ICP Filter Criteria
- Must have a discoverable website
- Consumer brand (food/bev, health, beauty, pet, household)
- Prioritized titles: Influencer Marketing > Partnerships > Brand Manager > CMO > VP Marketing

## Brands Extracted Per Email (Example)
From the 2026 KeHE Summer Show Spotlight email:
- Bari Olive Company
- Yellow Yak
- 1st Phorm International
- Fire Department Coffee
- Hakubaku
- Windy City Organics (Dastony, Rawmio, Veggimins, Brothers Nuts)
- Honolulu Cookie Company
- Perfect Hydration
- Prime Bites
- Intrastate Distributors (Frostie, Kist)
- Bobelo
- Hydy Inc. (Boka, Viking Revolution, Puracy, FreshCap)
