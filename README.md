# KeHE Spotlight → Lead Gen → Influencer Outreach

n8n workflow that automatically processes KeHE Distributors "Spotlight Brands" emails, finds decision makers at those brands, and runs personalized cold outreach for influencer marketing partnerships.

Built for **ENT Agency**.

## Architecture

Two n8n workflows working together:

1. **Main workflow** (`kehe-lead-gen-workflow.json`) — ingests KeHE emails, enriches brands, finds contacts, pushes leads + personalized email templates to Smartlead
2. **Webhook handler** (`smartlead-webhook-handler.json`) — receives Smartlead webhook events and syncs status back to Google Sheets

Smartlead owns the outreach lifecycle: sending schedule, warmup, follow-up delays, reply detection, and unsubscribe handling. The n8n workflows handle lead discovery and status tracking.

```
KeHE Email → Parse Brands → Enrich Companies → Find Decision Makers → Find & Verify Emails → Dedup → Personalized Outreach → Smartlead Campaign
                                                                                                                                       ↓
                                                                              Google Sheets ← Webhook Handler ← Smartlead (reply/category/unsub events)
```

### Main Workflow: 7 Phases, 18 Nodes

| Phase | What it does | Tools |
|-------|-------------|-------|
| 0. Secrets Fetch | Pulls all API keys from Doppler at runtime | Doppler |
| 1. Email Ingestion | Gmail trigger polls for KeHE emails, extracts brand names & descriptions | Gmail OAuth2 |
| 2. Company Enrichment | Finds brand website, enriches company data, filters by ICP | SerpAPI, Apollo.io |
| 3. Find Decision Makers | Searches LinkedIn for marketing/partnerships contacts | PhantomBuster + LinkedIn Sales Navigator |
| 4. Email Finding | Finds and verifies contact email addresses | FindyMail |
| 5. CRM & Dedup | Checks for duplicates, logs new leads | Google Sheets |
| 6. Personalized Outreach | Generates 3 personalized emails, adds lead + templates to Smartlead campaign | Smartlead.ai |

### Webhook Handler: 7 Nodes

| Event | Action |
|-------|--------|
| `EMAIL_REPLY` | Updates Google Sheets status → "replied" |
| `LEAD_CATEGORY_UPDATED` | Maps Smartlead category (Interested/Not Interested/etc.) → sheet status |
| `LEAD_UNSUBSCRIBED` | Updates Google Sheets status → "unsubscribed" |

## Email Templates

All 3 emails are generated per-contact and passed to Smartlead as custom fields. Smartlead's campaign sequence references these fields to send at the configured delays.

**Email 1 (Initial, Day 1):** KeHE Spotlight congratulations + influencer marketing pitch

**Email 2 (Follow-Up, Day 3):** Category-specific case study with concrete results

**Email 3 (Final, Day 7):** FOMO/scarcity angle with Calendly link

## Setup

### 1. Import Workflows into n8n
- Workflows → Import from File → select `kehe-lead-gen-workflow.json`
- Workflows → Import from File → select `smartlead-webhook-handler.json`
- Activate the webhook handler workflow so it listens for incoming events

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

The workflow's "Fetch Secrets from Doppler" node calls `GET /v3/configs/config/secrets/download` at the start of each execution, making all secrets available to downstream nodes.

### 3. n8n Credentials
- **Gmail OAuth2** — `marketingteam@nickient.com` (used for email trigger; can share the same Google OAuth2 app as Sheets)
- **Google Sheets OAuth2** — for Lead Tracker access (used by both workflows)

### 4. Google Sheet
Spreadsheet: [Lead Tracker](https://docs.google.com/spreadsheets/d/1tnv0RcwQ-kLCEBI8YQckNRVcaHnOeavaMDuOfufg5D4/edit)

Tab name: `Lead Tracker`

Columns:
```
brand_name | brand_description | brand_category | kehe_feature_date | contact_name | contact_title | contact_email | contact_linkedin | company_website | employee_count | outreach_status | outreach_date | last_updated
```

### 5. Smartlead Campaign Setup
1. Create a campaign in Smartlead with a 3-step email sequence:
   - **Step 1:** Use `{{email_1_subject}}` / `{{email_1_body}}` custom fields
   - **Step 2:** 3-day delay, use `{{email_2_subject}}` / `{{email_2_body}}`
   - **Step 3:** 4-day delay, use `{{email_3_subject}}` / `{{email_3_body}}`
2. Configure campaign schedule (weekdays 9am-6pm recommended)
3. Grab Campaign ID from the Smartlead dashboard URL
4. Get API key from Settings → API

### 6. Smartlead Webhook Setup
1. Activate the `smartlead-webhook-handler` workflow in n8n
2. Copy the webhook URL from the Webhook Trigger node (will look like `https://your-n8n-domain.com/webhook/smartlead-webhook`)
3. In Smartlead → Campaign Settings → Webhooks → paste the URL
4. Enable events: `EMAIL_REPLY`, `LEAD_CATEGORY_UPDATED`, `LEAD_UNSUBSCRIBED`

## Outreach Status Values

| Status | Set by | Meaning |
|--------|--------|---------|
| `new_lead` | Main workflow | Lead added to sheet, not yet sent to Smartlead |
| `outreach_sent` | Main workflow | Lead pushed to Smartlead campaign |
| `replied` | Webhook handler | Prospect replied to any email in the sequence |
| `interested` | Webhook handler | Categorized as interested in Smartlead |
| `not_interested` | Webhook handler | Categorized as not interested in Smartlead |
| `unsubscribed` | Webhook handler | Prospect unsubscribed |
| `out_of_office` | Webhook handler | Auto-reply / OOO detected |
| `wrong_contact` | Webhook handler | Wrong person categorization |
| `info_requested` | Webhook handler | Prospect requested more information |

## ICP Filter Criteria
- Must have a discoverable website
- Consumer brand (food/bev, health, beauty, pet, household)
- Prioritized titles: Influencer Marketing > Partnerships > Brand Manager > CMO > VP Marketing

## Verification Checklist
1. Import both workflows into n8n
2. Configure Google Sheets OAuth2 credential
3. Set `DOPPLER_SERVICE_TOKEN` env var in n8n
4. Activate the webhook handler workflow
5. Set up Smartlead campaign with 3-step sequence using custom fields
6. Register webhook URL in Smartlead pointing to n8n
7. Test: manually trigger main workflow with a test email → verify lead appears in Smartlead campaign
8. Test: simulate reply in Smartlead → verify webhook fires → Google Sheet updates to "replied"
