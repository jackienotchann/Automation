# TroopMessenger Lead Generation Workflow

An automated n8n workflow that scrapes Reddit and Quora every few hours, scores potential leads using a two-stage AI pipeline (Groq → Gemini), and saves qualified leads to Google Sheets with a reply draft ready to post.

Built during a summer internship at [Tvisha Technologies](https://tvisha.com) for the [Troop Messenger](https://www.troopmessenger.com) product.

---

## What it does

Every 2 hours the workflow runs four independent pipelines in parallel:

| Pipeline | Source | Output sheet |
|---|---|---|
| Reddit Lead Generation | Reddit via Serper API | `Reddit_Leads` |
| Quora Lead Generation | Quora via Serper API | `Quora_Leads` |
| Reddit Warmup | Reddit (non-sales subreddits) | `Reddit_Warmup_Posts` |
| Quora Warmup | Quora (casual topics) | `Quora_Warmup_Posts` |

**Lead pipelines** find posts where people are frustrated with Slack/Teams, evaluating self-hosted chat, or asking about team communication tools — then score, verify, and draft a reply for each one.

**Warmup pipelines** generate organic-looking replies to unrelated posts (cricket, gaming, food, etc.) to build account credibility before outreach.

---

## Architecture

### Lead scoring (two-stage AI)

```
Serper search
    └── 300 posts fetched
        └── Pre-filter (deduplicate against sheet)
            └── Stage 1: Groq (llama-3.1-8b-instant)
                    Scores 0.0–1.0, extracts intent/role/pain points
                └── Score ≥ 0.6 gate
                    └── Stage 2: Gemini 2.0 Flash
                            Validates score, writes reply draft
                        └── Both AIs ≥ 0.6 gate
                            └── Save to Google Sheets + Gmail alert
```

### Warmup pipeline

```
Serper search (casual topics)
    └── Loop with 3s rate limit delay
        └── Groq generates casual human-like reply
            └── Deduplicate check
                └── Save to Google Sheets + Gmail alert
```

---

## Prerequisites

- **n8n** self-hosted or cloud (tested on n8n v1.x)
- **Google Sheets** with the four tab names listed below
- API keys for the four services below

---

## API Keys needed

| Service | Where to get it | Used for |
|---|---|---|
| [Serper](https://serper.dev) | serper.dev | Searching Reddit & Quora |
| [Groq](https://console.groq.com) | console.groq.com | Stage 1 AI scoring (free tier works) |
| [Google Gemini](https://aistudio.google.com) | aistudio.google.com | Stage 2 AI crosscheck (free tier works) |
| [ChatAnywhere](https://chatanywhere.tech) | chatanywhere.tech | Secondary crosscheck model |

> **Tip:** Groq's free tier has rate limits. The workflow uses 3 separate Groq API keys in rotation (`YOUR_GROQ_API_KEY_1/2/3`) to stay within limits. You can use the same key for all three if you have a paid plan.

---

## Google Sheets setup

Create a spreadsheet and add these four sheets (exact tab names matter):

### `Reddit_Leads`
| Column | Description |
|---|---|
| POST_ID | Unique Reddit post ID |
| Title | Post title |
| Text | Full post body |
| URL | Link to post |
| Subreddit | e.g. `sysadmin` |
| Reddit_Score | Reddit upvote score |
| Author | Reddit username |
| AI_Score | Final combined AI score (0.0–1.0) |
| Intent | e.g. `competitor_switch` |
| Urgency | `high` / `medium` / `low` |
| Poster_Role | e.g. `CTO`, `IT Manager` |
| Pain_Points | Extracted pain points |
| Tools_Mentioned | Tools the poster mentioned |
| Is_Decision_Maker | `true` / `false` |
| Outreach_Angle | Suggested angle for reply |
| Reply_Draft | AI-generated reply ready to post |

### `Quora_Leads`
Same as above minus `Text`, `Subreddit`, `Reddit_Score` — plus:

| Column | Description |
|---|---|
| Answer_Draft | AI-generated Quora answer |
| Reply_Posted | `Yes` / `No` |
| Reply_Date | Date replied |
| Converted | `Yes` / `No` |
| Notes | Manual notes |

### `Reddit_Warmup_Posts`
| Column | Description |
|---|---|
| POST_ID | Reddit post ID |
| Title | Post title |
| URL | Post URL |
| Subreddit | Subreddit name |
| Topic | Topic category |
| Warmup_Reply | Generated casual reply |
| Reply_Posted | `Yes` / `No` |

### `Quora_Warmup_Posts`
| Column | Description |
|---|---|
| POST_ID | Quora question ID |
| Title | Question title |
| URL | Question URL |
| Topic | Topic category |
| Warmup_Answer | Generated casual answer |
| Reply_Posted | `Yes` / `No` |

---

## Setup instructions

### 1. Import the workflow

1. Open your n8n instance
2. Go to **Workflows → Import**
3. Upload `TroopMessenger_LeadGen_Workflow_PUBLIC.json`

### 2. Set up credentials in n8n

Go to **Settings → Credentials** and create:

- **Google Sheets OAuth2** — connect your Google account
- **Gmail OAuth2** — connect your Gmail account (for lead alerts)

### 3. Replace API key placeholders

Search for `YOUR_` in the workflow's Code nodes and replace:

| Placeholder | Replace with |
|---|---|
| `YOUR_SERPER_API_KEY` | Your Serper.dev API key |
| `YOUR_GROQ_API_KEY_1` | Groq key for Reddit scoring |
| `YOUR_GROQ_API_KEY_2` | Groq key for Quora scoring |
| `YOUR_GROQ_API_KEY_3` | Groq key for warmup replies |
| `YOUR_GEMINI_API_KEY` | Gemini API key (in the HTTP Request node URL) |
| `YOUR_CHATANYWHERE_API_KEY` | ChatAnywhere bearer token |

### 4. Update the Spreadsheet ID

In every Google Sheets node, update the **Document ID** to your spreadsheet's ID.  
You can find it in the URL: `https://docs.google.com/spreadsheets/d/YOUR_SPREADSHEET_ID/edit`

### 5. Update Gmail alert recipient

In the Gmail nodes (`Reddit Gmail Alert`, `Quora Gmail Alert`, etc.), update the **To** field to your email address.

### 6. Activate

Toggle the workflow to **Active**. It will run every 2 hours automatically.

---

## Scoring guide

The AI uses this scoring scale:

| Score | Meaning |
|---|---|
| **1.0** | Perfect lead — actively asking for self-hosted/on-premise chat, evaluating Slack/Teams with data sovereignty concerns |
| **0.8** | Strong lead — dissatisfied with SaaS pricing/privacy, IT/CTO evaluating messaging stack |
| **0.6** | Marginal lead — vague communication pain, no clear switch intent |
| **0.0** | Not a lead — personal use, already satisfied, unrelated topic |

Only posts where **both** Groq and Gemini score ≥ 0.6 are saved.

---

## Dashboard

A React + Tailwind dashboard to visualise leads is available separately. It reads directly from the Google Sheet and provides:

- Filterable leads table with full post text on expand
- Score, urgency, intent, and role filters
- One-click copy of reply drafts
- Mark replied tracking
- Analytics charts

---

## Subreddits monitored

`sysadmin` · `msp` · `remotework` · `Slack` · `startups` · `smallbusiness` · `Entrepreneur` · `remoteteams` · `projectmanagement` · `ProductManagement` · `IndianStartups` · `itmanagers` · `devops` · `selfhosted` · `homelab` · `networking` · `cybersecurity` · `businessowners` and more.

---

## Rate limiting

The workflow includes built-in delays to avoid hitting API limits:

- 3s wait between warmup post iterations
- 5s wait between lead scoring iterations
- Three Groq keys in rotation for scoring

---

## License

MIT — free to use, fork, and adapt.

---

## Author

**Prathamesh Mohite**  
PGDM 2025–27 · Marketing + IT & Analytics · IMT Hyderabad  
Summer Intern · Tvisha Technologies, Hyderabad
