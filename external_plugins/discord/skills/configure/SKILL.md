---
name: configure
description: Set up the Discord channel — create sessions, save bot tokens, and review access policy. Use when the user pastes a Discord bot token, asks to configure Discord, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /discord:configure — Discord Channel Setup (Multi-Session)

Each Discord bot runs as a named **session**. Sessions are tracked in
`~/.claude/channels/discord/sessions.json` and each session's state lives in
`~/.claude/channels/discord/sessions/<name>/`.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — list all sessions and overall status

1. Read `~/.claude/channels/discord/sessions.json` (missing file = no sessions).
2. For each session listed:
   - Check `~/.claude/channels/discord/sessions/<name>/.env` for
     `DISCORD_BOT_TOKEN`. Show set/not-set; if set, show first 6 chars masked.
   - Read `~/.claude/channels/discord/sessions/<name>/access.json` (missing =
     defaults). Show: DM policy, allowed senders count, pending pairings count,
     guild channels opted in.
3. If no sessions exist: *"No sessions configured. Run
   `/discord:configure <name> <token>` to create one — pick any short name
   like `personal` or `work`."*
4. If sessions exist, show status table and next steps based on state.

### `<name>` (one arg, not a token) — show status for that session

1. Check if `<name>` exists in `sessions.json`. If not, tell the user:
   *"Session '<name>' not found. Available sessions: …"* and stop.
2. Read `.env` and `access.json` from
   `~/.claude/channels/discord/sessions/<name>/`.
3. Show the same detail as the no-args case but for this session only.
4. Show next steps:
   - No token → *"Run `/discord:configure <name> <token>` with your bot token."*
   - Token set, policy is pairing, nobody allowed → *"DM your bot on Discord.
     It replies with a code; approve with `/discord:access <name> pair <code>`."*
   - Token set, someone allowed → *"Ready. DM your bot to reach the assistant."*
5. Drive toward lockdown — same guidance as before (push `allowlist` once all
   IDs are captured).

### `<name> <token>` — create or update a session

1. `<name>` is the first arg (short name, alphanumeric + hyphens).
   `<token>` is the second arg (Discord bot token — long base64-ish string).
2. `mkdir -p ~/.claude/channels/discord/sessions/<name>`
3. Read existing `.env` in that dir if present; update/add `DISCORD_BOT_TOKEN=`
   line, preserve other keys. Write back, no quotes.
4. `chmod 600 ~/.claude/channels/discord/sessions/<name>/.env`
5. Read `~/.claude/channels/discord/sessions.json` (or `[]` if missing).
   Add `{ "name": "<name>", "stateDir": "<absolute path>" }` if not already
   present (match on name). Write back (pretty JSON).
6. Confirm, then show status for this session.
7. Remind: *"Restart the session or `/reload-plugins` for the new bot to
   connect."*

### `<name> clear` — remove a session

1. Read `sessions.json`, remove the entry matching `<name>`.
2. Write back the updated `sessions.json`.
3. Optionally note that the state dir
   `~/.claude/channels/discord/sessions/<name>/` still exists (don't delete
   it automatically — may contain access.json the user wants to keep).
4. Remind: *"Restart the session or `/reload-plugins` to disconnect."*

---

## Implementation notes

- The sessions dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `sessions.json` at boot. Session changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/discord:access` take effect immediately, no restart.
- Session names should be simple alphanumeric strings (with hyphens allowed).
  Reject names with spaces or special characters.
