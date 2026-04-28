---
name: resume-team
description: Use this skill when an agent-team session has lost runtime team context — typically after a tmux server crash, a `claude --resume` of the team lead, or any state where SendMessage to teammates fails with "No agent named 'X' is currently addressable. Spawn a new one or use the agent ID." Restores warm-context teammates by `claude --resume`-ing each member's saved session jsonl with the right flags + env, AND rehydrates the team lead's appState.teamContext, AND places the resumed workers into the lead's tmux window in a lead-left/workers-right layout. Triggers on: "restore the team", "rehydrate team", "tmux died bring agents back", "agents lost", "swarm died", "resume my team".
compatibility: Requires tmux, claude CLI, and jq. Designed for Claude Code agent-team sessions.
metadata:
  version: "0.1.0"
---

# Restore an agent team after a runtime context loss

## When this happens

Three failure modes converge on the same symptom — `SendMessage` returns "No agent named 'X' is currently addressable":

1. **tmux server died** while the swarm was running. All teammate panes are gone; their processes are gone. The team lead's session may also have been resumed via `claude --resume`.
2. **Team lead resumed via `claude --resume`** without team-context flags. The conversation history loaded but `appState.teamContext` is empty, so even with live teammates the lead can't address them.
3. **Manual respawn of teammates** without `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in their env. Their `isAgentSwarmsEnabled()` returns false, so `useInboxPoller`'s `enabled` prop is false and they never read messages from their mailbox.

The fix is bilateral: restore each teammate as a properly-flagged process AND rehydrate the lead's `teamContext` AND place the workers in the lead's tmux window so the user can see them.

## Prerequisites

- `tmux` reachable from the calling shell. Verify via `tmux ls` — the user's interactive tmux server must be addressable.
- `claude` binary in `$PATH` (typically `~/.local/bin/claude`).
- `jq` available.
- The team config exists at `~/.claude/teams/<team-name>/config.json`.
- Each teammate has a saved session jsonl at `~/.claude/projects/<project-id>/<session-id>.jsonl` from before the crash. Without this, warm context is lost — fall back to fresh-spawn-and-brief.
- **Each `claude --resume` must run with cwd inside the project that owns the session.** `--resume` resolves the session id against the project derived from the current working directory; launching from a subdir of an unrelated project fails with `No conversation found`. Either `cd <project-root>` before invoking, or set the spawn pane's start directory (see step 4).

## Procedure

### Step 1: Identify the team and its members

Read the team config:

```bash
cat ~/.claude/teams/<team-name>/config.json | jq '.members[] | {name, agentId, color: .color}'
```

For each member (other than `team-lead`), record:
- `name` (e.g. `coord-dev`)
- `agentId` (e.g. `coord-dev@p2claw-phase2`)
- `color` (e.g. `blue`)

### Step 2: Find each member's saved session jsonl

The session files live at `~/.claude/projects/<project-id>/`. The `<project-id>` is a slug of the working directory (e.g. `-home-tato-Desktop-p2claw`).

For each member, find the jsonl whose first three lines contain `You are **<name>**` in the prompt body:

```bash
for f in ~/.claude/projects/<project-id>/*.jsonl; do
  if [ -s "$f" ]; then
    role=$(head -3 "$f" | jq -r 'select(.type=="user")? | .message.content?' 2>/dev/null \
      | grep -oE 'You are \*\*[a-z-]+\*\*' | head -1)
    if [ "$role" = "You are **<name>**" ]; then
      echo "<name> session: $f"
    fi
  fi
done
```

The session id is the basename without `.jsonl`.

### Step 3: Locate the team lead's tmux pane

The resumed workers should land in the same tmux window the user is already using to drive the lead, with the lead on the left and workers tiled on the right.

Find the pane running the lead's `claude` process:

```bash
tmux list-panes -aF '#{session_name}:#{window_index}.#{pane_index} #{pane_pid} #{pane_current_command}' \
  | grep -E '\bclaude\b' | head -5
```

If multiple `claude` panes appear, the lead is the one whose pid matches the team-lead session. Verify by inspecting `/proc/<pid>/cmdline` if needed; otherwise ask the user which is the lead.

Record the lead's location as `<lead-target>` in the form `session:window.pane` (e.g. `0:0.0`). The window target without the pane suffix is `<lead-window>` (e.g. `0:0`).

### Step 4: Spawn each teammate as a split-pane in the lead's window

For each teammate, split the lead's window and run their `claude --resume` inside the new pane. Two approaches depending on tmux version + ergonomic preference; use whichever the user has working.

**Preferred — split-window, then arrange with main-vertical layout:**

The resume command must mirror what fresh-spawn passes (see `buildInheritedCliFlags` + `buildInheritedEnvVars` in `src/utils/swarm/spawnUtils.ts`). Missing flags don't fail loudly — messaging still works — but the first time the teammate invokes a real tool (Read/Write/Bash) it crashes inside Ink's reconciler `createInstance` because the permission-prompt component tree renders against an inconsistent `appState.toolPermissionContext`. `--dangerously-skip-permissions` (or matching `--permission-mode`) is the load-bearing fix; the rest prevent secondary breakage (plugin tools missing, wrong API endpoint, broken SendMessage routing back to the lead).

```bash
# For each teammate, in order. -c <project-root> sets the pane's cwd so --resume
# can resolve the session id against the right project.
tmux split-window -h -c <project-root> -t <lead-window> \
  "env CLAUDECODE=1 CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 \
       [forward any of CLAUDE_CODE_USE_BEDROCK CLAUDE_CODE_USE_VERTEX CLAUDE_CODE_USE_FOUNDRY \
        ANTHROPIC_BASE_URL CLAUDE_CONFIG_DIR CLAUDE_CODE_REMOTE CLAUDE_CODE_REMOTE_MEMORY_DIR \
        HTTPS_PROXY HTTP_PROXY NO_PROXY SSL_CERT_FILE NODE_EXTRA_CA_CERTS \
        REQUESTS_CA_BUNDLE CURL_CA_BUNDLE that are set in the lead's env] \
   claude --resume <session-id> \
     --agent-id <agent-id> --agent-name <name> --team-name <team-name> --agent-color <color> \
     --parent-session-id <lead-session-id> \
     --teammate-mode <mode-from-lead> \
     --dangerously-skip-permissions \
     [--settings <path>]  [--plugin-dir <dir>]…  [--model <override>]  [--chrome|--no-chrome] \
     'back online; check your inbox and stand by for next task'"
```

To get the lead's session id for `--parent-session-id`, read it from the most recently modified jsonl in `~/.claude/projects/<project-id>/` whose first user message identifies the lead, or check `/proc/<lead-pid>/cmdline`. The lead's teammate mode lives in `appState` — if the user can't easily read it, default to the same mode used when the team was first spawned (commonly `proactive`).

After all teammates are spawned, apply the layout in one shot:

```bash
tmux select-layout -t <lead-window> main-vertical
```

`main-vertical` puts the largest pane on the left (the lead) and tiles the rest as a single column on the right. The size split is configurable; the default is roughly 50/50 — adjust with:

```bash
tmux resize-pane -t <lead-target> -x 60%
```

if the user wants the lead pane wider.

**Alternative — tiled layout (3+ teammates, more columns):**

After spawning, run `tmux select-layout -t <lead-window> tiled` instead. Each teammate gets equal pane area. Less directional ("workers on the right") but readable when there are 4+.

**Critical bits in the spawn command:**

- `env CLAUDECODE=1 CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` — `CLAUDECODE=1` is the "I am the bundled CLI" marker that gates several runtime paths; the EXPERIMENTAL flag enables `isAgentSwarmsEnabled()` so the inbox poller runs. Without the swarm flag the teammate appears alive but won't read messages.
- **`--dangerously-skip-permissions` (or matching `--permission-mode`)** — load-bearing for tool use. Fresh-spawn always inherits the lead's permission mode; without it, the resumed teammate falls back to default mode and the permission-prompt UI render crashes Ink's `createInstance` on first Read/Write/Bash. SendMessage doesn't need permission, which is why messaging-only turns appear to work.
- **`--parent-session-id <lead-session-id>`** — sets `parentSessionId` on the resumed teammate's `dynamicTeamContext`. Used by `SendMessage` to route replies back to the lead. Without it, teammate→lead messages can't find a parent.
- **`--teammate-mode <mode>`** — propagates the lead's teammate mode (proactive/reactive/etc.). Without it the agent's mode is undefined.
- **`--settings <path>`** if the lead was started with a settings flag, **`--plugin-dir <dir>`** for each inline plugin, **`--model <override>`** if applicable, **`--chrome`/`--no-chrome`** if explicitly set — these mirror what `buildInheritedCliFlags` always passes during fresh-spawn. Plugin-provided MCP servers in particular won't load on the teammate without `--plugin-dir`, leading to mismatched tool registries.
- **Forwarded env vars** (`ANTHROPIC_BASE_URL`, `CLAUDE_CODE_USE_BEDROCK`/`VERTEX`/`FOUNDRY`, proxy + CA cert vars, `CLAUDE_CONFIG_DIR`, `CLAUDE_CODE_REMOTE`, `CLAUDE_CODE_REMOTE_MEMORY_DIR`) — these don't transit a fresh tmux login shell, so forward them explicitly via `env`. Missing API-provider vars send teammate requests to the wrong endpoint (firstParty default).
- `tmux split-window -c <project-root>` — sets the new pane's cwd to the project root that owns the session. Without it, the pane inherits the lead's cwd which may be a sibling project, and `--resume` fails with `No conversation found`.
- `--resume <session-id>` — replays the conversation jsonl for warm context.
- `--agent-id`, `--agent-name`, `--team-name`, `--agent-color` — set `dynamicTeamContext` on the spawned process so it identifies as a teammate.
- **Trailing wake prompt** (e.g. `'back online; check your inbox and stand by for next task'`) — current `claude --resume` requires a prompt arg or it exits with `No deferred tool marker found in the resumed session. Provide a prompt to continue the conversation.` The bare `claude --resume <id>` form no longer works.

After spawning, wait ~5-10 seconds for the process to load and the inbox poller to start. Verify by:

```bash
tmux capture-pane -t <name> -p | tail -5
# Should show the resumed prompt + a `── <name> ──` border indicating teammate identity is set.
```

Or check the inbox: write a test message to the teammate's mailbox and see if the unread count drops to 0 within a few seconds:

```bash
inbox=~/.claude/teams/<team-name>/inboxes/<name>.json
jq '. + [{"from":"team-lead","text":"poller test","summary":"poller test","timestamp":"'$(date -u +%Y-%m-%dT%H:%M:%S.000Z)'","color":"blue","read":false}]' "$inbox" > /tmp/inbox.json
mv /tmp/inbox.json "$inbox"
sleep 3
jq '[.[] | select(.read==false)] | length' "$inbox"  # should be 0
```

### Step 5: Rehydrate the team lead's appState.teamContext

The lead's `SendMessage` checks `appState.teamContext.teamName` before allowing a send. After `claude --resume`, that's empty. **The fix is to spawn a throwaway agent via the `Agent` tool with the team_name parameter** — the spawn flow's `setAppState` call rehydrates the field as a documented side effect. Verified empirically: a single throwaway spawn restores SendMessage routing for the rest of the lead's session.

Use the `Agent` tool from the lead's session:

```
Agent({
  subagent_type: "general-purpose",
  team_name: "<team-name>",
  name: "rehydrate-throwaway",
  prompt: "Throwaway agent. Reply with 'ack' and stop. Do nothing else.",
  run_in_background: true,
  description: "Rehydrate teamContext"
})
```

This adds a member to the team config under the throwaway name and creates a tmux pane (or in-process worker, depending on backend). Clean up after step 6:

```bash
# Remove the throwaway from team config
jq '.members |= map(select(.name != "rehydrate-throwaway"))' \
  ~/.claude/teams/<team-name>/config.json > /tmp/clean.json \
  && mv /tmp/clean.json ~/.claude/teams/<team-name>/config.json

# Kill its pane / session if it created one
tmux kill-session -t rehydrate-throwaway 2>/dev/null
# (or kill the pane directly if it spawned in the lead's window)
```

### Step 6: Verify routing works both directions

**Lead → teammate**: send via `SendMessage` tool to one of the resumed teammates. If it succeeds, the gate passed (= teamContext rehydrated).

**Teammate → lead**: ask the teammate to reply via SendMessage to "team-lead". If that arrives in the lead's inbox notifications, the round-trip works.

If `SendMessage` from the lead still returns "currently addressable", the rehydrate step (5) didn't take effect. Try the throwaway spawn again. If it still fails, fall back to direct mailbox writes — the teammate's poller (now running thanks to step 4's env var) will pick them up:

```bash
jq '. + [{"from":"team-lead","text":"<your message>","summary":"<short>","timestamp":"'$(date -u +%Y-%m-%dT%H:%M:%S.000Z)'","color":"blue","read":false}]' \
  ~/.claude/teams/<team-name>/inboxes/<name>.json > /tmp/inbox-new.json \
  && mv /tmp/inbox-new.json ~/.claude/teams/<team-name>/inboxes/<name>.json
```

## Diagnostics for stuck cases

| Symptom | Cause | Fix |
|---|---|---|
| Spawn fails: `tmux: server not running` | No tmux server | Have user start tmux: `tmux new -d` then re-run |
| Pane shows the agent but inbox writes go unread | Missing env var on spawn | Kill pane, respawn with `env CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` prefix |
| `tmux split-window` fails with "no current target" | Lead window was specified imprecisely | Use the full `session:window` form (e.g. `0:0`) — not just `0` |
| Lead's `SendMessage` still 404s after step 5 | Throwaway spawn didn't fire `setAppState` | Repeat step 5 once. If still broken, lead's session itself may need restart with team flags |
| Lead's `TaskCreate` still routes to the wrong list / 404s after step 5 | Step 5 rehydrates `appState.teamContext` but `TaskCreate` reads from a different state path (`~/.claude/tasks/<team-name>/`) that the throwaway-spawn doesn't touch | Workaround: write the task JSON directly to `~/.claude/tasks/<team-name>/<id>.json`. The throwaway-spawn fix is SendMessage-only |
| Resumed teammate processes a few messaging turns then crashes inside `createInstance` (Ink reconciler) on first Read/Write/Bash | Resume command missing `--dangerously-skip-permissions` (or matching `--permission-mode`); permission-prompt UI renders against an inconsistent toolPermissionContext | Kill the pane and respawn with the full fresh-spawn-equivalent flag set listed in step 4 — start with the permission flag, then layer `--plugin-dir` / `--settings` / `--teammate-mode` / `--parent-session-id` |
| Resumed teammate's tool calls hit unexpected MCP / endpoint errors | Plugin-provided MCP servers didn't load (missing `--plugin-dir`) or API-provider env vars not forwarded | Respawn with `--plugin-dir <dir>` for each inline plugin and forward `ANTHROPIC_BASE_URL` + `CLAUDE_CODE_USE_*` env vars |
| Pane border shows wrong name (e.g. shows the resumed prompt's name not the flag) | `--agent-name` not honored | Verify CLI flag spelling; if persists, the resumed jsonl may have a hardcoded mismatch — fall back to fresh-spawn |
| `select-layout main-vertical` makes panes too small | Too many workers | Switch to `tiled` layout, or kill less-relevant workers first |
| Multiple sessions / panes with the same name | Previous spawn didn't clean up | `tmux kill-session -t <name>` for orphans, or `tmux list-panes -a` and prune by pid |

## What this skill does NOT do

- **Reconstruct lost session jsonls.** If the saved transcripts are gone, warm context is unrecoverable. Fall back to fresh-spawn via the `Agent` tool with a brief from `git log` + the task list.
- **Restore in-process teammates.** Those run inside the lead's process and die with it. Only out-of-process (tmux-paned) teammates can be `--resume`-d.
- **Reconstruct in-flight task ownership.** Task list state at `~/.claude/tasks/<team-name>/` persists across resumes, but any "I'm currently working on X" runtime ownership state lost when the original session died is gone.

## Cleanup after the procedure

If you spawned a throwaway for rehydration in step 5:

```bash
# Kill the throwaway pane if it created one
tmux kill-session -t rehydrate-throwaway 2>/dev/null

# Remove from team config
jq '.members |= map(select(.name != "rehydrate-throwaway"))' \
  ~/.claude/teams/<team-name>/config.json > /tmp/clean.json \
  && mv /tmp/clean.json ~/.claude/teams/<team-name>/config.json
```

If at the end of the user's work session you want to send all teammates away (and re-spawn fresh later from disk):

```bash
# Workers in the lead's window — kill the panes (lead pane stays)
tmux list-panes -t <lead-window> -F '#{pane_id} #{pane_current_command}' \
  | grep claude | grep -v <lead-pane-id> | awk '{print $1}' \
  | xargs -I{} tmux kill-pane -t {}
```

Their session jsonls remain on disk for future restoration.
