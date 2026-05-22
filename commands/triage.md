---
description: Read your Slack, Jira, Gmail and Google Drive, classify everything into tiers, and update your Monday.com board. Tier 1 items are numbered so you can say "draft reply to #2".
---

# /triage

Reads all your sources, classifies messages into three tiers, creates board cards for Tier 1 and Tier 2, and summarises Tier 3 noise inline. Tier 1 items are numbered for reply drafting.

## Before you start

Read `skills/triage/references/workspace-config.md`. It contains your personal board ID, group IDs, column IDs, Slack user ID, Jira channel ID, channel priorities, and key people. All API calls below use values from that file.

If workspace-config.md is missing or empty: "Run `/triage setup` first to configure your workspace."

## Board structure reference

Your board has these groups and columns (exact IDs are in workspace-config.md):

| Group | Purpose |
|---|---|
| P0 — Fire 🔥 | Critical production incidents |
| Today | Items due or promoted today |
| Tier 1 — Reply | Needs your response |
| Tier 2 — Review | Needs your eyes, no reply required |
| Soon | Upcoming items |
| Backlog | Low priority |
| Done | Completed |

Columns:
- **Status/Kanban** — labels: `Tier 1 — Reply`, `Tier 2 — Review`, `Today`, `Soon`, `Done`
- **Urgency** — labels: `Critical 🔥`, `High`, `Medium`, `Low`
- **Source** — labels: `DM`, `@mention`, `Thread reply`
- **Channel** (text)
- **Link** (link)
- **Triaged at** (date)
- **Due date** (date)

---

## Steps

### 1. Load config and board

Read `skills/triage/references/workspace-config.md` to get:
- `BOARD_ID`, `BOARD_URL`
- Group IDs: `GROUP_P0`, `GROUP_TODAY`, `GROUP_TIER1`, `GROUP_TIER2`, `GROUP_SOON`, `GROUP_BACKLOG`, `GROUP_DONE`
- Column IDs: `COL_STATUS`, `COL_URGENCY`, `COL_SOURCE`, `COL_CHANNEL`, `COL_LINK`, `COL_TRIAGED_AT`, `COL_DUE`
- `MY_SLACK_ID`, `JIRA_CHANNEL_ID`, `JIRA_BOT_ID`

Call `get_board_items_page` on `BOARD_ID` with `includeColumns: true`, fetching `COL_TRIAGED_AT`, `COL_DUE`, and `COL_LINK`. Collect all existing link URLs as an exclusion set.

- **Time window**: most recent "Triaged at" date on any card → use as the fetch start. Default to last 24 hours if board is empty.

If board not found: "Triage Board not found — run `/triage setup` first."

### 2. Promote due items to Today

Find items in **Soon** (`GROUP_SOON`) or **Backlog** (`GROUP_BACKLOG`) with a "Due date" on or before today. For each, `move_item_to_group` → `GROUP_TODAY` and set `COL_STATUS` to `{"label": "Today"}`.

### 3. Clear Done items

Find all items where Status = "Done" or in group `GROUP_DONE`. For each, `delete_item`. Add their links to the exclusion set.

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

**C. Gmail (unread inbox, no Jira/Confluence)**

`search_threads` with `is:unread in:inbox after:YYYY/MM/DD -from:jira -subject:[JIRA] -from:confluence`. Skip newsletters, marketing, bulk emails.

For **meeting recording emails**: `get_thread` → extract actions → create Soon cards with due dates (see step D logic).

For other direct emails: classify by recipient count and content.

**D. Google Drive — meeting notes and transcripts**

`list_recent_files` (recency, pageSize 20). Filter to docs modified since the triage window. Identify meeting docs:
- Title contains "Transcript" + a date
- Title starts with "Notes –"

Skip any whose view URL is in the exclusion set. For each, `read_file_content` → extract action items assigned to you or unowned → create one **Soon** card per action with:
- Group: `GROUP_SOON`
- Status: `{"label": "Soon"}`
- Urgency: `{"label": "Medium"}`
- Due date: deadline from text if found, else 7 days from today
- Link: Google Doc view URL

---

### 5. Classify each message

Apply `skills/triage/SKILL.md` rules. Assign:
- **Tier 1 — Reply**: someone is waiting; create a board card
- **Tier 2 — Review**: needs eyes, no reply required; create a board card
- **Tier 3 — Noise**: do NOT create a card; collect for inline summary

Assign an **Urgency** for all Tier 1 and Tier 2 items: Critical 🔥 / High / Medium / Low.

---

### 6. Create board cards (Tier 1 and Tier 2 only)

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
> [Open your Triage Board](BOARD_URL)

If nothing actionable: "All clear — no new actionable messages."

---

### 8. Token usage estimate

After completing the run, estimate token consumption based on content volume.

Track these counts during the run:
- `slack_chars`: total characters in all Slack messages read (DMs, @mentions, Jira channel)
- `email_chars`: total characters in all email snippets and thread bodies fetched
- `drive_chars`: total characters in all Google Doc content read
- `board_chars`: total characters in all Monday.com API responses
- `skill_overhead`: fixed ~4,000 tokens (skill instructions + classification logic loaded each run)

Print at the end of every triage summary:

```
📊 Token estimate (this run)
   Slack:         ~[slack_chars ÷ 4] tokens  ([N] messages)
   Jira:          ~[jira_chars ÷ 4] tokens   ([N] notifications)
   Gmail:         ~[email_chars ÷ 4] tokens  ([N] threads)
   Drive:         ~[drive_chars ÷ 4] tokens  ([N] docs)
   Board + API:   ~[board_chars ÷ 4] tokens
   Skill overhead: ~4,000 tokens
   ─────────────────────────────────
   Total input:   ~[sum] tokens
   Output:        ~[count of cards created × 200 + response length ÷ 4] tokens
```

Note: these are estimates (÷4 heuristic). Actual usage visible in your Anthropic console under Usage. Typical light run: 5–10k tokens. Heavy run with multiple Drive docs: 20–40k tokens.
