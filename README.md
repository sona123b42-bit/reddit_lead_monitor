# Reddit Lead Monitor

An n8n workflow that watches a bunch of subreddits (and Hacker News) for posts
that smell like a lead — someone describing a manual, repetitive, or painful
business process — and drops them into Slack with a one-click "Mark Done"
button so you can track which ones you've already followed up on.

Built for freelance automation work: the idea isn't just "find people asking
for automation," it's "find people complaining about a problem automation
could fix," even if they never say the word "automation."

## How it works

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌────────────┐
│ Hourly timer │────▶│ Pull RSS from │────▶│ Keyword filter │────▶│ Dedup vs.  │
│ + HN search  │     │ 11 subreddits │     │ (title scan)   │     │ Data Table │
└─────────────┘     └──────────────┘     └───────────────┘     └─────┬──────┘
                                                                       │ new only
                                                                       ▼
┌────────────────┐     ┌───────────────┐     ┌───────────────────────────┐
│ Slack message   │◀────│ AI relevance  │◀────│ Clean HTML out of the RSS  │
│ w/ "Mark Done"  │     │ check (LLM)   │     │ post content                │
└────────┬────────┘     └───────────────┘     └───────────────────────────┘
         │
         ▼
┌─────────────────┐     ┌───────────────────────────────────────────────┐
│ Save link to     │     │ Slack "Mark Done" button click → webhook →     │
│ Data Table        │     │ edits the original message to show who        │
│ (so it's not      │     │ marked it done and when                       │
│ re-sent again)     │     └───────────────────────────────────────────────┘
└─────────────────┘
```

Two independent triggers feed the same pipeline:

1. **Every Hour** (schedule trigger) → fans out to 11 subreddit RSS feeds
   and a Hacker News search, then merges everything back together.
2. **Webhook** (`POST /reddit-done`) → handles both Slack's URL-verification
   handshake and the interactive button clicks from messages this workflow
   sent earlier.

### Pipeline stages

| Stage | Node(s) | What it does |
|---|---|---|
| Collect | `RSS - r/*`, `Code in JavaScript` (HN) | Pulls new posts. Each RSS node is spaced out by a `Wait` node (90s) so Reddit doesn't rate-limit the run. |
| Merge & normalize | `Merge everything` | Pulls every feed's output together, keeps only posts from the last 6.5h. |
| Keyword pre-filter | `Filter all data` | Cheap title-only keyword scan (`automate`, `manual`, `spreadsheet`, `follow-up`, etc.) so the AI step isn't called on obviously irrelevant posts. |
| Dedup | `Check If Seen` → `Is New?` | Looks the post's link up in the `Reddit Data` n8n Data Table. Already-seen posts are dropped here. |
| Clean content | `Make it so AI can read this` | Strips HTML tags out of the RSS content and truncates to 500 chars. |
| AI screening | `Message a model` → `Parse from ai` | Sends title + cleaned content to `gpt-4o-mini` with a prompt that asks it to judge whether the post describes a pitchable business problem. Expects strict JSON back (`{"relevant": bool, "reason": "..."}`). |
| Notify | `Send slack message` | Posts a Slack message (via `chat.postMessage`) with the title, a link to the Reddit post, which subreddit it came from, why the AI flagged it, and a **Mark Done** button. |
| Record | `Mark As Seen` | Writes the link + title into the `Reddit Data` Data Table so it's never sent twice. |

### The Slack "Mark Done" button

This is what the webhook branch is for:

- Slack's Events API sends a one-time `url_verification` POST when you first
  point it at the webhook — `Handle Slack Verification` → `Is Challenge?` →
  `Respond With Challenge` answers that automatically.
- A real button click arrives as a `block_actions` payload (form-urlencoded,
  with a JSON string in the `payload` field). `Parse Button Click` pulls out
  the action, the value (the Reddit link), and who clicked it.
- `HTTP Request` POSTs back to Slack's `response_url` with
  `replace_original: true`, so the original Slack message is edited in place
  to say `✅ Done — marked by <username>` instead of popping up a new message.

## Repo structure

```
reddit_lead_monitor/
├── README.md
└── workflows/
    └── reddit-lead-monitor.json   # importable n8n workflow export
```

## Setting it up

1. **Import** `workflows/reddit-lead-monitor.json` into n8n (Workflows →
   Import from File).
2. **Credentials** — the workflow needs:
   - An **HTTP Header Auth** credential (named `Reddit bot` in the export)
     used for both the outbound Slack API calls (`Send slack message`,
     `HTTP Request`). Set the header to `Authorization: Bearer xoxb-...`
     with a Slack bot token that has the `chat:write` scope.
   - An **OpenAI / AI Gateway** credential for the `Message a model` node
     (any provider n8n supports for the AI node works — swap the model if
     you're not using OpenAI).
3. **Data Table** — create an n8n Data Table named `Reddit Data` with two
   text columns: `link` and `title`. This is the dedup store; without it
   every run will re-post everything. Re-point the `Check If Seen` and
   `Mark As Seen` nodes at your table's ID after creating it (imports don't
   carry the table ID over).
4. **Slack app**:
   - Enable **Interactivity & Shortcuts**, pointing the Request URL at your
     webhook (`https://<your-n8n-host>/webhook/reddit-done`).
   - If you also want the Events API handshake handled, point the Events
     Request URL at the same webhook.
   - Update the hardcoded `channel` ID in the `Send slack message` node's
     JSON body to your own channel.
5. **Activate** the workflow. The schedule trigger runs hourly; adjust the
   `Every Hour` node if you want a different cadence (watch out for Reddit
   rate limits if you go much more frequent — that's what the `Wait` nodes
   between feeds are for).

### Subreddits monitored

`r/smallbusiness`, `r/forhire`, `r/nocode`, `r/Entrepreneur`, `r/ecommerce`,
`r/shopify`, `r/dropship`, `r/realestateinvesting`, `r/agency`,
`r/consulting`, `r/business` — plus a Hacker News "new stories" search as a
bonus source.

## Notes / known quirks

- This is an export straight out of n8n, so node names/IDs and canvas
  positions are exactly as n8n generated them (e.g. `Wait`, `Wait1`,
  `Wait9`–`Wait15` aren't in execution order — that's just n8n's
  auto-numbering when nodes were added over time).
- The export's `pinData` (sample webhook payloads from testing, including
  Slack signing/IP metadata) and instance-specific IDs (`meta`, `versionId`,
  `errorWorkflow`) were stripped before committing — they're local n8n
  debug artifacts, not part of the workflow logic, and some of that sample
  data was sensitive enough not to belong in a repo.
- The keyword pre-filter in `Filter all data` only scans the **title**, not
  the body — intentional, to keep the AI call cheap, but it does mean a
  post with a generic title and a relevant body gets skipped.
