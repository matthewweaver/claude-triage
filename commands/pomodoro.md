---
description: Start a 25-minute Pomodoro focus timer. When the timer ends, triage runs automatically. Use /pomodoro start to begin and /pomodoro stop to cancel.
---

# /pomodoro

A 25-minute focus timer. At the end of each session, triage runs automatically so you surface any new messages without breaking focus during the block.

## Usage

- `/pomodoro` or `/pomodoro start` — start a new 25-minute session
- `/pomodoro stop` — cancel the current session

## Steps

### /pomodoro start

1. **Confirm with the user** before starting:
   > Starting a 25-minute Pomodoro. I'll run triage automatically when it ends. Go do your best work — I'll handle the inbox.

2. **Schedule the triage run** using the schedule skill to trigger `/triage` in 25 minutes (1500 seconds from now).

3. **Track state** by noting the start time in a brief status message. If the user asks "how long left?", calculate and report the remaining time.

4. At the end of the 25 minutes, **run `/triage` automatically** (as if the user had typed it), then report:
   > ⏱ Pomodoro complete! Here's what came in during your focus session:
   > [triage summary]
   >
   > Take a 5-minute break, then type `/pomodoro` again when you're ready for the next session.

### /pomodoro stop

Cancel any pending triage run and tell the user:
> Pomodoro cancelled. Run `/triage` whenever you're ready to check your messages, or `/pomodoro` to start a new session.

## Notes

- Only one Pomodoro session can run at a time. If the user types `/pomodoro start` while one is already running, report how much time is left and ask if they want to restart.
- The triage run at the end is exactly the same as running `/triage` manually — it reads the last 25 minutes of Slack activity (since the Pomodoro started).
- If the triage run at the end finds nothing actionable, keep the message brief: "All clear — nothing new came in. Ready for another session?"
