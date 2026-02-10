# KeHE Lead Gen Workflow — Automation Audit & Planning Document

**Prepared for:** ENT Agency
**Workflow Platform:** n8n (self-hosted or cloud)
**Last Updated:** 2026-02-10

---

## Table of Contents

1. [End-to-End Pipeline Summary](#end-to-end-pipeline-summary)
2. [What Is Currently Automated](#what-is-currently-automated)
3. [What Is NOT Automated (Gaps)](#what-is-not-automated-gaps)
4. [External Services & API Dependencies](#external-services--api-dependencies)
5. [Data Flow Diagram](#data-flow-diagram)
6. [Node-by-Node Breakdown](#node-by-node-breakdown)
7. [Known Limitations & Fragile Points](#known-limitations--fragile-points)
8. [Recommendations for Full End-to-End Automation](#recommendations-for-full-end-to-end-automation)
9. [Process Improvement Opportunities](#process-improvement-opportunities)
10. [One-Time Manual Setup Checklist](#one-time-manual-setup-checklist)

---

## End-to-End Pipeline Summary

The workflow converts KeHE Distributors "Spotlight Brands" email announcements into qualified, personalized influencer marketing outreach — automatically.

```
┌─────────────┐    ┌──────────────┐    ┌────────────────┐    ┌──────────────────┐
│ KeHE Email  │───>│ Extract      │───>│ Google Search + │───>│ ICP Filter       │
│ (IMAP)      │    │ Brands       │    │ Apollo Enrich   │    │ (Has website?)   │
└─────────────┘    └──────────────┘    └────────────────┘    └──────────────────┘
                                                                     │
                              ┌───────────────────────────────────────┘
                              v
┌──────────────────┐    ┌──────────────┐    ┌────────────────┐    ┌─────────────┐
│ PhantomBuster    │───>│ Rank & Limit │───>│ FindyMail Find │───>│ FindyMail   │
│ LinkedIn Search  │    │ Top 2-3      │    │ + Verify Email │    │ Verify      │
└──────────────────┘    └──────────────┘    └────────────────┘    └─────────────┘
                                                                        │
                              ┌──────────────────────────────────────────┘
                              v
┌──────────────────┐    ┌──────────────┐    ┌────────────────┐    ┌─────────────┐
│ Dedup via        │───>│ Add to       │───>│ Generate 3     │───>│ Add Lead to │
│ Google Sheets    │    │ Google Sheet  │    │ Email Templates│    │ Smartlead   │
└──────────────────┘    └──────────────┘    └────────────────┘    └─────────────┘
                                                                        │
                              ┌──────────────────────────────────────────┘
                              v
┌──────────────────┐    ┌──────────────┐    ┌────────────────┐    ┌─────────────┐
│ Update Sheet:    │───>│ Wait 3 Days  │───>│ Check Reply    │───>│ Follow-Up   │
│ "outreach_sent"  │    │              │    │ (PLACEHOLDER)  │    │ Sequence    │
└──────────────────┘    └──────────────┘    └────────────────┘    └─────────────┘
```

**7 Phases / 24+ Nodes / 8 External APIs**

---

## What Is Currently Automated

| Phase | Component | Automation Status | How It Works |
|-------|-----------|:-----------------:|--------------|
| **1** | Email ingestion | AUTOMATED | IMAP trigger watches inbox 24/7 for `@kehe.com` / `@marketing.kehe.com` senders |
| **1** | Brand extraction | AUTOMATED | JavaScript parses HTML for bold orange (`#f07327`) headers + descriptions |
| **1** | Brand categorization | AUTOMATED | Keyword matching assigns: Food & Bev, Health & Wellness, Health & Beauty, Pet, Household |
| **2** | Website discovery | AUTOMATED | SerpAPI Google search: `{brand_name} brand official site` |
| **2** | Company enrichment | AUTOMATED | Apollo.io API enriches domain → industry, employee count, HQ location |
| **2** | ICP qualification | AUTOMATED | Filters out brands without discoverable websites or Apollo data |
| **3** | Decision maker search | AUTOMATED | PhantomBuster launches LinkedIn Sales Navigator search (10 results/brand) |
| **3** | Contact ranking | AUTOMATED | Title-priority scoring algorithm (Influencer=10 → Generic Marketing=1), top 3 selected |
| **4** | Email discovery | AUTOMATED | FindyMail API: first name + last name + domain → email address |
| **4** | Email verification | AUTOMATED | FindyMail verification API: valid / invalid / catch-all status |
| **4** | Invalid email filtering | AUTOMATED | Conditional node drops invalid emails, keeps valid + catch-all |
| **5** | Duplicate checking | AUTOMATED | Google Sheets lookup by `contact_email` column |
| **5** | Lead logging | AUTOMATED | Appends new lead row with all fields + `outreach_status: new_lead` |
| **6** | Email template generation | AUTOMATED | JavaScript generates 3 personalized templates with role-based openers + category-specific case studies |
| **6** | Campaign enrollment | AUTOMATED | Smartlead API adds lead + all 3 email templates as custom fields |
| **6** | Status tracking | AUTOMATED | Google Sheets updated to `outreach_sent` with timestamp |
| **7** | Wait timers | AUTOMATED | n8n wait nodes: 3-day and 4-day delays between steps |
| **7** | Status updates (replied/complete) | AUTOMATED | Google Sheets updated based on reply detection outcome |

**Bottom line: Phases 1–6 are fully automated. Phase 7 is structurally automated but has a critical placeholder.**

---

## What Is NOT Automated (Gaps)

### CRITICAL: Reply Detection is a Placeholder

The two reply-check nodes (`Check for Reply (Before FU1)` and `Check for Reply (Before FU2)`) contain **placeholder code** that always returns `has_replied: false`.

```javascript
// Current code (PLACEHOLDER — always sends follow-ups):
return [{ json: { ...emailData, has_replied: false, check_timestamp: new Date().toISOString() } }];
```

**Impact:** Every lead receives all 3 emails regardless of whether they replied. This can damage sender reputation and annoy prospects who already responded.

**Fix required:** Replace with actual Smartlead API call:
```
GET https://server.smartlead.ai/api/v1/campaigns/{CAMPAIGN_ID}/leads?api_key={KEY}&email={LEAD_EMAIL}
→ Check if lead_status === 'REPLIED'
```

### Gap Summary Table

| Gap | Severity | What's Missing | Impact |
|-----|:--------:|----------------|--------|
| Reply detection (Phase 7) | **CRITICAL** | Smartlead API call to check `lead_status` | Follow-ups sent even after reply |
| Error handling / retries | **HIGH** | No retry logic on any API call | Single API timeout kills the whole run for that brand |
| Notifications | **MEDIUM** | No Slack/email/webhook alerts on failures or completions | Team has no visibility into workflow health |
| PhantomBuster polling | **MEDIUM** | LinkedIn search is fire-and-forget — no result polling | May not wait long enough for PhantomBuster agent to finish |
| Logging / observability | **MEDIUM** | No execution logging beyond n8n's built-in history | Hard to diagnose issues at scale |
| Rate limiting | **LOW** | No explicit throttling between API calls | Could hit API rate limits with large email batches |
| Fallback email sending | **LOW** | SMTP fallback node is disabled | If Smartlead fails, no backup sending path |
| Multi-email batching | **LOW** | KeHE emails processed one at a time | Not an issue now, but batch-processing could be faster |

---

## External Services & API Dependencies

| Service | What It Does | API Key Variable | Monthly Cost Range | Failure Impact |
|---------|-------------|------------------|-------------------|----------------|
| **IMAP (Email)** | Triggers workflow on new KeHE emails | Credential-based | Free (email account) | Workflow never starts |
| **SerpAPI** | Google search to find brand websites | `SERP_API_KEY` | $50–$100 (100–5000 searches) | Can't find brand websites |
| **Apollo.io** | Company enrichment (industry, size, HQ) | `APOLLO_API_KEY` | Free tier: 10K credits/mo | No company data → ICP filter fails |
| **PhantomBuster** | LinkedIn Sales Navigator search | `PHANTOMBUSTER_API_KEY` | $56–$128/mo | Can't find decision makers |
| **FindyMail** | Email discovery + verification | `FINDYMAIL_API_KEY` | $49–$99/mo | Can't find/verify contact emails |
| **Google Sheets** | CRM, dedup, status tracking | OAuth2 credential | Free | No lead tracking, dedup breaks |
| **Smartlead.ai** | Email sending, warmup, campaign management | `SMARTLEAD_API_KEY` | $39–$94/mo | Outreach emails don't send |
| **SMTP** | Fallback email sending (disabled) | Credential-based | Varies | Currently disabled |

**Total estimated monthly API spend: ~$200–$420/mo** (depending on volume and plan tiers)

---

## Data Flow Diagram

### Input → Processing → Output

```
INPUT:                    KeHE Spotlight Email (HTML)
                                    │
                                    ▼
EXTRACTION:               Brand Name + Description + Category
                          (per brand in email, typically 10-15 brands)
                                    │
                                    ▼
ENRICHMENT:               Website URL (SerpAPI)
                          + Industry, Employee Count, HQ (Apollo.io)
                                    │
                                    ▼
QUALIFICATION:            ICP Filter: Must have website + Apollo data
                          ❌ ~20-30% of brands filtered out here
                                    │
                                    ▼
CONTACT DISCOVERY:        LinkedIn search → 10 results/brand
                          → Ranked by title score → Top 3 selected
                          (yields 2-3 contacts per qualified brand)
                                    │
                                    ▼
EMAIL DISCOVERY:          FindyMail: Name + Domain → Email
                          + Verification (valid / invalid / catch-all)
                          ❌ ~15-25% of contacts have no findable email
                                    │
                                    ▼
DEDUPLICATION:            Google Sheets lookup
                          ❌ Previously contacted leads skipped
                                    │
                                    ▼
OUTPUT:                   New row in Google Sheets (Lead Tracker)
                          + 3 personalized email templates
                          + Lead added to Smartlead campaign
                          + 7-day follow-up sequence initiated
```

### Expected Conversion Funnel (Per KeHE Email)

```
~12 brands per email
  → ~9 pass ICP filter (have website + Apollo data)
    → ~25 contacts found (2-3 per brand)
      → ~18 have verifiable emails
        → ~15 are new (not duplicates)
          → 15 leads enrolled in Smartlead campaign
            → 3 emails each over 7 days = ~45 emails sent
```

---

## Node-by-Node Breakdown

### Phase 1: Email Ingestion (2 nodes)

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 1 | `KeHE Email Trigger (IMAP)` | IMAP Trigger | Watches inbox for unseen emails from `@kehe.com` or `@marketing.kehe.com` |
| 2 | `Extract Brands from Email` | JavaScript Code | Parses HTML for orange `#f07327` bold headers; extracts brand names, descriptions; auto-categorizes by keyword matching |

### Phase 2: Company Enrichment (4 nodes)

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 3 | `Process Each Brand` | Split in Batches | Processes brands one at a time (batch size: 1) |
| 4 | `Google Search - Find Brand Website` | HTTP Request (SerpAPI) | Searches `{brand} brand official site`, returns top 3 results |
| 5 | `Apollo.io - Enrich Company` | HTTP Request (Apollo) | Enriches domain → website, industry, employee count, city, state |
| 6 | `ICP Filter - Qualified Brand?` | IF Conditional | Requires: website URL exists AND organization data returned |

### Phase 3: Find Decision Makers (2 nodes)

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 7 | `LinkedIn - Find Decision Makers` | HTTP Request (PhantomBuster) | Launches LinkedIn Sales Nav search for marketing/partnerships titles |
| 8 | `Limit to Top 2-3 Contacts` | JavaScript Code | Scores contacts by title priority (Influencer=10 ... Marketing=1), returns top 3 |

### Phase 4: Email Finding & Verification (3 nodes)

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 9 | `FindyMail - Find Email` | HTTP Request | Submits first_name + last_name + domain, returns email address |
| 10 | `FindyMail - Verify Email` | HTTP Request | Verifies email deliverability status (valid/invalid/catch-all) |
| 11 | `Filter Valid Emails Only` | IF Conditional | Drops invalid emails; keeps valid + catch-all |

### Phase 5: CRM & Deduplication (3 nodes)

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 12 | `Check Google Sheets for Duplicates` | Google Sheets Read | Looks up `contact_email` in Lead Tracker sheet |
| 13 | `IF Not Duplicate` | IF Conditional | Proceeds only if email not found in sheet |
| 14 | `Add Lead to Google Sheets` | Google Sheets Append | Adds full lead record with `outreach_status: new_lead` |

### Phase 6: Personalized Outreach (3 nodes)

| # | Node Name | Type | What It Does |
|---|-----------|------|-------------|
| 15 | `Generate Personalized Emails` | JavaScript Code | Builds 3 email templates: Initial hook, Case study, FOMO/scarcity |
| 16 | `Add Lead to Smartlead Campaign` | HTTP Request (Smartlead) | Enrolls lead + all 3 templates as custom fields in Smartlead campaign |
| 17 | `Send Outreach Email (SMTP Fallback)` | Email Send | **DISABLED** — fallback if Smartlead is down |

### Phase 7: Follow-Up Sequence (9 nodes)

| # | Node Name | Type | Status | What It Does |
|---|-----------|------|--------|-------------|
| 18 | `Update Status - Outreach Sent` | Google Sheets Update | Working | Sets `outreach_status: outreach_sent` |
| 19 | `Wait 3 Days` | Wait | Working | Pauses execution for 3 days |
| 20 | `Check for Reply (Before FU1)` | JavaScript Code | **PLACEHOLDER** | Should check Smartlead API for reply — currently hardcoded `false` |
| 21 | `IF No Reply (Check 1)` | IF Conditional | Working | Routes based on `has_replied` boolean |
| 22 | `Send Follow-Up #1 (Case Study)` | Email Send (SMTP) | Working | Sends email template #2 |
| 23 | `Update Status - Replied (Check 1)` | Google Sheets Update | Working | Sets `outreach_status: replied` |
| 24 | `Wait 4 Days` | Wait | Working | Pauses execution for 4 more days |
| 25 | `Check for Reply (Before FU2)` | JavaScript Code | **PLACEHOLDER** | Same placeholder issue as node 20 |
| 26 | `IF No Reply (Check 2)` | IF Conditional | Working | Routes based on `has_replied` boolean |
| 27 | `Send Follow-Up #2 (FOMO/Scarcity)` | Email Send (SMTP) | Working | Sends email template #3 |
| 28 | `Update Status - Replied (Check 2)` | Google Sheets Update | Working | Sets `outreach_status: replied` |
| 29 | `Update Final Status - Sequence Complete` | Google Sheets Update | Working | Sets `outreach_status: sequence_complete` |

---

## Known Limitations & Fragile Points

### 1. HTML Parsing is Brittle
The brand extraction regex depends on KeHE's exact email HTML structure:
```javascript
/<b>([^<]{2,60})<\/b><\/span><\/p>[\s\S]*?<span style="font-size:16px">([^<]{20,500})<\/span>/gi
```
If KeHE changes their email template format (colors, structure, font sizes), extraction will silently fail. There is a plaintext fallback, but it's less reliable.

### 2. PhantomBuster is Fire-and-Forget
The LinkedIn search node launches a PhantomBuster agent but doesn't poll for completion. If the agent takes longer than the HTTP timeout (60s), results may be incomplete or empty. PhantomBuster agents often need 30-120 seconds to run.

### 3. Follow-Up Emails Use SMTP, Not Smartlead
The initial outreach goes through Smartlead (with proper warmup and deliverability features), but follow-ups #1 and #2 are sent via raw SMTP. This means:
- No warmup protection on follow-ups
- No unified reply tracking in Smartlead
- Different sending infrastructure for the same sequence

### 4. No Error Recovery
If any API call fails (SerpAPI, Apollo, PhantomBuster, FindyMail, Smartlead), the entire pipeline for that brand stops. There are no:
- Retry mechanisms
- Error catch nodes
- Dead letter queues
- Partial failure recovery

### 5. Google Sheets as CRM
Google Sheets works for low volume but has limitations:
- No concurrent write protection (race conditions at scale)
- 10M cell limit
- No relational data (brand ↔ contacts are flat rows)
- No built-in dashboards or reporting

---

## Recommendations for Full End-to-End Automation

### Priority 1 — Fix Critical Gap (Reply Detection)

**What:** Replace placeholder code in `Check for Reply (Before FU1)` and `Check for Reply (Before FU2)` with actual Smartlead API calls.

**API endpoint:**
```
GET https://server.smartlead.ai/api/v1/campaigns/{CAMPAIGN_ID}/leads?api_key={API_KEY}&email={LEAD_EMAIL}
```
Check response for `lead_status === 'REPLIED'`.

**Alternative:** Let Smartlead handle the entire 3-step sequence natively using its built-in subsequence feature. This would simplify Phase 7 to just status tracking and eliminate the SMTP follow-up issue entirely.

### Priority 2 — Add Error Handling

Add n8n Error Trigger + catch nodes to:
- Retry failed API calls (1-2 retries with backoff)
- Log failures to a separate Google Sheet tab ("Errors")
- Send Slack notification on workflow failures

### Priority 3 — Add Notifications & Observability

| Event | Notification Channel | Suggested Tool |
|-------|---------------------|----------------|
| Workflow triggered (new KeHE email) | Slack | n8n Slack node |
| Brands extracted (count) | Slack | n8n Slack node |
| Lead enrolled in campaign | Slack | n8n Slack node |
| Prospect replied | Slack + Email | Smartlead webhook → n8n |
| Workflow error | Slack | n8n Error Trigger |
| Weekly summary (leads generated, replies, completion rate) | Email | Scheduled n8n sub-workflow |

### Priority 4 — Consolidate Email Sending

Move follow-up emails from raw SMTP to Smartlead's built-in subsequence system:
- All 3 emails sent through same warmed-up infrastructure
- Reply detection handled natively by Smartlead
- Unified analytics in one dashboard
- Phase 7 simplifies to just Google Sheets status updates via Smartlead webhooks

### Priority 5 — Add PhantomBuster Polling

Replace the single fire-and-forget HTTP call with a polling loop:
1. Launch agent → get `containerId`
2. Poll `GET /api/v2/containers/fetch?id={containerId}` every 15 seconds
3. When `status === 'finished'`, fetch results
4. Timeout after 3 minutes → log error, skip brand

### Priority 6 — Scale-Ready Improvements (When Needed)

| Improvement | When to Implement | Benefit |
|------------|-------------------|---------|
| Move CRM to Airtable or HubSpot | > 500 leads | Better querying, dashboards, relational data |
| Add Zapier/Make.com as backup orchestrator | If n8n uptime is a concern | Redundancy |
| Implement webhook-based triggers | If KeHE provides an API | More reliable than email parsing |
| Add A/B testing for email templates | After 100+ leads | Optimize open/reply rates |
| Build reporting dashboard | After 1 month of data | Track ROI, conversion rates |

---

## Process Improvement Opportunities

These are areas where external tools or process changes could enhance the overall system — things to plan around with your broader toolset.

### Lead Scoring & Prioritization
- Currently all qualified brands are treated equally
- Could add scoring based on: employee count, industry fit, Apollo revenue data, social media presence
- Higher-scored leads could get priority outreach or different email sequences

### Multi-Source Lead Ingestion
- Currently only KeHE Spotlight emails trigger the workflow
- Could add triggers for: UNFI New Products, trade show attendee lists, industry newsletters, competitor brand monitoring

### CRM Integration
- Google Sheets works but doesn't scale
- Consider: HubSpot Free CRM, Pipedrive, or Airtable
- Benefits: pipeline stages, deal tracking, automated reminders, team collaboration

### Response Handling
- Currently, when a prospect replies, the sheet is marked "replied" — and that's it
- Could add: auto-categorization of reply sentiment (interested / not interested / wrong person), auto-creation of calendar booking link, Slack alert with reply content for immediate follow-up

### Analytics & Reporting
- No analytics currently exist
- Key metrics to track:
  - Brands per KeHE email (extraction success rate)
  - ICP filter pass rate
  - Email discovery rate
  - Open rates (from Smartlead)
  - Reply rates by brand category
  - Meeting-booked rate
  - Cost per qualified lead

### Email Template Optimization
- Templates are hardcoded in JavaScript
- Could move to: an Airtable/Sheet-based template system that non-technical team members can edit
- Could add: AI-powered personalization using OpenAI API to generate truly unique emails per brand (not just template fill)

---

## One-Time Manual Setup Checklist

These are the only manual steps required. Once done, the workflow runs hands-free.

- [ ] **n8n Instance** — Self-hosted or n8n Cloud account provisioned
- [ ] **Import Workflow** — `kehe-lead-gen-workflow.json` imported into n8n
- [ ] **IMAP Credential** — Email account credentials configured in n8n
- [ ] **Google Sheets OAuth2** — Google API credentials set up, OAuth2 flow completed
- [ ] **SMTP Credential** — Outreach email SMTP settings (if using direct send)
- [ ] **Environment Variables** — All 9 variables set in n8n Settings → Variables:
  - [ ] `SERP_API_KEY`
  - [ ] `APOLLO_API_KEY`
  - [ ] `PHANTOMBUSTER_API_KEY`
  - [ ] `PHANTOMBUSTER_LINKEDIN_AGENT_ID`
  - [ ] `FINDYMAIL_API_KEY`
  - [ ] `SMARTLEAD_API_KEY`
  - [ ] `SMARTLEAD_CAMPAIGN_ID`
  - [ ] `SMTP_FROM_EMAIL`
  - [ ] `SMTP_REPLY_TO`
- [ ] **Google Sheet** — "Lead Tracker" tab created with 13 columns
- [ ] **Smartlead Campaign** — Campaign created with custom field templates
- [ ] **PhantomBuster Agent** — LinkedIn Sales Navigator search agent configured
- [ ] **Activate Workflow** — Toggle workflow to "Active" in n8n

**After setup, the workflow runs autonomously on every new KeHE email.**

---

## Summary: Current Automation Score

| Category | Score | Notes |
|----------|:-----:|-------|
| Email ingestion & parsing | 10/10 | Fully automated, includes fallback |
| Company enrichment | 9/10 | Works well; no retry on failure |
| Decision maker discovery | 8/10 | Works but no PhantomBuster polling |
| Email finding & verification | 9/10 | Solid; no retry on failure |
| CRM & deduplication | 9/10 | Works; Google Sheets has scale limits |
| Personalized outreach generation | 10/10 | Strong templates with category/role personalization |
| Campaign enrollment (Smartlead) | 10/10 | Fully automated |
| Follow-up sequence | 4/10 | **Reply detection is placeholder; SMTP vs Smartlead split** |
| Error handling | 1/10 | No retries, no error notifications |
| Observability & reporting | 1/10 | No notifications, no dashboards |
| **Overall** | **7/10** | **Strong foundation — needs reply detection fix and error handling to be production-ready** |

---

*Use this document as your planning baseline. The workflow's core pipeline (Phases 1–6) is solid and fully automated. The highest-impact improvements are: (1) fixing reply detection, (2) consolidating all emails through Smartlead, and (3) adding error handling with notifications.*
