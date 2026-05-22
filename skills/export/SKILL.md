---
name: export
description: >
  Re-package this plugin as a shareable .plugin file, safe for git and public sharing.
  Excludes personal workspace-config.md. Use when: export plugin, package plugin,
  build plugin, share plugin, publish plugin, update plugin file, push to git, commit plugin.
---

# /export

Re-packages this plugin into a distributable `.plugin` file — personal config excluded,
safe to commit or share publicly.

## What gets excluded

- `skills/triage/references/workspace-config.md` — personal config, stays local
- `*.DS_Store`, `__MACOSX` — OS noise
- Any existing `*.plugin` files at the plugin root (regenerated fresh)

These exclusions match `.gitignore` so the git folder and the `.plugin` file stay in sync.

## Steps

### 1 — Find the plugin root

This SKILL.md lives at `skills/export/SKILL.md` inside the plugin. The plugin root is
two directories up from this file. Read `.claude-plugin/plugin.json` from that root to
get the plugin `name` field.

### 2 — Build the .plugin file

Run this bash command, substituting `PLUGIN_ROOT` and `NAME`:

```bash
cd "PLUGIN_ROOT" && \
zip -r "/tmp/NAME.plugin" . \
  --exclude "*/workspace-config.md" \
  --exclude "*/.DS_Store" \
  --exclude "*.DS_Store" \
  --exclude "*/__MACOSX/*" \
  --exclude "*/.git/*" \
  --exclude "*.plugin"
```

### 3 — Save outputs

Copy `/tmp/NAME.plugin` to:

1. **`PLUGIN_ROOT/NAME.plugin`** — replaces the tracked file so git picks it up
2. **`~/Downloads/NAME.plugin`** — for sharing or drag-and-drop install

If writing to the plugin root fails (read-only install location), save to Downloads only
and tell the user: "Saved to Downloads — copy `NAME.plugin` into your git clone to commit it."

### 4 — Report

List the files packaged and those excluded. Confirm both output paths. Then say:

> "Commit `NAME.plugin` alongside the source files — anyone cloning the repo can open it
> to import the plugin. Run `/export` again any time you make changes."
