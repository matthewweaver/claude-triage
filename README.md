# triage

Turn Claude into your Chief of Staff. Monitors Slack, Gmail, Jira, and Google Drive, classifies everything into three tiers, and keeps your Kanban board up to date so you always know what needs your attention.

## What it does

Each `/triage` run:
- Reads your Slack DMs, @mentions, and Jira bot notifications
- Scans your Gmail inbox and extracts actions from Google Meet recordings
- Pulls action items from meeting notes and transcripts in Google Drive
- Classifies everything into **Tier 1 (Reply)**, **Tier 2 (Review)**, or **Tier 3 (Noise)**
- Creates cards on your Notion board for anything actionable
- Summarises noise inline so you can see it without it cluttering your board

Tier 1 items are numbered so you can say "draft reply to #2" and Claude will write it.

## Prerequisites

You'll need the Claude desktop app (Cowork mode) with these connectors enabled under **Customize → Connectors**:

- Notion
- Slack
- Gmail
- Google Drive

You don't need all four — setup will ask which ones you want to use.

## Install

1. Clone this repo:
   ```bash
   git clone https://github.com/matthewweaver/claude-triage.git
   ```

2. Open `triage.plugin` — the Cowork app will detect it and show an install prompt. Click **Install**.

3. Run setup to connect your tools and configure your preferences:
   ```
   /setup
   ```
   This walks you through connecting each service, sets up your Notion board, and saves your personal config. Takes about 5 minutes.

4. Run your first triage:
   ```
   /triage
   ```

## Commands

| Command | What it does |
|---|---|
| `/triage` | Run a full triage pass across all connected sources |
| `/setup` | First-time setup wizard, or reconfigure at any time |
| `/pomodoro` | Start a 25-minute focus timer — triage runs automatically when it ends |
| `/export` | Re-package the plugin as a `.plugin` file for sharing or committing |

## Personal config

Your preferences (Slack user ID, board IDs, priority channels, key people) live in `skills/triage/references/workspace-config.md`. This file is **gitignored** — it stays local to your machine and is never committed.

The file is created automatically by `/setup`. If you want to edit it manually, see `skills/triage/references/workspace-config.template.md` for the full structure.

## Contributing

Since `workspace-config.md` is gitignored, everyone can make changes to the plugin logic without affecting each other's personal settings.

After making changes:

1. Run `/export` — this rebuilds `triage.plugin` from the current source files (excluding your personal config) and saves it to the repo folder
2. Commit both the source files and the updated `triage.plugin`:
   ```bash
   git add .
   git commit -m "describe your change"
   git push
   ```

Others can then pull and reinstall from the updated `.plugin` file.
