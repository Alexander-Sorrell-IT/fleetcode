# FleetCode 2 — Multi-Brain Fleet Orchestration for Claude Code

## What Is FleetCode 2

FleetCode 2 combines two systems into one:

1. **FleetCode** (v1) — launches multiple Claude Code sessions in separate terminal windows, connected by a shared filesystem mailbox. Each session can create its own internal agent team. A captain session steers the whole fleet.

2. **Brain Stream** — a consciousness-driven investigation mode where agents act as neurons in a shared brain. Instead of tasks and reports, they send short 1-3 line signals, react to each other in real-time, and converge on the most interesting thread. A shared consciousness file is the brain's working memory.

**FleetCode 2 = multiple brains running in parallel, sharing findings across a fleet mailbox, steered by a captain.**

## How It Works

```
You (talking to captain session)
  │
  ├─── fleet_status / fleet_send / fleet_findings
  │
  ├── [Terminal 1] Brain V1 EVM ──── consciousness.md
  │     ├── Neuron ENTRY (reading Endpoint.send)
  │     ├── Neuron PROOF (reading FPValidator)
  │     ├── Neuron MONEY (reading fee calc)
  │     └── Pacemaker (heartbeat + bridge to mailbox)
  │
  ├── [Terminal 2] Brain V2 EVM ──── consciousness.md
  │     ├── Neuron SEND (reading SendULN302)
  │     ├── Neuron RECEIVE (reading ReceiveULN302)
  │     ├── Neuron DVN (reading DVN.verify)
  │     └── Pacemaker (heartbeat + bridge to mailbox)
  │
  ├── [Terminal 3] Brain Cross-Chain ── consciousness.md
  │     ├── Neuron ENCODE (address encoding)
  │     ├── Neuron NONCE (nonce desync)
  │     ├── Neuron GAS (executor gas models)
  │     └── Pacemaker (heartbeat + bridge to mailbox)
  │
  └── [Terminal 4] Tool Runner (standard mode)
        ├── Teammate 1 (hunt-auto V1)
        ├── Teammate 2 (hunt-auto V2)
        └── Teammate 3 (hunt-auto Solana+TON)
              │
              └── Posts tool findings to mailbox
                    → Brain teams see them as stimuli
```

### Two Modes Per Team

Each team in the fleet config can run in either mode:

**`"mode": "brain-stream"`** — Neurons + consciousness file + pacemaker
- No tasks. Neurons self-direct by reacting to each other's signals.
- Shared consciousness file is the synchronization barrier — every neuron reads the full file before sending.
- Pacemaker keeps the heartbeat going, injects stimuli when stalled, bridges critical findings to the fleet mailbox.
- `TeammateIdle` hook detects stalled streams (>120s no new signal) and pushes the pacemaker to act.
- Subagent definitions: `brain-neuron.md` and `brain-pacemaker.md` installed automatically.

**`"mode": "standard"`** (default) — Tasks + teammates + reports
- Traditional agent team with task list.
- Teammates claim and complete tasks.
- `TaskCompleted` hook notifies captain when work finishes.
- `TeammateIdle` hook checks for unread cross-team messages before allowing idle.
- Subagent definitions: `hunter.md`, `reviewer.md`, `researcher.md`.

### The Two-Way Bridge

Cross-team communication works the same regardless of mode:

**Fleet Mailbox** (`~/.claude/meta-teams/<name>/mailbox/`)
- JSON messages between teams: findings, questions, directives, status updates
- `check-mailbox.sh` hook injects unread messages into each session's context on every prompt
- Captain sends directives via `fleet_send`, teams post findings via `meta_post_finding`

**Consciousness Bridge** (brain-stream teams only)
- `consciousness-bridge.sh` hook reads fleet mailbox messages and appends them to the consciousness file as `[HH:MM:SS] CROSS-BRAIN(team-name): message`
- Neurons see cross-brain intel as part of the stream and can react to it
- When a brain team posts a finding to the fleet mailbox, all other brains get it injected into their consciousness

This means: **Brain V1 finds a reentrancy → posts to mailbox → Brain Cross-Chain's consciousness file gets `CROSS-BRAIN(brain-v1): reentrancy found in send()` → neurons react and test if it chains across chains.**

### The Captain

The captain is your session. You talk to it, it steers the fleet.

```bash
source multi-team/captain.sh hunt-lz-brain

fleet_status          # dashboard: all teams, findings, process health
fleet_findings        # all findings with details
fleet_send team all "Focus on the reentrancy brain-v1 found"
fleet_ask brain-v2 "Did you check DVN.verify for the same pattern?"
fleet_focus brain-crosschain "nonce desync in V2 lazy inbound"
fleet_escalate "CRITICAL: brain-v1 confirmed fund theft vector"
fleet_mailbox         # read messages from teams
fleet_logs brain-v1   # tail a team's log
fleet_kill team-tools  # shut down a team
fleet_cleanup         # tear down everything
```

## Setup

```bash
tar xzf fleetcode.tar.gz
cd multi-team
./setup.sh    # enables agent teams, detects terminal, sets teammateMode, verifies deps
```

`setup.sh` does:
1. Checks claude >= v2.1.32
2. Enables `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` in `~/.claude/settings.json`
3. Sets `skipDangerousModePermissionPrompt` so teams don't stall
4. Detects terminal emulator (gnome-terminal, xfce4-terminal, konsole, kitty, alacritty, wezterm, xterm)
5. Sets `teammateMode: "in-process"` in `~/.claude.json`
6. Makes all scripts executable
7. Tests the protocol (send + receive)

## Launch

```bash
# Brain stream fleet (3 brains + 1 tool runner = 4 terminals, ~15 agents)
./launch.sh examples/hunt-layerzero-brainstream.json

# Standard fleet (4 task-based teams = 4 terminals, ~13 agents)
./launch.sh examples/hunt-layerzero.json

# Background mode (no terminals, uses claude -p)
./launch.sh examples/hunt-layerzero.json --background
```

## Config Format

```json
{
  "meta_team": "fleet-name",
  "project_dir": "/path/to/project",
  "teams": [
    {
      "name": "brain-v1",
      "role": "What this brain investigates",
      "mode": "brain-stream",
      "teammates": 4,
      "task": "Prompt that tells the pacemaker what to investigate and where neurons should start"
    },
    {
      "name": "team-tools",
      "role": "What this team does",
      "mode": "standard",
      "teammates": 3,
      "task": "Prompt that tells the lead what tasks to create"
    }
  ]
}
```

## File Layout

```
multi-team/                          (25KB tar)
├── setup.sh                         # One-time setup
├── launch.sh                        # Opens terminal windows per team
├── captain.sh                       # Source in your session to steer fleet
├── status.sh                        # Dashboard
├── send.sh                          # CLI: send cross-team messages
├── cleanup.sh                       # Tear down + archive
├── lib/
│   └── protocol.sh                  # Core: mailbox, registry, tasks, findings
├── hooks/
│   ├── check-mailbox.sh             # Injects fleet messages into context
│   ├── consciousness-bridge.sh      # Two-way bridge: mailbox ↔ consciousness
│   ├── stream-stall-check.sh        # Detects brain-stream stalls (>120s)
│   ├── teammate-idle.sh             # Checks unread messages before idle
│   └── task-completed.sh            # Notifies captain on task completion
├── agents/
│   ├── brain-neuron.md              # Neuron: signal-react-investigate loop
│   ├── brain-pacemaker.md           # Pacemaker: heartbeat + bridge
│   ├── hunter.md                    # Standard: bug hunter
│   ├── reviewer.md                  # Standard: adversarial reviewer
│   └── researcher.md                # Standard: attack surface mapper
├── templates/
│   ├── brain-stream-lead.md         # CLAUDE.md for brain-stream teams
│   └── team-lead.md                 # CLAUDE.md for standard teams
├── examples/
│   ├── hunt-layerzero-brainstream.json  # 3 brains + 1 tool runner
│   └── hunt-layerzero.json              # 4 standard teams
└── FLEETCODE2.md                    # This document
```

## What Each Hook Does

| Hook | Event | Mode | Action |
|------|-------|------|--------|
| `check-mailbox.sh` | UserPromptSubmit | Both | Reads unread fleet messages, injects into context |
| `consciousness-bridge.sh` | UserPromptSubmit | Brain | Writes fleet messages into consciousness file |
| `stream-stall-check.sh` | TeammateIdle | Brain | Exit 2 if no consciousness entry for >120s |
| `teammate-idle.sh` | TeammateIdle | Standard | Exit 2 if unread fleet messages exist |
| `task-completed.sh` | TaskCompleted | Standard | Notifies captain via mailbox |

## What Each Agent Definition Does

| Agent | Mode | Role |
|-------|------|------|
| `brain-neuron` | Brain | Core loop: read consciousness → investigate → signal → react. 1-3 line observations. Drops own thread to chase stronger signals. |
| `brain-pacemaker` | Brain | Spawns neurons, initializes consciousness, keeps heartbeat, bridges findings to fleet mailbox. Participates as a neuron too. |
| `hunter` | Standard | Runs tools first, analyzes output, writes PoCs. Posts findings to mailbox. |
| `reviewer` | Standard | Verifies findings from other teammates. Adversarial — tries to disprove. |
| `researcher` | Standard | Maps attack surfaces, traces fund flows, identifies high-value targets. |

## Verified (2026-04-12)

### Protocol Tests
- All functions: init, register, send, read, post finding, status — working
- Targeted messages delivered only to named team
- Broadcasts delivered to all teams
- 10 concurrent writes: 10/10 delivered, zero lost

### Background Mode (3 teams × 2 teammates)
- 3 teams posted findings (CRITICAL, HIGH, MEDIUM)
- Cross-team reading confirmed (team-gamma read team-alpha's findings)
- Targeted messages delivered correctly

### Interactive Mode (2 teams × 2 teammates)
- gnome-terminal windows opened automatically
- Real interactive Claude Code sessions with real agent teams
- Both teams registered with live PIDs
- Cross-team findings shared via mailbox — team-beta read team-alpha's finding

### Brain Stream Integration
- Brain-stream mode generates correct CLAUDE.md from brain template
- Consciousness file path set per-team (not global /tmp/)
- Agent definitions (brain-neuron, brain-pacemaker) installed with correct placeholders filled
- Settings.json includes consciousness-bridge hook + stall detection for brain teams
- Standard teams get regular hooks (teammate-idle, task-completed)
- Mixed config (3 brains + 1 standard) generates correctly
