---
name: export-plugin
description: >
  Re-package this plugin as a shareable .plugin file, safe for git and public sharing.
  Excludes personal workspace-config.md. Use when: export plugin, package plugin,
  build plugin, share plugin, publish plugin, update plugin file, push to git, commit plugin.
---

# /export-plugin

Re-packages this plugin into a distributable `.plugin` file — personal config excluded,
safe to commit or share publicly.

## What gets excluded

- `skills/triage/references/workspace-config.md` — personal config, stays local
- `skills/triage/references/reminders.md` — personal reminders, stays local
- `*.DS_Store`, `__MACOSX` — OS noise
- Any existing `*.plugin` files at the plugin root (regenerated fresh)

These exclusions match `.gitignore` so the git folder and the `.plugin` file stay in sync.

## Steps

### 1 — Find the plugin root

This SKILL.md lives at `skills/export-plugin/SKILL.md` inside the plugin. The plugin root is
two directories up from this file. Read `.claude-plugin/plugin.json` from that root to
get the plugin `name` field.

### 2 — Build the .plugin file

Zip only the known source paths explicitly (avoids capturing `.git`, build artefacts, or personal config):

```bash
cd "PLUGIN_ROOT" && \
zip -r "/tmp/NAME.plugin" \
  .claude-plugin \
  commands \
  skills \
  .mcp.json \
  .gitignore \
  README.md \
  --exclude "*/workspace-config.md" \
  --exclude "*/reminders.md" \
  --exclude "*/.DS_Store" \
  --exclude "*.DS_Store"
```

This allowlist approach is safer than a denylist — `.git/`, personal config, and OS noise are excluded by not being listed, not by pattern matching.

### 3 — Save outputs

Copy `/tmp/NAME.plugin` to:

1. **`PLUGIN_ROOT/NAME.plugin`** — replaces the tracked file so git picks it up
2. **`~/Downloads/NAME.plugin`** — for sharing or drag-and-drop install

If writing to the plugin root fails (read-only install location), save to Downloads only
and tell the user: "Saved to Downloads — copy `NAME.plugin` into your git clone to commit it."

### 4 — Sync the scheduled task

Use `mcp__scheduled-tasks__list_scheduled_tasks` to check whether a task with `taskId = "triage"` exists.

If it exists:
1. Read `commands/triage.md` (the steps), `skills/triage/references/workspace-config.md` (identity + board config), and `skills/triage/references/reminders.md` (reminders table) from the plugin root.
2. Build a new prompt by combining them:
   - Preamble: "You are acting as a Chief of Staff for Matt Weaver (Engineering Manager)…"
   - Inline all identity, board, channel, key people, Jira, Gmail, Drive, email voice fields from workspace-config.md
   - Inline the full Active Reminders tables from reminders.md (with the schedule logic explanation)
   - Steps 1–9 from commands/triage.md, with Monday.com branches removed (backend is Notion)
3. Call `mcp__scheduled-tasks__update_scheduled_task` with `taskId = "triage"` and the compiled `prompt`.

This keeps the scheduled job in sync with every export — no plugin reimport required.

If the `triage` task does not exist in the scheduler, skip silently and note: "No scheduled triage task found — skipped sync."

### 5 — Report

List the files packaged and those excluded. Confirm both output paths. If the scheduled task was synced, note: "✅ Scheduled task SKILL.md updated." Then say:

> "Commit `NAME.plugin` alongside the source files — anyone cloning the repo can open it
> to import the plugin. Run `/export-plugin` again any time you make changes."
