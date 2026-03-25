# Product Operations Hub

A product operations dashboard showing how teams can centralize feedback, track launches, and stay aligned across Engineering, Sales, and Leadership.

**[View Live Demo](https://joannapedrina.github.io/product-ops-hub-2.0/)**

## What This Is

**This is a UI prototype + technical spec.**

The demo shows the user experience with sample data. The docs below describe how you'd build the real thing.

| Layer | Status |
|-------|--------|
| UI/UX Design | Done |
| Sample Data | Done |
| Backend API | Spec only |
| Database | Spec only |
| AI Integration | Spec only |
| Connectors | Spec only |

## The Problem

| Challenge | Solution |
|-----------|----------|
| Feedback scattered across tools | Unified dashboard pulling from all sources |
| Sales doesn't know what's shipping | Launch calendar with status filters |
| Launch readiness unclear | Checklists with team playbooks |
| No single source of truth | Hub connecting all views |

## Architecture

```
                    PRODUCT OPERATIONS HUB
    +--------------------------------------------------+
    |                                                  |
    |   Feedback       Launch         Launch           |
    |   Dashboard      Calendar       Checklists       |
    |       |             |              |             |
    |       +-------------+-------------+              |
    |                     |                            |
    |               Unified API                        |
    |                     |                            |
    +---------------------|----------------------------+
                          |
        +-----------------+------------------+
        |                 |                  |
    Intercom          Linear             Gong
    Zendesk           Jira               Chorus
    Support           GitHub             Calls
```

## Components

### Product Operations Hub
Main command center with tabs:
- **Overview**: Key metrics at a glance
- **Product Feedback**: Dashboard + All Feedback views (see below)
- **Launch Calendar**: Interactive 12-month roadmap
- **Launch Checklists**: Team playbooks by launch type
- **Product Roadmap**: Sales-Engineering sync view

### Feedback Dashboard
Two tabs with a simple narrative: **Raw feedback → TechCo categorises → Customer data enriches → Prioritised queue**

**Dashboard Tab** (3 sub-views):
- **Overview**: Key metrics (Total Feedback, Critical Issues, Sentiment, ARR at Risk) + Weighted Priority Queue
- **Feedback Queue**: Breakdown by category and product area (mapped to engineering teams)
- **AI Pipeline**: Visual flow showing Ingest → Categorise → Enrich → Prioritise

**All Feedback Tab**:
- Searchable, filterable table of all feedback items
- Filter by source, category, priority
- Export to CSV

**Data Sources:**

| Support | Product | Sales |
|---------|---------|-------|
| Intercom | GitHub Issues | Granola transcripts |
| Zendesk | Linear | CRM notes |

**AI Processing (TechCo models):**
- Auto-categorises: Bug, Feature Request, Docs, Praise
- Sentiment scoring: -1.0 to +1.0
- Duplicate detection via TechCo Embed
- Theme clustering

**Customer-Weighted Prioritisation:**
```
priority_boost = (ARR / $100K) × tier_multiplier × (1 - days_since/90)
```
- Enterprise: 2.0×, Growth: 1.5×, Pro: 1.2×, Starter: 1.0×

**MVP Implementation:** Can be built with Airtable + Zapier + TechCo API (no engineering dependency)

### Launch Calendar
Two views:
- Calendar: 12-month grid
- Timeline: quarterly breakdown

Filters by status (Shipped, Beta, In Dev, At Risk) and team (Engineering, ML, Product).

Syncs with Linear/Jira for real status. Links back to customer requests.

### Launch Checklists

| Launch Type | Complexity |
|-------------|------------|
| Major Model Release | High |
| API/Platform Update | Medium |
| Enterprise Feature | Medium |
| Experimental/Beta | Low |
| Bug Fix / Hotfix | Low |
| Documentation | Low |

Each team gets their own checklist:
- Engineering: tests, docs, monitoring, rollback plan
- Marketing: blog, social, email, landing page
- Sales: battlecards, demo, pricing, FAQ
- Legal: ToS, privacy, compliance
- Support: KB articles, training, escalation paths

## Data Flow

**Feedback to Roadmap:**
```
Customer submits feedback (Intercom)
    |
    v
AI triage: categorize, score, detect duplicates
    |
    v
Feedback Dashboard: aggregate by theme, track votes
    |
    v
PM reviews top requests
    |
    v
Creates Linear project --> appears in Launch Calendar
    |
    v
Generates checklist --> Launch Checklists
```

**Launch to Customer Notification:**
```
Feature ships
    |
    v
Query: find all customers who requested this
    |
    v
Send: in-app message, email, Intercom campaign
```

## AI Triage System

The AI layer processes every piece of feedback before it hits the dashboard. Here's how it works:

### Pipeline Overview

```
Raw Feedback (Intercom, Zendesk, etc.)
    |
    v
+-------------------+
|  Ingestion Queue  |  (BullMQ)
+-------------------+
    |
    v
+-------------------+
|  AI Triage Job    |  (TechCo API)
+-------------------+
    |
    +---> Categorization
    +---> Sentiment Analysis
    +---> Priority Scoring
    +---> Tag Extraction
    +---> Duplicate Detection
    |
    v
+-------------------+
|  Enriched Record  |  (PostgreSQL)
+-------------------+
    |
    v
Dashboard displays processed feedback
```

### What the AI Does

**1. Categorization**

The AI reads the feedback and assigns a category:

| Category | Examples |
|----------|----------|
| Bug | "API returns 500 error", "Dashboard won't load" |
| Feature Request | "Would love batch processing", "Need SAML support" |
| Improvement | "Docs could be clearer", "Latency is a bit high" |
| Question | "How do I rotate API keys?", "What's the rate limit?" |
| Complaint | "Been waiting 3 days for support", "This is frustrating" |
| Praise | "Love the new embeddings!", "Your team is amazing" |

**2. Sentiment Scoring**

Returns a score from -1 to +1:

| Score | Label | Signals |
|-------|-------|---------|
| 0.5 to 1.0 | Positive | Happy customer, praise, satisfaction |
| -0.3 to 0.5 | Neutral | Questions, normal requests |
| -1.0 to -0.3 | Negative | Frustration, urgency, complaints |

**3. Priority Inference**

AI suggests priority based on:
- Customer tier (Enterprise > Pro > Free)
- Severity language ("production down" vs "minor inconvenience")
- Revenue at risk (ARR from CRM lookup)
- Historical churn signals

```
Priority Score = (Tier Weight x 3) + (Severity x 2) + (Revenue x 1)

Enterprise + Production Issue + High ARR = Critical
Free + Nice-to-have + Low ARR = Low
```

**4. Smart Tagging**

Extracts technical keywords and product areas:

| Input | Tags Generated |
|-------|----------------|
| "Getting 429 errors on embeddings endpoint" | rate-limit, embeddings, api-error |
| "Fine-tuning job stuck at 80%" | fine-tuning, training, stuck |
| "Need SSO for our team" | sso, enterprise, authentication |

**5. Duplicate Detection**

Uses embeddings to find similar feedback:

1. Generate embedding vector for new feedback
2. Compare against existing feedback (cosine similarity)
3. If similarity > 0.85 and within 30 days, flag as potential duplicate
4. Link to parent issue for grouping

### Triage Prompt Example

```
You are analyzing customer feedback for a product ops dashboard.

Feedback:
Title: {title}
Message: {message}
Customer: {company} ({plan} plan, ${arr} ARR)

Tasks:
1. Categorize: bug, feature_request, improvement, question, complaint, praise
2. Sentiment score: -1.0 to 1.0
3. Priority: critical, high, medium, low (explain reasoning)
4. Tags: up to 5 technical keywords
5. Product area: API, Dashboard, Fine-tuning, Embeddings, Billing, Docs
6. Similar issues: any patterns you notice

Return JSON.
```

### Processing Queue

Feedback processing happens async via job queue:

| Job Type | Trigger | Timing |
|----------|---------|--------|
| New feedback | Webhook from source | Immediate |
| Re-triage | Manual or bulk update | On demand |
| Embedding refresh | New model available | Scheduled |

### AI Model Choice

| Task | Model | Why |
|------|-------|-----|
| Categorization + Sentiment | TechCo Lite | Fast, cheap, accurate for classification |
| Priority reasoning | TechCo Mid | Needs context awareness |
| Embedding generation | TechCo Embed | Vector search for duplicates |
| Complex analysis | TechCo Pro | Sales call transcripts, long context |

### Output Structure

After AI processing, each feedback item has:

```javascript
{
  // Original fields
  id: "fb_12345",
  title: "Rate limiting issues",
  description: "...",

  // AI-generated fields
  ai_triage: {
    category: "bug",
    sentiment_score: -0.4,
    sentiment_label: "negative",
    priority: "high",
    priority_reasoning: "Enterprise customer, production impact",
    tags: ["rate-limit", "api", "429"],
    product_area: "API",
    similar_items: ["fb_11234", "fb_10998"],
    processed_at: "2026-01-10T14:35:00Z",
    model_version: "techco-lite-2024"
  }
}
```

### Human Override

AI suggestions are just that. PMs can:
- Change category or priority
- Add/remove tags
- Mark false duplicates
- Flag for re-processing

Every override gets logged for model improvement.

### Important: Validating AI Triage Before Relying On It

AI triage is only useful if it's accurate. Don't deploy it blindly. Here's how to validate:

**Step 1: Create a Test Set**

Take 50-100 feedback items you've already manually categorized. Run them through the AI and compare:

```
Manual: Bug, High Priority
AI:     Bug, High Priority  ✓

Manual: Feature Request, Medium
AI:     Improvement, Low    ✗ (wrong)
```

Target accuracy:
- Category: 85%+
- Priority: 80%+
- Sentiment direction: 90%+

If you're below these thresholds, fix the prompt before deploying.

**Step 2: Shadow Mode**

Run AI triage alongside human triage for 2-3 weeks:

```
Feedback arrives
    |
    +--> Human triages (source of truth)
    |
    +--> AI triages (logged separately)
    |
    v
Weekly review: where did AI disagree?
```

This shows you exactly where AI struggles without any risk.

**Step 3: Watch for Common Failures**

| Problem | Example | Fix |
|---------|---------|-----|
| Sarcasm missed | "Great, another outage" marked Positive | Add sarcasm examples to prompt |
| Missing context | "This blocks us" - blocks what? | Include more context |
| Priority inflation | Everything marked High | Calibrate with real ARR thresholds |
| Edge cases | Bug reports phrased as questions | Add examples to prompt |

**Step 4: Build a Feedback Loop**

Add a simple UI for corrections:

```
[AI says: Bug, High Priority]
[Correct?  ✓ Yes  |  ✗ No]
    If No: [Select correct category] [Select correct priority]
```

Log every correction. Review monthly and update the prompt.

**Step 5: Track What Matters**

Don't obsess over raw accuracy. Track:

| Metric | Why It Matters |
|--------|----------------|
| False negatives on Critical | Did AI miss something urgent? |
| Override rate | If >30%, AI isn't helping |
| Time saved per PM | Are they actually triaging faster? |

**Minimum Test Before Building**

Before writing any pipeline code:

1. Export 20 real Intercom conversations
2. Paste each into TechCo with your triage prompt
3. Compare AI output to what a PM would say
4. If accuracy is good, build the pipeline
5. If not, iterate on the prompt first

## Connector Pattern

Each source uses an adapter:

```javascript
interface DataSourceAdapter {
  authenticate(): Promise<void>;
  fetchFeedback(params): Promise<FeedbackItem[]>;
  fetchIncremental(since: Date): Promise<FeedbackItem[]>;
  transformToStandard(raw): FeedbackItem;
}
```

Sync strategies:
- Batch: daily at 2am for backfill
- Incremental: every 15 min for recent updates
- Webhook: real-time for new items

## Unified Data Model

All feedback normalizes to:

```javascript
{
  id: "fb_12345",
  source: "intercom",
  source_id: "conv_abc123",
  title: "Need CPU inference support",
  description: "We can't use GPUs in our edge deployment...",
  category: "feature_request",
  priority: "high",
  sentiment_score: -0.3,
  customer: {
    id: "cust_789",
    email: "user@company.com",
    company: "Acme Corp",
    tier: "enterprise",
    arr: 50000,
    health_score: 72
  },
  tags: ["cpu", "edge", "deployment"],
  product_area: "inference",
  created_at: "2026-01-10T14:30:00Z",
  status: "open"
}
```

## Tech Stack (Production)

| Layer | Tech |
|-------|------|
| Frontend | Next.js 15, React, TypeScript |
| UI | Shadcn/ui, Tailwind |
| Charts | Recharts |
| State | Zustand, TanStack Query |
| Backend | Next.js API Routes |
| Database | PostgreSQL |
| Cache | Redis (Upstash) |
| Queue | BullMQ |
| AI | TechCo API |
| Auth | Auth.js |
| Hosting | Vercel |

## Design Principles

**Clarity over complexity**
- Show summary first, details on click
- Max 3-4 metrics per screen
- No jargon

**Honest data**
- Y-axis starts at 0
- Show sample sizes
- Display when data was last updated

**Actionable**
- Every metric links to underlying data
- Priority = Impact x Urgency x Customer Value

## Files

```
index.html                           # Main Product Ops Hub (all-in-one)
interactive-launch-calendar.html     # Launch calendar (embedded)
launch-checklist.html                # Launch checklists (embedded)
dashboard-mockup-v4-real-data.html   # Standalone feedback dashboard (legacy)
design-program.html                  # Enterprise Design Program Framework
README.md                            # This file
```

## Try It

1. Open the [Live Demo](https://joannapedrina.github.io/product-ops-hub-2.0/)
2. Click **Product Feedback** in the top nav
3. Explore the Dashboard sub-tabs (Overview, Feedback Queue, AI Pipeline)
4. Switch to **All Feedback** for the searchable table
5. Check **Launch Calendar** and **Launch Checklists**

## Future

- [ ] Real-time updates via WebSocket
- [ ] Slack bot
- [ ] Custom dashboard builder
- [ ] Mobile views
- [ ] Dark mode

Built with HTML, CSS, vanilla JS.

*Sample data for demo purposes. Example use case: TechCo.*
