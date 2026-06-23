---
description: Read your Slack, Jira, Gmail, Google Drive and GitHub, classify everything into tiers, and update your Notion board. Tier 1 items are numbered so you can say "draft reply to #2".
---

# /triage

Reads all your sources, classifies messages into three tiers, creates board cards for Tier 1 and Tier 2, and summarises Tier 3 noise inline. Tier 1 items are numbered for reply drafting.

---

## Steps

### 0. Read workspace config

Read `skills/triage/references/workspace-config.md`. Identify the **board backend** (`monday` or `notion`) and the **Board ID** / **Collection ID**. All board operations below branch on this value.

If workspace-config.md is missing or empty: "Run `/triage setup` first to configure your workspace."

**Triage window — precise timestamp:**
Read `skills/triage/references/last-run-timestamp.md`. It contains a single ISO 8601 timestamp (e.g. `2026-06-17T05:49:00Z`) representing when the last triage run started. Use this as the triage window start for all source fetches this run. If the file is missing or empty, default to 24 hours ago.

Record the current UTC time now (before fetching anything) and write it to `skills/triage/references/last-run-timestamp.md` at the **end** of the run (step 7), so the next run uses an accurate start time.

**Exclusion set:** Read `skills/triage/references/exclusion-set.md` if it exists. This file contains one link URL per line — every Slack message, Jira ticket, email, Drive doc, or GitHub PR that has already been triaged into a board card. Load these into an in-memory set at startup. This is the primary deduplication mechanism for all message types. Do NOT add reminder links to this file — reminders use a live board check instead (see Step 4F), so a deleted reminder card will correctly re-fire on the next run.

**Plugin update check:** Read the `version` field from `.claude-plugin/plugin.json`. Fetch `https://raw.githubusercontent.com/captify-mweaver/claude-triage/main/.claude-plugin/plugin.json` and compare. If the remote version is newer, append a note at the end of the triage report: "⬆️ Plugin update available (vX.Y.Z → vA.B.C) — run `/export` after pulling the latest from git."

---

### 1. Verify board config

Confirm the board backend and Board ID / Collection ID are present in workspace-config.md. If missing: "Triage Board not found — run `/triage setup` first."

No board pre-load is needed. Deduplication is handled by `exclusion-set.md` (step 0). Targeted board queries happen in steps 2 and 3.

---

### 2. Promote due items to Today

Query for overdue items:

```sql
SELECT id, Name FROM "collection://261f9847-2f3c-4166-a939-e001bb013b0a"
WHERE Status IN ('Backlog', 'This Week')
AND "date:Due date:start" <= DATE('now')
```

For each result, call `notion-update-page` setting Status = `Today`.

---

### 3. Build exclusion set from Done items (safety net)

This catches any manually-created Done cards whose links were never written to `exclusion-set.md`.


```sql
SELECT Link FROM "collection://261f9847-2f3c-4166-a939-e001bb013b0a"
WHERE Status = 'Done' AND Link IS NOT NULL
```

Merge results into the in-memory exclusion set for this run only (do not write to file — `exclusion-set.md` is append-only at the end of the run in step 7).

---

### 4. Fetch all sources

Run all five fetches in parallel using the triage window from step 0.

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

⚠️ **Date validation (critical):** Each Jira bot message ends with a timestamp in brackets, e.g. `[2025-07-21 14:52:36 BST]`. This is the date the Jira activity actually occurred — NOT when Slack delivered it. Before classifying a message, extract this date and compare it to the triage window start. **Skip any message whose activity date is before the triage window**, even if the Slack message itself was delivered recently.

Use tier/urgency rules from `workspace-config.md` Jira filters. Set **Channel** = `Jira`, use the Jira ticket URL as the link.

**C. Gmail (unread inbox, no Jira/Confluence) — snippet-first approach**

`search_threads` with `is:unread in:inbox after:YYYY/MM/DD -from:jira -subject:[JIRA] -from:confluence`.

> ⚠️ High-volume warning: if the query returns 50+ threads, report the count and ask "That's a lot — want me to triage all of them, or just the last 24 hours?" before proceeding.

**Classify from snippet first** — do NOT call `get_thread` for every email:
- Read subject, sender, snippet, and recipient count from the search results
- Auto-Tier 3: marketing, newsletters, bulk sends (10+ recipients), social notifications
- Auto-Tier 1: time-sensitive language ("urgent", "by EOD", "ASAP"), invoices, fraud/security alerts, replies in threads you started
- For ambiguous Tier 1 and Tier 2 candidates only: call `get_thread` to read the full body and confirm classification

For **meeting recording emails**: `get_thread` → extract action items → create This Week cards (see step 6 logic). Scan each action item for due date mentions and set `date:Due date:start` if found; leave blank otherwise.

Skip newsletters, marketing, bulk emails.

**D. Google Drive — meeting notes and transcripts**

`list_recent_files` (recency, pageSize 20). Filter to docs modified since the triage window. Identify meeting docs:
- Title contains "Transcript" + a date
- Title starts with "Notes –"

Skip any whose view URL is in the exclusion set. For each, `download_file_content` with `exportMimeType: text/plain` → decode content → extract action items assigned to you or unowned → create one **This Week** card per action.

When extracting action items, always scan for due date mentions (e.g. "by Friday", "before June 4", "EOD Thursday", "next week"). Convert any relative dates to an absolute ISO date using today's date. Set this as the `date:Due date:start` on the card. If no due date is mentioned, leave the Due date blank.

**E. GitHub — PRs awaiting your review**

If `GITHUB_USERNAME` is set in workspace-config, search for open PRs where you are a requested reviewer:

Fetch `https://api.github.com/search/issues?q=is:open+is:pr+review-requested:[GITHUB_USERNAME]` using the GitHub connector or web fetch.

For each PR:
- Skip if the PR URL is in the exclusion set
- Skip if you've already submitted a review (check `reviewed-by` in the query results)
- Classify as **Tier 1** if the PR has been waiting for your review for more than 24 hours, or if the PR description contains urgent language
- Classify as **Tier 2** if opened within the last 24 hours
- Set **Channel** = `GitHub`, **Source** = `@mention`, use the PR URL as the link

If GitHub is not configured: skip silently.

**F. Scheduled reminders**

Read `skills/triage/references/reminders.md`. For each row in the **Active Reminders** table:

1. Parse the `Schedule` field (day + BST time).
2. Determine whether that day + time slot falls within the current triage window (last triaged → now). Use today's date and the current BST time to evaluate day-of-week and hour. For `biweekly`, check whether the current ISO week number is even. For `monthly-first-monday`, check whether today is the first Monday of the month.
3. If the slot is due, do a **live board check** before creating using an exact SQL lookup:

   ```sql
   SELECT Name, Status FROM "collection://261f9847-2f3c-4166-a939-e001bb013b0a"
   WHERE Link = 'reminder://[ID]'
   ```

   If any row is returned, that reminder card is still active on the board — skip it silently.

   If no match is found, create the card:
   - Classify as **Tier 1 — Reply**, Urgency = **Medium**
   - Name: the reminder `Title`
   - Channel: `Reminder`
   - Source: `@mention`
   - Link: `reminder://[ID]` (no date — a stable identifier so the live check works across days)
   - Card body: the `Notes` value (if any), prefixed with `⏰ Scheduled reminder.`
   - Set `date:Triaged at:start` = today

**Do NOT add reminder links to `exclusion-set.md`.** The live board check is intentional: if you delete the card, the next run will see nothing on the board and correctly create a fresh one.

---

### 5. Classify each message

Apply `skills/triage/SKILL.md` rules. Assign:
- **Tier 1 — Reply**: someone is waiting; create a board card
- **Tier 2 — Review**: needs eyes, no reply required; create a board card
- **Tier 3 — Noise**: do NOT create a card; collect for inline summary

Assign an **Urgency** for all Tier 1 and Tier 2 items: Critical 🔥 / High / Medium / Low.

---

### 6. Create or update board cards (Tier 1 and Tier 2 only)

**Deduplication — check before creating:**

**For message/ticket/doc cards** (Slack, Jira, Gmail, Drive, GitHub): check `exclusion-set.md` (loaded in step 0) for the Link URL. If present, skip — this item has already been triaged. Do not re-raise it even if the board card was later deleted.

**For reminder cards**: use the live SQL check from step 4F only. Do not check `exclusion-set.md`.

Use `notion-create-pages` with `data_source_id` from workspace-config. Properties:

**Assign a Card ID** for each new card: generate a random 4-digit number (1000–9999). Store the session number → Card ID mapping for the report.

| Field | Notion property | Value |
|---|---|---|
| Name | title | Concise one-line summary |
| Status | select | `Tier 1 — Reply` / `Tier 2 — Review` / `Today` / `On Hold` / `Backlog` / `This Week` |
| Group | select | `Tier 1 — Reply` / `Tier 2 — Review` / `P0 — Fire 🔥` / `Today` / `On Hold` / `Backlog` |
| Urgency | select | `Critical 🔥` / `High` / `Medium` / `Low` |
| Source | select | `DM` / `@mention` / `Thread reply` |
| Channel | rich_text | Channel or sender name |
| Link | url | Permalink URL |
| Card ID | rich_text | The 4-digit ID assigned above (e.g. `1042`) |
| Triaged at | date | `date:Triaged at:start` = today's date (ISO 8601) |
| Due date | date | `date:Due date:start` = from text only (This Week cards only) |

**P0 / Critical**: set Group = `P0 — Fire 🔥`, Status = `P0 — Fire 🔥`.

**Tier 1 cards — draft reply in card body:**

For every Tier 1 card, fetch the full thread context if not already retrieved (call `slack_read_thread` for Slack items, `get_thread` for email items), then write a suggested draft reply into the card's `content` field using Notion markdown.

Use the voice rules from `workspace-config.md`:
- **Slack items**: casual, short (1–3 sentences), lead with the answer
- **Email items**: follow `## Email Voice` from workspace-config — tone, length, sign-off

**Calendar awareness:** If the message involves scheduling (meeting request, "are you free", "when can we", availability question), fetch the user's calendar for the next 5 working days using the Google Calendar connector (if available). Include 2–3 suggested time slots in the draft reply. If calendar is not connected, note "Check your calendar for availability" in the draft.

Format the card body as:

```
## 💬 Suggested reply

[draft reply text, including availability slots if scheduling-related]

---
## 🧵 Context

[1–3 sentence summary of what the message is asking and any relevant background]
```

Do not pre-fill a draft for Tier 2 or This Week cards — body left empty.

---

### 7. Report back

**Tier 1 items are numbered** so you can request a reply draft by number.

Format:

> Triaged [N] messages across Slack, Jira, Gmail, Drive and GitHub.
> [Y] items promoted to Today.
>
> **Tier 1 — Reply** ([count])
> 1. `[1042]` Francisco in #team-channel asking about X — [Urgency]
> 2. `[2317]` Jira: CPE-229 assigned to you — [Urgency]
> 3. `[5891]` ↑ PR #412 upgraded from Tier 2 — waiting 2 days — [Urgency]
>
> **Tier 2 — Review** ([count])
> - `[4403]` Bhagyashri shared the Hypothesis ticket update
> - …
>
> **Tier 3 — Noise** ([count] skipped)
> - 12 GitHub bot notifications
> - 8 Google Calendar reminders
> - 4 messages in #random
> - 3 Confluence digest emails
>
> Each Tier 1 card already contains a suggested reply — open the card in Notion to review it.
> Say "send reply to #2" or "send reply to 1042" to send the draft (session number or Card ID both work).
> Say "redraft #2 [shorter|more formal|add context]" to adjust tone.
> Say "snooze #2 until [date]" to defer an item. Card IDs persist across sessions so you can snooze later.
> Say "archive the noise" to archive the Tier 3 emails (requires explicit confirmation).
> [Open your Triage Board]([BOARD_URL])

If nothing actionable: "All clear — no new actionable messages."

**After generating the report**, write the timestamp recorded at the start of this run (step 0) to `skills/triage/references/last-run-timestamp.md` as a single ISO 8601 UTC string (e.g. `2026-06-17T09:05:00Z`). This ensures the next run's triage window starts from the correct point.

**Append to exclusion-set.md:** For every message/ticket/doc card created this run (Slack, Jira, Gmail, Drive, GitHub — not reminders), append the Link URL to `skills/triage/references/exclusion-set.md`, one URL per line. Create the file if it doesn't exist. This prevents the same item ever resurfacing in a future triage run, even if the board card is deleted.

---

### 7a. Archive offer (only if triggered by user)

If the user says "archive the noise" or similar:

1. List the Tier 3 emails you would archive (subject, sender).
2. Ask for explicit confirmation: "Archive these [N] emails? This cannot be undone." — use `AskUserQuestion`.
3. Only proceed after confirmation. Never auto-archive.

---

### 8. Sending, redrafting and snoozing

**Card references:** All actions accept either a session number (`#2`) or a persistent 4-digit Card ID (`1042`). When a 4-digit ID is given, look up the card by filtering the board for `Card ID = 1042` — use the session number→ID map if still in the same session, or call `notion-fetch` with a filter if in a fresh session.

**"send reply to #N" / "send reply to 1042"** — retrieve the draft from the Notion card body, show it in chat for final review, then send it:
- Slack items: `slack_send_message` as a thread reply, then set card Status to `Done` via `notion-update-page`
- Email items: `create_draft` with subject "Re: [original subject]", then update the card Status to `Done`

**"redraft #N"** or **"draft reply to #N"** — generate a fresh draft in chat:
- Fetch the full thread again, write a new draft using the voice rules below
- Show in chat, wait for confirmation, then send or update the Notion card body via `notion-update-page`

**"redraft #N [modifier]"** — regenerate with a specific tone adjustment. Supported modifiers:
- `shorter` — cut to 1–2 sentences, remove any preamble
- `more formal` — professional register, no contractions, full sign-off
- `more casual` — conversational, first name, drop formality
- `add context` — expand with relevant background before the answer
- `add availability` — fetch calendar and append free slot suggestions

Apply the modifier and show the revised draft. Wait for confirmation before sending.

**"snooze #N until [date/day]"** / **"snooze 1042 until [date/day]"** — defer the item without losing it:
- Parse the date (e.g. "Monday", "next week", "June 10")
- Look up the card by session number or 4-digit Card ID
- Update the card: set Status = `Backlog`, set `date:Due date:start` = the parsed ISO date
- The item will auto-promote to Today when that date arrives (step 2)
- Confirm: "Snoozed — #N will resurface on [date]."

---

### 9. Token usage estimate

After completing the run, estimate token consumption based on content volume.

Track these counts during the run:
- `slack_chars`: total characters in all Slack messages read
- `jira_chars`: total characters in all Jira notification messages parsed
- `email_snippet_chars`: characters in email snippets classified without `get_thread`
- `email_full_chars`: characters in full email threads fetched via `get_thread`
- `drive_chars`: total characters in all Google Doc content read
- `github_chars`: total characters in GitHub PR data fetched
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
   GitHub:         ~[github_chars ÷ 4] tokens  ([N] PRs)
   Board + API:    ~[board_chars ÷ 4] tokens
   Skill overhead: ~4,000 tokens
   ─────────────────────────────────
   Total input:    ~[sum] tokens
   Output:         ~[count of cards created × 200 + response length ÷ 4] tokens
```

Note: these are estimates (÷4 heuristic). Actual usage visible in your Anthropic console under Usage.
