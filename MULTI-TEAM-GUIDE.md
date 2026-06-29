# Multi-Team Coordination for Claude Code

## The Problem

Claude Code agent teams have hard limits:
- **One team per session** — a lead can only manage one team
- **No nested teams** — teammates can't spawn their own teams
- **No cross-team communication** — teams in separate sessions can't talk to each other

For large-scale parallel work (like hunting across 25 contracts on 4 chains), you need multiple teams running simultaneously and sharing findings.

## The Solution

A file-based protocol that connects multiple independent Claude Code sessions. Each session creates its own internal agent team. All teams share a filesystem mailbox for coordination — using the same JSON-on-disk pattern Claude uses internally, extended across team boundaries.

```
┌──────────────────────────────────────────────────────────────┐
│                    Shared Filesystem                         │
│  ~/.claude/meta-teams/<name>/                                │
│  ├── registry.json    (who's running)                        │
│  ├── mailbox/         (cross-team messages)                  │
│  ├── findings/        (shared bug reports)                   │
│  ├── tasks.json       (cross-team task list)                 │
│  └── status/          (read cursors per team)                │
└─────────┬───────────────┬───────────────┬────────────────────┘
          │               │               │
    ┌─────▼─────┐   ┌────▼──────┐   ┌────▼──────┐
    │  Team A   │   │  Team B   │   │  Team C   │
    │  (lead)   │   │  (lead)   │   │  (lead)   │
    │   ├─mate1 │   │   ├─mate1 │   │   ├─mate1 │
    │   ├─mate2 │   │   ├─mate2 │   │   ├─mate2 │
    │   └─mate3 │   │   └─mate3 │   │   └─mate3 │
    └───────────┘   └───────────┘   └───────────┘
    Each is a separate claude process with its own agent team
```

## How It Works

### 1. Launch

`launch.sh` reads a JSON config that defines your teams:

```json
{
  "meta_team": "my-hunt",
  "project_dir": "/path/to/project",
  "teams": [
    {
      "name": "team-alpha",
      "role": "Hunt EVM contracts",
      "teammates": 3,
      "task": "Create an agent team with 3 teammates to audit EVM contracts..."
    },
    {
      "name": "team-beta",
      "role": "Hunt cross-chain vectors",
      "teammates": 4,
      "task": "Create an agent team with 4 teammates to find cross-chain bugs..."
    }
  ]
}
```

For each team, it:
1. Creates a working directory with a generated `CLAUDE.md` containing the cross-team protocol
2. Installs a `UserPromptSubmit` hook that checks the shared mailbox
3. Spawns `claude --dangerously-skip-permissions -p "<task>"` in the background
4. Registers the team in the shared registry

### 2. Communication

Teams communicate via JSON files dropped in the shared mailbox directory:

```json
{
  "id": "1776025430226063823",
  "from": "team-alpha",
  "to": "team-beta",
  "type": "finding",
  "timestamp": "2026-04-12T15:23:50-05:00",
  "content": "Found reentrancy in EndpointV2.send() — see findings/team-alpha-Bug99.json"
}
```

Message types: `finding`, `task`, `question`, `status`, `directive`

The `check-mailbox.sh` hook reads unread messages (tracked via per-team cursor files) and injects them into the Claude session's context.

### 3. Findings

When a team finds something, it posts to the shared findings directory AND broadcasts to all teams:

```json
{
  "id": "Bug99",
  "team": "team-alpha",
  "severity": "CRITICAL",
  "title": "Reentrancy in send()",
  "details": "EndpointV2.send() allows reentrant calls via lzCompose callback",
  "timestamp": "2026-04-12T15:23:51-05:00",
  "status": "new"
}
```

Other teams see this via the mailbox broadcast and can investigate related vectors.

### 4. Monitoring

```bash
./status.sh my-hunt
```

Output:
```
=== META-TEAM: my-hunt ===

--- Teams ---
  team-alpha           [active  ] pid=12345 ✓  role: Hunt EVM contracts
  team-beta            [active  ] pid=12346 ✓  role: Hunt cross-chain vectors

--- Tasks ---
  [in_progress ] task-001        → team-alpha     | Fuzz UltraLightNodeV2
  [pending      ] task-002        → unassigned     | Test nonce desync

--- Findings ---
  [CRITICAL] team-alpha      | Reentrancy in send()

--- Mailbox ---
  5 messages total
```

### 5. Cleanup

```bash
./cleanup.sh my-hunt          # fails if processes still running
./cleanup.sh my-hunt --force  # kills all, archives findings + logs
```

Findings and logs are archived to tar.gz before deletion.

## Quick Start

```bash
cd /media/phantom-orchestrator/Elements1/Ubuntu/bounty-recon/multi-team

# 1. See example config
./launch.sh --example

# 2. Edit the example or use the pre-built LayerZero config
./launch.sh examples/hunt-layerzero.json

# 3. Monitor progress
./status.sh hunt-lz

# 4. Send messages to teams
./send.sh hunt-lz coordinator all directive "Prioritize V1 ULN reentrancy"
./send.sh hunt-lz coordinator team-crosschain question "Found anything on nonce desync?"

# 5. Watch logs live
tail -f ~/.claude/meta-teams/hunt-lz/logs/*.log

# 6. When done
./cleanup.sh hunt-lz --force
```

## Protocol Functions

Source `lib/protocol.sh` to use these in scripts or from the shell:

| Function | Purpose |
|----------|---------|
| `meta_init <name>` | Create shared state directory |
| `meta_register_team <meta> <team> <role> <pid> <workdir>` | Register a team in the registry |
| `meta_send <meta> <from> <to\|all> <type> <content>` | Send a message to one team or broadcast |
| `meta_read_messages <meta> <team>` | Read unread messages (updates read cursor) |
| `meta_post_finding <meta> <team> <id> <severity> <title> <details>` | Post finding + broadcast |
| `meta_add_task <meta> <id> <title> <assigned> <depends>` | Add a cross-team task |
| `meta_update_task <meta> <id> <status> <result>` | Update task status |
| `meta_status <meta>` | Print full dashboard |

## File Layout

```
multi-team/
├── launch.sh              # Spawns N claude sessions with shared mailbox
├── status.sh              # Dashboard: teams, tasks, findings, health
├── send.sh                # CLI: send cross-team messages
├── cleanup.sh             # Tear down + archive
├── lib/
│   └── protocol.sh        # Core protocol (all shared functions)
├── hooks/
│   └── check-mailbox.sh   # UserPromptSubmit hook for message injection
├── templates/
│   └── team-lead.md       # CLAUDE.md template for team leads
└── examples/
    └── hunt-layerzero.json # 4-team LayerZero config
```

Runtime state at `~/.claude/meta-teams/<name>/`:
```
├── registry.json          # Team roster (name, role, PID, status)
├── tasks.json             # Cross-team task list
├── mailbox/               # JSON messages (timestamped, from→to)
├── findings/              # Shared findings from all teams
├── status/                # Per-team read cursors
├── logs/                  # Per-team stdout/stderr
└── workdirs/              # Per-team working directory + CLAUDE.md
```

## How the Hook Works

`check-mailbox.sh` runs on every `UserPromptSubmit` event. It:

1. Checks env vars `META_TEAM_NAME` and `TEAM_NAME` — skips if not in multi-team context
2. Reads the cursor file (`status/<team>.cursor`) to find the last-read message timestamp
3. Scans `mailbox/` for messages addressed to this team or "all" that are newer than the cursor
4. Outputs them as `[CROSS-TEAM FINDING]` / `[CROSS-TEAM STATUS]` / etc. — this text gets injected into Claude's context
5. Updates the cursor to the latest message timestamp

This means every time a team lead processes a prompt, it automatically sees any new messages from other teams.

## Writing Your Own Config

```json
{
  "meta_team": "unique-name",
  "project_dir": "/absolute/path/to/project",
  "teams": [
    {
      "name": "short-name",
      "role": "Human-readable description of what this team does",
      "teammates": 3,
      "task": "The exact prompt sent to claude to create the internal agent team. Be specific: include file paths, tool commands, and what to focus on."
    }
  ]
}
```

Tips:
- Keep team count at 3-5. Each team spawns its own teammates, so 4 teams × 3 teammates = 16 agents.
- Make roles non-overlapping. If two teams audit the same contract, they'll duplicate work.
- Be specific in the task prompt. Include exact file paths, tool commands, and what to look for.
- Use `send.sh` to redirect teams mid-run if priorities change.

## Limitations

- **Non-interactive by default**: `launch.sh` uses `claude -p` (print mode). Teams run autonomously and exit when done. For interactive sessions, launch claude manually in separate terminals and set the env vars.
- **No tmux currently**: Processes run in background. Install tmux for split-pane visibility.
- **Message polling, not push**: Teams only see new messages when the hook fires (on prompt submit). In `claude -p` mode, there's only one prompt, so messages arrive after the work is done. Interactive mode fixes this.
- **No finding deduplication**: Two teams might find the same bug independently.
- **Token cost scales linearly**: Each team lead + teammates = independent context windows. 4 teams × 4 agents = 16 concurrent Claude instances.

## Interactive Mode (Default — Recommended)

The default launch mode opens a **real gnome-terminal window** per team, each running a fully interactive `claude` session. This means:
- Real agent teams with actual teammates (not simulated)
- You can type into any team's terminal directly
- The mailbox hook fires on every prompt, delivering cross-team messages in real-time
- Each session creates its own internal team with `Shift+Down` to cycle teammates

```bash
# Launch — opens N terminal windows automatically
./launch.sh examples/hunt-layerzero.json

# Each window is a full interactive claude session.
# Click into any window to interact with that team.
```

For headless/CI use, add `--background` to use `claude -p` instead:
```bash
./launch.sh examples/hunt-layerzero.json --background
```

### How It Works Under the Hood
1. `launch.sh` creates a per-team workdir with `git init` (so claude trusts it without prompting)
2. Generates a `CLAUDE.md` with the team's role and cross-team protocol
3. Creates `.claude/settings.json` with the mailbox hook and env vars
4. Writes a launcher script that sets env vars, registers the team, and runs `claude --dangerously-skip-permissions "<initial-prompt>"`
5. Opens `gnome-terminal --title="<team-name>" -- bash <launcher-script>`

### Manual Launch (Alternative)
You can also launch teams manually in separate terminals:

```bash
# Terminal 1
export META_TEAM_NAME="hunt-lz" TEAM_NAME="team-evm-v1" META_TEAM_DIR="$HOME/.claude/meta-teams"
source multi-team/lib/protocol.sh && meta_init "hunt-lz"
meta_register_team "hunt-lz" "team-evm-v1" "V1 EVM" $$ "$(pwd)"
claude --dangerously-skip-permissions

# Terminal 2
export META_TEAM_NAME="hunt-lz" TEAM_NAME="team-crosschain" META_TEAM_DIR="$HOME/.claude/meta-teams"
meta_register_team "hunt-lz" "team-crosschain" "Cross-chain" $$ "$(pwd)"
claude --dangerously-skip-permissions

# Terminal 3 — coordinator
./status.sh hunt-lz
./send.sh hunt-lz coordinator all directive "Focus on reentrancy"
```

## Tested (2026-04-12)

### Test 1: Protocol Unit Tests
All protocol functions verified: `meta_init`, `meta_register_team`, `meta_send`, `meta_read_messages`, `meta_post_finding`, `meta_status`. Targeted messages only go to the named team. Broadcasts go to all. Cursor tracking prevents re-reading.

### Test 2: Concurrent Writes
10 simultaneous `meta_send` calls from background processes — all 10 messages written, zero lost, zero collisions. Nanosecond timestamps in filenames prevent overwrites.

### Test 3: Background Mode (claude -p) — 3 Teams
3 teams, each told to create 2 internal teammates. All 3 posted findings to shared state. team-gamma read team-alpha's and team-beta's findings via the mailbox. Targeted message from gamma to alpha delivered correctly. 17 messages in mailbox total.

### Test 4: Interactive Mode (gnome-terminal) — 2 Teams
2 teams launched in separate gnome-terminal windows. Both registered in the registry with correct PIDs. No settings errors. Hook format validated. Sessions started with initial prompts automatically.
