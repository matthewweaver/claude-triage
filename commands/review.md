---
description: Weekly review of your Triage Board — work through stale Tier 1 and Tier 2 cards, check Jira for assigned tickets, and clean up the backlog. Run on Fridays or at the start of the week.
---

# /review

Weekly board sweep: surface overdue cards, prompt decisions on stale items, and pull Jira tickets assigned to you so nothing slips through.

---

## Steps

### 0. Read workspace config

Read `skills/triage/references/workspace-config.md` to get board backend, board ID/collection ID, and board URL.

---

### 1. Load the board

Use `notion-search` with `data_source_url` set to the configured **Database collection ID** (`collection://...`) and `page_size: 25`, paginating until all pages are retrieved. Do NOT use `notion-fetch` on the database ID — that returns schema only.

Group results by Status. Build:
- **Stale Tier 1 list**: Tier 1 — Reply cards with `Triaged at` date more than 3 days ago (overdue — no reply sent)
- **Stale Tier 2 list**: Tier 2 — Review cards with `Triaged at` date more than 5 days ago
- **This Week list**: This Week cards, sorted by Due date ascending
- **Done this week**: Done cards triaged or updated in the last 7 days

---

### 2. Fetch Jira tickets assigned to you

Call `searchJiraIssuesUsingJql` with:

```
assignee = currentUser() AND statusCategory != Done ORDER BY updated DESC
```

Request fields: `summary`, `status`, `priority`, `updated`, `assignee`, `url`.

For each ticket, check whether a card already exists on the board (match by Jira ticket URL in the Link field). If no card exists, flag it as a potential gap.

Bucket the results:
- **In progress**: status = In Progress / In Review / In Development
- **To do / Backlog**: status = To Do / Backlog / Open
- **Blocked**: tickets with a "Blocked" label or linked blocker issue

Report after fetching: "Found [N] Jira tickets assigned to you — [N] in progress, [N] to do, [N] blocked."

---

### 3. Board sweep

For each **stale Tier 1** card (overdue, no reply sent):

```
AskUserQuestion: "[Card name] has been in Tier 1 for [N] days. What would you like to do?"
Options: Draft reply now / Snooze 3 days / Snooze 1 week / Mark Done / Dismiss (move to Backlog)
```

For each **stale Tier 2** card (5+ days, no action):

```
AskUserQuestion: "[Card name] — reviewed this week?"
Options: Yes, mark Done / Still relevant — keep / Snooze 1 week / Dismiss
```

For each **Jira gap** (assigned ticket with no board card):

```
AskUserQuestion: "[Ticket ID] [Title] — no board card exists. Add one?"
Options: Add as Tier 1 / Add as Tier 2 / Add as This Week / Skip
```

If adding a card, create it via `notion-create-pages` using the Jira ticket URL as the Link and today's date as Triaged at.

Process choices:
- **Draft reply now** → run reply drafting flow from `/triage` step 8
- **Snooze N days** → set Status = `Backlog`, set Due date = today + N days
- **Mark Done** → set Status = `Done`
- **Dismiss** → set Status = `Backlog`, clear Due date

---

### 4. Weekly meeting summary

Call `list_events` for the past 7 days (from last Monday to today). Filter to events where you are an attendee (not just organiser). Exclude all-day events and declined events.

Compute:
- **Total meeting time**: sum of event durations in hours
- **Meeting count**: total distinct meetings
- **Recurring vs one-off**: split by whether the event has a recurrence rule
- **Busiest day**: day with the most meeting time
- **Largest blocks of focus time**: gaps of 90+ minutes between meetings, per day — count how many you had across the week
- **1:1s this week**: events with exactly 2 attendees

Format as a compact summary:

```
📅 Meetings this week

[N] meetings — [Xh Ym] total
[N] recurring · [N] one-off
Busiest day: [day] ([Xh])
Focus blocks (90m+): [N]
1:1s: [N]
```

If Google Calendar is not connected, skip this section silently.

---

### 5. Report

```
🔍 Weekly review — [date]

Jira: [N] tickets assigned ([N] in progress, [N] to do, [N] blocked)
Board gaps: [N] Jira tickets with no triage card

Board sweep:
- [N] stale Tier 1 cards resolved
- [N] stale Tier 2 cards resolved
- [N] cards snoozed
- [N] cards kept

📅 Meetings this week
[meeting summary from step 4]

[Open Triage Board]([BOARD_URL])
```

Remind: "Run `/triage` to pick up anything new, or `/standup` to generate your standup summary."
