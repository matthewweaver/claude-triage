---
description: Generate a concise standup summary from your Jira tickets and Triage Board. Covers what you did, what you're working on, and any blockers. Optionally post to Slack.
---

# /standup

Pulls your Jira tickets and board state to generate a standup-ready summary in seconds. Works for daily standups or weekly check-ins.

---

## Steps

### 0. Read workspace config

Read `skills/triage/references/workspace-config.md` to get board backend, board ID/collection ID, board URL, and Slack channel preferences.

---

### 1. Fetch Jira tickets assigned to you

Call `searchJiraIssuesUsingJql` with:

```
assignee = currentUser() AND updated >= -1d ORDER BY updated DESC
```

Request fields: `summary`, `status`, `priority`, `updated`, `assignee`, `url`, `comment`.

Bucket results:
- **Done recently** (status changed to Done / Closed / Resolved in the last 24 hours)
- **In progress** (status = In Progress / In Review / In Development)
- **To do** (status = To Do / Backlog / Open — only surface if updated in the last 24 hours or high priority)
- **Blocked** (tickets with "Blocked" label, a linked blocker issue, or no status change in 5+ days while In Progress)

---

### 2. Load the board

Use `notion-search` with `data_source_url` set to the configured **Database collection ID** (`collection://...`) and `page_size: 25`, paginating until all pages are retrieved.

Collect:
- **Done today**: Done cards with `Triaged at` date = today
- **Active Tier 1**: current Tier 1 — Reply cards (awaiting reply)
- **Upcoming**: Soon cards with Due date in the next 7 days, sorted by date

---

### 3. Generate standup summary

Combine Jira and board data into a standup summary. Deduplicate — if a Jira ticket and a board card describe the same item, show it once.

Format:

```
📋 Standup — [Day, Date]

✅ Done
- [Jira ticket or board card — concise one-liner]
- [Jira ticket or board card]
- …

🔄 In progress
- [CPE-XXX] [Title] — [one-line status note if available]
- [Board Tier 1 item still awaiting reply]
- …

🚧 Blockers
- [CPE-XXX] [Title] — blocked [N] days, [brief reason if known]
- [Board Tier 1 item] — waiting [N] days, no response
- …

📅 Coming up
- [action] — due [date]
- [Jira ticket in To Do with high priority or upcoming deadline]
```

Keep each line to one sentence. Omit sections if empty. If there are no blockers, omit the blockers section entirely.

---

### 4. Today's meetings

Call `list_events` for today. Filter to events where you are an attendee and have not declined. Exclude all-day events.

List them in time order as a simple schedule:

```
📅 Today
- 09:30 · Weekly sync (45m)
- 11:00 · 1:1 with Baltej (30m)
- 14:00 · Sprint planning (1h)
```

If no meetings: omit the section. If Google Calendar is not connected: skip silently.

---

### 5. Close out

Link to the board at the end of the summary: [Open Triage Board]([BOARD_URL])
