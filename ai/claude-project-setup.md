Claude Code Project Setup
The Problem It Solves
Without configuration, Claude starts every session with zero knowledge of your project and no guardrails. These files give Claude persistent instructions, define what it can/can't do, automate quality gates, package reusable workflows, and connect it to external systems.

Core Concepts
CLAUDE.md
Why: Claude has no memory between sessions. This file is read automatically at startup so you don't re-explain your project every time.

Mental model: Like a Dockerfile but for Claude — written once, applied every session. README is for humans; CLAUDE.md is instructions for Claude.

Example:


## Build & Test
- Build: `go build ./...`
- Test: `go test ./...`

## Standards
- Wrap errors: `fmt.Errorf("context: %w", err)`
- Use `slog` for logging, never `fmt.Println`
.claude/ Directory
Why: One file isn't enough. You need structured homes for config, hooks, and skills — separated by purpose and by who commits them.

Mental model: Like .github/ for GitHub Actions — a dedicated directory for Claude Code's project-specific configuration.

Example:


.claude/
├── settings.json        # shared, committed
├── settings.local.json  # local only, gitignored
├── skills/              # custom /commands
└── hooks/               # automation scripts
settings.json
Why: Claude can do almost anything by default. You need guardrails — hard blocks for dangerous commands, env vars for DB connections, hooks wired up.

Mental model: Like Kubernetes RBAC — define what the principal (Claude) is allowed to do. deny is like an explicit deny policy that overrides everything.

Example:


{
  "permissions": {
    "allow": ["Bash(go test ./...)"],
    "deny": ["Bash(rm -rf *)", "Bash(git push --force)"]
  },
  "env": { "DATABASE_URL": "${DATABASE_URL}" }
}
Hooks
Why: Static permissions aren't enough for conditional logic. Hooks let you run scripts at lifecycle events to inspect, block, or log dynamically.

Mental model: Like Git hooks (pre-commit, post-merge) — fire at specific moments, your script decides what happens. Exit 2 = block with message to Claude.

Example:


"hooks": {
  "PreToolUse": [{
    "matcher": "Bash",
    "hooks": [{"type": "command", "command": ".claude/hooks/validate.sh"}]
  }]
}
Skills (Custom /commands)
Why: Repeated workflows shouldn't require typing full instructions every time. Package them once, invoke with arguments on demand.

Mental model: Like make deploy — a named target that runs a defined sequence. CLAUDE.md is always-on context; skills are on-demand workflows you invoke explicitly.

Example:


---
name: generate-service
description: Generate a new Go microservice
---
Create Go microservice called $ARGUMENTS with handler, service, and tests.
MCP Servers
Why: Claude can't natively reach GitHub, your database, Sentry, or Slack. MCP is a standardized protocol that connects Claude to external systems so it fetches live data directly instead of you copy-pasting.

Mental model: Like Kubernetes operators — they extend what Claude can reach, from local files to any connected system.

Example:


claude mcp add --transport http github https://api.githubcopilot.com/mcp/
Gotchas
CLAUDE.md over ~200 lines loses effectiveness — Claude's context window has limits; later instructions get less attention
deny beats allow — if the same command matches both, deny wins always
settings.local.json must be gitignored — never commit personal tokens or local paths
Hook exit code 2 blocks AND informs — Claude sees your message and adjusts; plain non-zero just logs an error
MCP servers need auth separately — adding a server doesn't authenticate it; use /mcp inside Claude Code to handle auth
Quick Reference
File locations:


CLAUDE.md                    # project instructions (committed)
.claude/settings.json        # permissions, hooks, env (committed)
.claude/settings.local.json  # local overrides (gitignored)
.claude/skills/<name>/SKILL.md  # custom /commands (committed)
.mcp.json                    # MCP servers (committed)
~/.claude/CLAUDE.md          # your personal defaults, all projects
~/.claude/settings.json      # your personal settings, all projects
Hook exit codes:


0   → allow, continue normally
2   → block, send message to Claude
other → non-blocking error, execution continues
MCP commands:


claude mcp add --transport http <name> <url>   # add remote server
claude mcp add <name> -- <command>             # add local server
claude mcp list                                # view all servers
claude mcp remove <name>                       # remove server
Session Handoff Prompt
Copy this into a new Claude Code session when you're ready to set up your project:


I understand Claude Code project configuration well. Quick context:
CLAUDE.md gives Claude persistent instructions each session. .claude/ holds
settings.json (permissions/hooks/env), skills (custom /commands), and local
overrides. Hooks are lifecycle scripts that can block tool use dynamically.
MCP servers connect Claude to external systems like GitHub or databases.

I'm building: [describe your idp-platform-orchestrator setup here]