# LinkedIn Intelligence System
### VysionAI · n8n Self-Hosted · Saswati Gorai

A two-workflow system that scrapes LinkedIn posts from a curated list of creators, scores each post for relevancy using AI, stores them in Google Sheets, and sends a personalised intelligence digest every 3–4 days.

---

## System Overview

```
WF1: LinkedIn Creator Dump
Config Sheet + Query Sheet → Apify Scraper → Score via Claude → Write to LinkedIn_Dump → Update Config

WF2: LinkedIn Post Digest
LinkedIn_Dump (unprocessed rows) → Claude Digest → Gmail → Mark processed
```

**Google Sheet tabs required:** `Config`, `LinkedIn_Query`, `LinkedIn_Dump`

---

## Workflow 1 — LinkedIn Creator Dump

**What it does:**
Reads a list of LinkedIn creators from the `LinkedIn_Query` sheet, scrapes their posts using Apify, filters posts by date, scores each post for relevancy using Claude via OpenRouter, and writes scored rows to the `LinkedIn_Dump` sheet. Also updates `last_scraped_date` in the Config sheet after each run.

**Two schedules run in parallel:**
- `Schedule Trigger1` — Weekly on Monday at 10pm: runs the full scrape pipeline
- `Schedule Trigger` — Weekly on Monday at 11pm: reads the dump and updates the config date

### Node-by-Node

| Node | Purpose |
|---|---|
| Schedule Trigger1 | Triggers the scrape pipeline weekly |
| Read Config Sheet | Reads `last_scraped_date` from Config tab |
| Read LinkedIn Query Sheet | Reads the list of creators from `LinkedIn_Query` tab |
| Prepare Creators + Date Range | Sets `from_date` per creator — `2026-01-01` for `new` type, `last_scraped_date` for `existing` type |
| Process Each Creator | Splits creators into individual items for per-creator processing |
| Run Apify Scraper | POSTs to Apify's `harvestapi~linkedin-profile-posts` actor — fetches up to 50 posts per creator sorted by date |
| Wait 30 Seconds | Pauses to allow Apify run to complete |
| Poll Run Status | Checks if the Apify run has finished |
| Fetch Apify Results | GETs up to 200 post items from the Apify dataset |
| Filter by Date + Map Fields | Drops posts before `from_date`, maps all fields, truncates post text at 5,000 chars |
| Score Relevancy via OpenRouter | Sends post text + your context to Claude — returns a score (0–100) and a short reason |
| Assemble Final Row | Merges score + reason into the post object, sets `processed = false` |
| Write to LinkedIn Dump | Appends the row to the `LinkedIn_Dump` sheet |
| Flip New to Existing in Query Sheet | Updates creator `type` from `new` → `existing` so future runs only fetch incremental posts |
| Schedule Trigger | Triggers the config update pipeline weekly (slightly after the scrape) |
| Read LinkedIn Dump | Reads all rows from `LinkedIn_Dump` to find the latest scraped date |
| Compute Max Scraped Date | Finds the maximum `scraped_at` value across all rows |
| Update Config Date | Writes the latest date back to `last_scraped_date` in Config |

---

## Relevancy Scoring

Each post is scored 0–100 by Claude based on your professional context.

**Scoring tiers:**
| Score | Meaning |
|---|---|
| 80–100 | Directly actionable — GTM workflow idea, AI tool announcement, hiring signal, outbound tactic, automation system to build or use now |
| 50–79 | Useful background — market trends, strategic framing, good content angle |
| 0–49 | Not relevant — motivational fluff, personal news, unrelated industry |

Only posts with `relevancy_score >= 20` are included in the digest (configurable).

---

## Workflow 2 — LinkedIn Post Digest

**What it does:**
Reads unprocessed rows from `LinkedIn_Dump`, filters to the last 7 days with a minimum relevancy score, builds a context-aware prompt, sends it to Claude via OpenRouter, emails a formatted digest, and marks all included posts as processed.

**Schedule:** Mon / Wed / Sat at 12pm

### Node-by-Node

| Node | Purpose |
|---|---|
| Schedule Every 3-4 Days | Triggers on Mon / Wed / Sat at noon |
| Read Unprocessed Posts from Dump | Reads rows where `processed = false` |
| Context + Build Prompt — EDIT HERE | Filters to last 7 days + min score, sorts by relevancy, builds the full Claude prompt |
| Claude Generate Digest | POST to OpenRouter → Claude Haiku — max 4,096 tokens |
| Send LinkedIn Digest Email | Sends styled HTML email with clickable post URLs to your address |
| Prep Processed IDs | Extracts `post_id` values from the current batch |
| Mark as Processed in Sheet | Updates `processed = true` for each post — prevents re-inclusion |

### Digest Format (7 sections)

1. **New AI Tools & Products** — tool name | what it does | why relevant | post URL
2. **GTM & Growth Trends** — key insights, interview flags, post URLs
3. **How Companies Are Running GTM** — team structures, hiring signals
4. **Automation & Workflow Ideas** — VysionAI offering or portfolio angles
5. **Interview & Outreach Language** — exact phrases and frameworks
6. **Content Ideas** — 3–5 trending content angles with post URLs
7. **Must-Read Posts This Round** — top 5 posts: creator | reason | URL

---

## Google Sheet Setup

Three tabs required in one sheet:

**Tab: `Config`**
| key | value | notes |
|---|---|---|
| last_scraped_date | 2026-01-01 | Set to your desired backfill start date on first run |

**Tab: `LinkedIn_Query`**
| creator_name | profile_url | type | niche |
|---|---|---|---|
| Creator Full Name | https://linkedin.com/in/handle | new | GTM / AI / Automation |

- Set `type` to `new` for first-time scrape of a creator
- Workflow automatically flips it to `existing` after first run

**Tab: `LinkedIn_Dump`**
Columns (in order):
`post_id` | `creator_name` | `creator_url` | `post_url` | `post_text` | `repost_text` | `post_type` | `posted_date` | `likes` | `comments` | `shares` | `relevancy_score` | `relevancy_reason` | `processed` | `scraped_at`

> Note: several column names have a trailing space in the sheet (e.g. `creator_name `) — keep this exactly as-is when creating the sheet or the workflow mappings will break.

---

## Configuration

### Change the lookback window (WF2)
In `Context + Build Prompt — EDIT HERE`:
```js
const DAYS_BACK = 7;
```

### Change the minimum relevancy score threshold (WF2)
In the same node:
```js
const MIN_SCORE = 20;
```

### Change the number of posts Apify fetches per creator (WF1)
In `Run Apify Scraper`, update the JSON body:
```json
"maxPostsPerProfile": 50
```

### Change your context / profile used for scoring and digest (both)
Edit the `MY_CONTEXT` object in:
- `Filter by Date + Map Fields` (WF1) — used for scoring
- `Context + Build Prompt — EDIT HERE` (WF2) — used for digest

### Change the AI model
In `Score Relevancy via OpenRouter` (WF1) and `Claude Generate Digest` (WF2):
```json
"model": "anthropic/claude-haiku-4-5"
```

### Change digest delivery email (WF2)
In `Send LinkedIn Digest Email`, update the `sendTo` field.

---

## Dependencies

| Credential | Used In | Type |
|---|---|---|
| Google Sheets OAuth2 | WF1 + WF2 | OAuth2 |
| Gmail OAuth2 | WF2 | OAuth2 |
| Apify API key | WF1 — Run Apify Scraper, Poll Run Status, Fetch Apify Results | Bearer token in Authorization header |
| OpenRouter API key | WF1 — Score Relevancy, WF2 — Claude Generate Digest | `x-api-key` header |

---

## First Run Checklist

- [ ] Create the Google Sheet with three tabs: `Config`, `LinkedIn_Query`, `LinkedIn_Dump`
- [ ] Add column headers exactly as listed above (including trailing spaces where noted)
- [ ] Set `last_scraped_date` in Config to `2026-01-01` (or your preferred start date)
- [ ] Add at least one creator row in `LinkedIn_Query` with `type = new`
- [ ] Connect Google Sheets and Gmail OAuth2 credentials in n8n
- [ ] Add your Apify API key to all three Apify HTTP nodes
- [ ] Add your OpenRouter API key to the scoring and digest nodes
- [ ] Import both workflow JSONs into n8n
- [ ] Run WF1 manually once — verify posts appear in `LinkedIn_Dump` with scores
- [ ] Run WF2 manually once — verify digest email arrives
- [ ] Activate both workflows for scheduled runs

---

## Troubleshooting

**No posts returned from Apify**
- Check the Apify API key is valid
- Verify the creator's LinkedIn profile URL is correct and public
- The `harvestapi~linkedin-profile-posts` actor must be available in your Apify account

**Posts appear but score is 0 or blank**
- OpenRouter API key may be invalid or out of credits
- Check the `Score Relevancy via OpenRouter` node response for error messages

**Digest email not sending**
- Check Gmail OAuth2 credential is still authorised — re-authenticate if expired
- Check there are rows with `processed = false` and `relevancy_score >= MIN_SCORE` in the last `DAYS_BACK` days

**Creator keeps getting scraped from Jan 2026 instead of incrementally**
- Check the `Flip New to Existing in Query Sheet` node ran successfully
- The `type` column in `LinkedIn_Query` should read `existing` after first run

**Trailing space issue in column names**
- Several columns were created with a trailing space (e.g. `processed `)
- If rows are not being marked as processed, verify the column name in the sheet matches exactly

---

*Built by Saswati Gorai · VysionAI · vysionai.com*
