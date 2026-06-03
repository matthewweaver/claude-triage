---
description: One-time interactive setup — walks you through connecting your tools and configuring your triage preferences, then saves everything to workspace-config.md so /triage is personalised from the first run.
---

# /triage setup

This is an interactive wizard. Work through each section in order. Ask one section at a time — don't dump all questions at once. Use `AskUserQuestion` for every question that needs a choice or list input.

---

## Section 0 — Welcome

Greet the user and explain what setup does:

> "I'll walk you through setting up your triage system. We'll connect your tools (Slack, Jira, Gmail, Google Drive) and configure which channels and people matter most to you. This saves to a config file so every future `/triage` run is personalised. Takes about 5 minutes."

Then proceed section by section.

---

## Section 1 — Board Backend

Ask:

```
AskUserQuestion: "Which Kanban board would you like to use for triage?"
Options: Monday.com / Notion
```

### If Monday.com

Ask:

```
AskUserQuestion: "Do you have an existing Triage Board on Monday.com, or should I create one?"
Options: Create a new board for me / I have an existing board
```

**If creating a new board:**

Call `create_board` with name "Triage Board". Then create the required groups and columns:

Groups to create (in order):
1. "P0 — Fire 🔥"
2. "Today"
3. "Tier 1 — Reply"
4. "Tier 2 — Review"
5. "Soon"
6. "Backlog"
7. "Done"

Columns to create on the board:
- Status column titled "Status" with labels: `Tier 1 — Reply`, `Tier 2 — Review`, `Today`, `Soon`, `Done`
- Status column titled "Urgency" with labels: `Critical 🔥`, `High`, `Medium`, `Low`
- Status column titled "Source" with labels: `DM`, `@mention`, `Thread reply`
- Text column titled "Channel"
- Link column titled "Link"
- Date column titled "Triaged at"
- Date column titled "Due date"

After creation, call `get_board_info` on the new board to capture all group IDs and column IDs. Save the board ID and URL. Confirm: "✅ Triage Board created."

**If existing board:**

Ask: "What's your board ID or URL?" Extract the board ID. Call `get_board_info` to capture all group IDs and column IDs. Confirm: "✅ Triage Board connected."

If Monday.com isn't connected: "Monday.com isn't connected yet. Go to Customize → Connectors, connect Monday.com, then run `/triage setup` again."

Save to workspace-config.md:
- `board_backend: monday`
- All board/group/column IDs

### If Notion

Check if the Notion connector is available (try `notion-search` with a simple query).

- **If not connected**: "Notion isn't connected yet. Go to Customize → Connectors, connect Notion, then run `/triage setup` again."
- **If connected**: "✅ Notion is connected."

Ask:

```
AskUserQuestion: "Do you have an existing Triage database in Notion, or should I create one?"
Options: Create a new database for me / I have an existing database
```

**If creating a new database:**

Call `notion-create-database` with name "Triage Board" and these properties:
- Title property: `Name`
- Select property: `Status` — options: `Tier 1 — Reply`, `Tier 2 — Review`, `Today`, `Soon`, `Backlog`, `Done`, `P0 — Fire 🔥`
- Select property: `Group` — same options as Status
- Select property: `Urgency` — options: `Critical 🔥`, `High`, `Medium`, `Low`
- Select property: `Source` — options: `DM`, `@mention`, `Thread reply`
- Rich text property: `Channel`
- URL property: `Link`
- Date property: `Triaged at`
- Date property: `Due date`

After creation, call `notion-fetch` on the new database URL to get the `data_source_id` (collection ID). Confirm: "✅ Triage Board created in Notion."

**If existing database:**

Ask: "What's your Notion database URL or ID?" Call `notion-fetch` on it to get the `data_source_id`. Confirm: "✅ Triage Board connected."

Save to workspace-config.md:
- `board_backend: notion`
- `Board ID`: the database ID
- `Board URL`: the Notion database URL
- `Database collection ID`: the `data_source_id` from notion-fetch

---

## Section 2 — Slack

Ask:

```
AskUserQuestion: "Would you like to connect Slack for triage?"
Options: Yes / No, skip Slack
```

**If No**: skip to Section 3.

**If Yes**:

Check if the Slack connector is available (try `slack_search_users` with the user's name).

- **If not connected**: "Open Customize → Connectors, enable the Slack connector, then come back and say 'continue setup'."
- **If connected**: "✅ Slack is connected."

Look up the Slack user ID for the person running setup (use `slack_search_users` with their name or email). Save as `MY_SLACK_ID`.

Then ask:

```
AskUserQuestion: "What are your priority Slack channels — ones where messages often need your attention? List them separated by commas (e.g. #team-eng, #incidents)"
Free text input
```

```
AskUserQuestion: "Any watch channels — ones you want to stay aware of but rarely act on? (or skip)"
Free text input
```

```
AskUserQuestion: "Any channels that are mostly noise you'd like summarised as counts only? (or skip)"
Free text input
```

```
AskUserQuestion: "Who are your key people in Slack — whose DMs always get top priority? List names or roles (e.g. 'my manager Sarah, direct reports')"
Free text input
```

---

## Section 3 — Jira (via Slack)

Ask:

```
AskUserQuestion: "Do you use the Jira Slack app for notifications?"
Options: Yes / No, skip Jira
```

**If Yes**:

Tell the user: "I'll find your Jira DM channel automatically." Search for a DM from a Jira bot (user `UAU3930NM`) using `slack_search_public_and_private` with `from:Jira`. Extract the channel ID and bot user ID from the first result. Save as `JIRA_CHANNEL_ID` and `JIRA_BOT_ID`.

- If found: "✅ Found your Jira notifications channel."
- If not found: Ask "I couldn't find it automatically — what's the name of the Jira bot DM in your Slack sidebar? (Usually just 'Jira')" and attempt to find the channel ID from `slack_search_channels`.

Ask:

```
AskUserQuestion: "Which Jira notifications do you want as Tier 1 (Reply)? Select all that apply."
Multi-select: Assigned to me / Mentioned in comment / Ticket blocked/waiting / All of the above
```

---

## Section 4 — Gmail

Ask:

```
AskUserQuestion: "Would you like to connect Gmail to triage emails?"
Options: Yes / No, skip Gmail
```

**If Yes**:

Check if Gmail tools are available (try `search_threads` with a simple query).

- **If not connected**: "Open Customize → Connectors, enable the Gmail connector, then say 'continue setup'."
- **If connected**: "✅ Gmail is connected."

Ask:

```
AskUserQuestion: "Should I include Google Meet recording emails and extract action items from them?"
Options: Yes, extract actions / Yes, but just add as a card / No, skip meeting emails
```

---

## Section 5 — Google Drive

Ask:

```
AskUserQuestion: "Would you like to connect Google Drive to extract actions from meeting notes docs?"
Options: Yes / No, skip Drive
```

**If Yes**:

Check if Drive tools are available (try `list_recent_files`).

- **If not connected**: "Open Customize → Connectors, enable the Google Drive connector, then say 'continue setup'."
- **If connected**: "✅ Google Drive is connected."

---

## Section 6 — Triage preferences

Ask:

```
AskUserQuestion: "How far back should each triage run look by default?"
Options: Since last run (recommended) / Last 24 hours / Last 48 hours / Last week
```

Ask:

```
AskUserQuestion: "For meeting actions with no stated deadline, what default due date should I use?"
Options: 3 days / 7 days (recommended) / 14 days / No due date
```

---

## Section 6b — Email Voice

Ask:

```
AskUserQuestion: "When I draft email replies for you, what tone should I use?"
Options: Direct and concise (lead with the answer) / Warm and conversational / Formal and professional / Same as Slack (casual)
```

Ask:

```
AskUserQuestion: "How should I sign off emails?"
Free text input (e.g. "Matt", "Thanks, Matt", "Best, Matthew")
```

Save the style and sign-off to workspace-config under `## Email Voice`.

---

## Section 7 — Write workspace-config.md

Using all the answers collected, write a fully populated `skills/triage/references/workspace-config.md` file. Use this structure:

```markdown
# Workspace Config — [Name]

## Identity
- **Slack display name**: [name]
- **Slack user ID**: [discovered user ID]

## Board Backend
- **Backend**: [monday | notion]
- **Board ID**: [board ID]
- **Board URL**: [board URL]
[If notion, also include:]
- **Database collection ID**: [data_source_id from notion-fetch]

## Monday.com Board
[Include this section only if backend = monday]

### Group IDs
| Group | ID |
|---|---|
| P0 — Fire 🔥 | [id] |
| Today | [id] |
| Tier 1 — Reply | [id] |
| Tier 2 — Review | [id] |
| Soon | [id] |
| Backlog | [id] |
| Done | [id] |

### Column IDs
| Column | ID |
|---|---|
| Status | [id] |
| Urgency | [id] |
| Source | [id] |
| Channel | [id] |
| Link | [id] |
| Triaged at | [id] |
| Due date | [id] |

## Priority Channels
[table of channels and notes from user's answers]

## Watch Channels
[table or "None configured"]

## Muted Channels (Tier 3 by default)
[table or "None configured"]

## Key People
[table of people/roles and relationship]

## Jira Bot Filters
- **Jira channel ID**: [discovered channel ID or "Not configured"]
- **Jira bot user ID**: [bot user ID or "Not configured"]

| Action | Default Tier | Urgency |
|---|---|---|
| Assigned to me | Tier 1 | High |
| Mentioned in comment | Tier 1 | High |
[customised based on their Section 3 selections]
| Transitioned → Done | Tier 3 | — |
| General status transitions | Tier 3 | — |

## Gmail Settings
- **Meeting recordings**: [their choice from Section 4]

## Google Drive Settings
- **Meeting notes extraction**: [Enabled / Disabled]
- **Doc patterns**: Title contains "Transcript" + date, or title starts with "Notes –"

## Email Voice
- [Tone — e.g. Direct and concise, warm but professional]
- [Length — e.g. Short, 2–4 sentences for most replies]
- [Norms — e.g. First names always, no filler phrases]
- Sign off: [e.g. "Matt" or "Thanks, Matt"]

## Default Preferences
- **Triage window**: [their choice]
- **Default due date for meeting actions**: [their choice]
```

After writing the file, confirm: "✅ Your workspace config has been saved."

---

## Section 8 — Scheduled Triage

Ask:

```
AskUserQuestion: "Would you like triage to run automatically on a schedule?"
Options: Yes, set a schedule / No, I'll run it manually
```

**If No**: skip to Section 9.

**If Yes**:

```
AskUserQuestion: "How often should triage run?"
Options: Every weekday morning / Every day (including weekends) / Every Monday morning / Custom schedule
```

If Custom: ask for their preferred time and days as free text, then convert to a cron expression.

Default times: morning runs at 08:00 local time.

**Create or update the scheduled task:**

First, check if a task with `taskId: "triage"` already exists by calling `list_scheduled_tasks`. 

- **If it exists**: use `update_scheduled_task` with `taskId: "triage"` — update the `prompt` (to sync any plugin changes) and `cronExpression` (if they want to change the schedule). Tell the user: "✅ Scheduled triage task updated."
- **If it doesn't exist**: use `create_scheduled_task` with the values below. Tell the user: "✅ Scheduled triage task created."

Task values:
- `taskId`: `"triage"`
- `description`: `"Run triage across Slack, Jira, Gmail and Drive and update the Kanban board"`
- `cronExpression`: per their choice above (e.g. `"0 8 * * 1-5"` for weekdays at 8am)
- `prompt`: construct a **fully self-contained** version of the triage instructions by reading `commands/triage.md` and inlining the current `workspace-config.md` values. The prompt must not rely on reading files at runtime, since scheduled tasks start fresh. Include:
  - All triage steps from `commands/triage.md`
  - The user's board backend, board ID, and all column/group IDs or Notion collection ID
  - The user's Slack ID, Jira channel ID, channel priorities, key people
  - Gmail and Drive settings
  - The classification rules from `skills/triage/SKILL.md`

This means re-running setup (e.g. after a plugin update) will also refresh the scheduled task prompt with any updated instructions.

---

## Section 9 — Done

Tell the user:

> "You're all set up. Here's what's configured:
> - [list which connectors are active]
> - [N] priority channels, [N] watch channels
> - [N] key people
> - Scheduled triage: [schedule description, or "manual only"]
>
> Run `/triage` to do your first triage pass. You can edit your preferences at any time by editing `workspace-config.md`, or re-run `/triage setup` to reconfigure from scratch."

If any connectors were skipped, remind them they can re-run `/triage setup` after connecting them.
