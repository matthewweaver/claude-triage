---
description: Read your Slack, Jira, Gmail and Google Drive, classify everything into tiers, and update your board (Monday.com or Notion). Tier 1 items are numbered so you can say "draft reply to #2".
---

# /triage

Reads all your sources, classifies messages into three tiers, creates board cards for Tier 1 and Tier 2, and summarises Tier 3 noise inline. Tier 1 items are numbered for reply drafting.

---

## Steps

### 0. Read workspace config

Read `skills/triage/references/workspace-config.md`. Identify the **board backend** (`monday` or `notion`) and the **Board ID** / **Collection ID**. All board operations below branch on this value.

If workspace-config.md is missing or empty: "Run `/triage setup` first to configure your workspace."

---

### 1. Load the board

**Monday.com:** Call `get_board_items_page` on the configured Board ID with `includeColumns: true`, fetching `COL_TRIAGED_AT`, `COL_DUE`, and `COL_LINK`. Collect all existing link URLs as an exclusion set.

**Notion:** Call `notion-fetch` on the configured database ID to retrieve all pages. For each page, extract the `Link` property URL into the exclusion set, and find the most recent `Triaged at` date.

- **Time window**: most recent "Triaged at" date on any card → use as the Slack/Gmail fetch start. Default to last 24 hours if board is empty.

If board not found: "Triage Board not found — run `/triage setup` first."

---

### 2. Promote due items to Today

Find items in **Soon** or **Backlog** with a "Due date" on or before today.

**Monday.com:** For each, `move_item_to_group` → `GROUP_TODAY` and set `COL_STATUS` to `{"label": "Today"}`.

**Notion:** For each matching page (Status = `Soon` or `Backlog`, Due date ≤ today), call `notion-update-page` setting Status = `Today`.

---

### 3. Clear Done items

**Monday.com:** Find all items where Status = "Done" or in group `GROUP_DONE`. For each, `delete_item`. Add their links to the exclusion set.

**Notion:** Find all pages where Status = `Done`. Add their link URLs to the exclusion set, then call `notion-update-page` to archive each (`archived: true`).

---

### 4. Fetch all sources

Run all four fetches in parallel using the time window from step 1.

**A. Slack — Human DMs and @mentions**

Search DMs and @mentions. For each message:
- Skip if permalink is in exclusion set
- Skip bots (GitHub, Google Calendar, etc.) — except Slackbot reminders you set yourself
- Skip if you have already replied after the message (check via `slack_read_channel` on the DM; if `MY_SLACK_ID` has a later message, skip)

Apply channel-priority rules from `workspace-config.md` to inform tier classification.

**B. Jira (channel `JIRA_CHANNEL_ID`, bot `JIRA_BOT_ID`)**

`slack_read_channel` on `JIRA_CHANNEL_ID` with `oldest` = triage window start. Parse messages:

```
*[Person] [action] on a [Task] in `[Status]`*
*[Ticket-ID] [Title]*
>>> [comment snippet]
```

Use tier/urgency rules from `workspace-config.md` Jira filters. Set **Channel** = `Jira`, use the Jira ticket URL as the link.

**C. Gmail (unread inbox, no Jira/Confluence) — snippet-first approach**

`search_threads` with `is:unread in:inbox after:YYYY/MM/DD -from:jira -subject:[JIRA] -from:confluence`.

> ⚠️ High-volume warning: if the query returns 50+ threads, report the count and ask "That's a lot — want me to triage all of them, or just the last 24 hours?" before proceeding.

**Classify from snippet first** — do NOT call `get_thread` for every email:
- Read subject, sender, snippet, and recipient count from the search results
- Auto-Tier 3: marketing, newsletters, bulk sends (10+ recipients), social notifications
- Auto-Tier 1: time-sensitive language ("urgent", "by EOD", "ASAP"), invoices, fraud/security alerts, replies in threads you started
- For ambiguous Tier 1 and Tier 2 candidates only: call `get_thread` to read the full body and confirm classification

For **meeting recording emails**: `get_thread` → extract action items → create Soon cards (see step 6 logic).

Skip newsletters, marketing, bulk emails.

**D. Google Drive — meeting notes and transcripts**

`list_recent_files` (recency, pageSize 20). Filter to docs modified since the triage window. Identify meeting docs:
- Title contains "Transcript" + a date
- Title starts with "Notes –"

Skip any whose view URL is in the exclusion set. For each, `download_file_content` with `exportMimeType: text/plain` → decode content → extract action items assigned to you or unowned → create one **Soon** card per action with due date from text if found, else per config default.

---

### 5. Classify each message

Apply `skills/triage/SKILL.md` rules. Assign:
- **Tier 1 — Reply**: someone is waiting; create a board card
- **Tier 2 — Review**: needs eyes, no reply required; create a board card
- **Tier 3 — Noise**: do NOT create a card; collect for inline summary

Assign an **Urgency** for all Tier 1 and Tier 2 items: Critical 🔥 / High / Medium / Low.

---

### 6. Create board cards (Tier 1 and Tier 2 only)

#### Monday.com

**Tier 1** → group `GROUP_TIER1`
**Tier 2** → group `GROUP_TIER2`
**P0 / Critical production incident** → group `GROUP_P0`

| Field | Column | Value |
|---|---|---|
| Name | — | Concise one-line summary (not copy-pasted) |
| Status | `COL_STATUS` | `{"label": "Tier 1 — Reply"}` or `{"label": "Tier 2 — Review"}` |
| Urgency | `COL_URGENCY` | `{"label": "Critical 🔥"}` / `{"label": "High"}` / `{"label": "Medium"}` / `{"label": "Low"}` |
| Source | `COL_SOURCE` | `DM` / `@mention` / `Thread reply` |
| Channel | `COL_CHANNEL` | Channel or sender name |
| Link | `COL_LINK` | `{"url": "...", "text": "..."}` |
| Triaged at | `COL_TRIAGED_AT` | `{"date": "YYYY-MM-DD"}` |

**Soon cards** (from Drive/Gmail actions):
- Group: `GROUP_SOON`
- Status: `{"label": "Soon"}`
- Urgency: `{"label": "Medium"}`
- Due date: from text or config default

#### Notion

Use `notion-create-pages` with `data_source_id` from workspace-config. Properties:

| Field | Notion property | Value |
|---|---|---|
| Name | title | Concise one-line summary |
| Status | select | `Tier 1 — Reply` / `Tier 2 — Review` / `Today` / `Soon` |
| Group | select | `Tier 1 — Reply` / `Tier 2 — Review` / `P0 — Fire 🔥` / `Soon` |
| Urgency | select | `Critical 🔥` / `High` / `Medium` / `Low` |
| Source | select | `DM` / `@mention` / `Thread reply` |
| Channel | rich_text | Channel or sender name |
| Link | url | Permalink URL |
| Triaged at | date | `date:Triaged at:start` = today's date (ISO 8601) |
| Due date | date | `date:Due date:start` = from text or config default (Soon cards only) |

**P0 / Critical**: set Group = `P0 — Fire 🔥`, Status = `P0 — Fire 🔥`.

---

### 7. Report back

**Tier 1 items are numbered** so you can request a reply draft by number.

Format:

> Triaged [N] messages across Slack, Jira, Gmail and Drive.
> [X] Done cards cleared. [Y] items promoted to Today.
>
> **Tier 1 — Reply** ([count])
> 1. Francisco in #team-channel asking about X — [Urgency]
> 2. Jira: CPE-229 assigned to you — [Urgency]
> 3. …
>
> **Tier 2 — Review** ([count])
> - Bhagyashri shared the Hypothesis ticket update
> - …
>
> **Tier 3 — Noise** ([count] skipped)
> - 12 GitHub bot notifications
> - 8 Google Calendar reminders
> - 4 messages in #random
> - 3 Confluence digest emails
>
> Say "draft reply to #2" to draft a response to any Tier 1 item.
> Say "archive the noise" to archive the Tier 3 emails (requires explicit confirmation).
> [Open your Triage Board]([BOARD_URL])

If nothing actionable: "All clear — no new actionable messages."

---

### 7a. Archive offer (only if triggered by user)

If the user says "archive the noise" or similar:

1. List the Tier 3 emails you would archive (subject, sender).
2. Ask for explicit confirmation: "Archive these [N] emails? This cannot be undone." — use `AskUserQuestion`.
3. Only proceed after confirmation. Never auto-archive.

---

### 8. Reply drafting

When asked to draft a reply (e.g. "draft reply to #3"), retrieve the full thread context and write a draft in the appropriate voice.

**Slack replies** — use the Slack voice from SKILL.md:
- Casual but clear, short (1–3 sentences), lead with the answer
- Thread reply by default, not a channel post
- Present the draft, wait for user confirmation before sending with `slack_send_message`

**Email replies** — use the Email Voice from `workspace-config.md`:
- Read the `## Email Voice` section for the user's preferred style
- Present the draft with subject line
- Wait for user confirmation before sending with `create_draft`

---

### 9. Token usage estimate

After completing the run, estimate token consumption based on content volume.

Track these counts during the run:
- `slack_chars`: total characters in all Slack messages read (DMs, @mentions, Jira channel)
- `jira_chars`: total characters in all Jira notification messages parsed
- `email_snippet_chars`: characters in email snippets classified without `get_thread`
- `email_full_chars`: characters in full email threads fetched via `get_thread`
- `drive_chars`: total characters in all Google Doc content read
- `board_chars`: total characters in all board API responses
- `skill_overhead`: fixed ~4,000 tokens

Print at the end of every triage summary:

```
📊 Token estimate (this run)
   Slack:          ~[slack_chars ÷ 4] tokens  ([N] messages)
   Jira:           ~[jira_chars ÷ 4] tokens   ([N] notifications)
   Gmail snippets: ~[email_snippet_chars ÷ 4] tokens  ([N] threads)
   Gmail full:     ~[email_full_chars ÷ 4] tokens  ([N] threads read in full)
   Drive:          ~[drive_chars ÷ 4] tokens  ([N] docs)
   Board + API:    ~[board_chars ÷ 4] tokens
   Skill overhead: ~4,000 tokens
   ─────────────────────────────────
   Total input:    ~[sum] tokens
   Output:         ~[count of cards created × 200 + response length ÷ 4] tokens
```

Note: these are estimates (÷4 heuristic). Actual usage visible in your Anthropic console under Usage. Typical light run: 5–10k tokens. Heavy run with multiple Drive docs: 20–40k tokens.
