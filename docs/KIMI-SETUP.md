---
scope: IWE
status: active
title: Connecting Kimi to IWE
updated: 2026-11-18
---

# Connecting Kimi to IWE

> Audience: a pilot who has forked the `IWE` and wants to work with IWE through Kimi Code.
> Time: ~15 minutes.
> Source-of-truth: `AGENTS.md`, `.claude/skills/kimi-peer-writer/SKILL.md`, `.claude/skills/peer-conversation/SKILL.md`.

## What You Get

- Kimi Code reads `AGENTS.md` when the repository is opened and applies IWE rules.
- Kimi sees the IWE skills (`/skill:kimi-peer-writer`, `/skill:peer-conversation`, and others).
- Kimi works alone in **Kimi Standalone** mode; Claude is not required.
- If Claude is available, optional peer sessions are also possible.

## Prerequisites

- VS Code with the **Kimi Code** extension, or Kimi Code CLI.
- The `IWE` forked and cloned.
- Kimi Code CLI for scheduled/headless tasks and command-line validation.

Install the CLI using the official script:

```bash
curl -fsSL https://code.kimi.com/kimi-code/install.sh | bash
```

Alternative for Node.js 22.19.0 or later:

```bash
npm install -g @moonshot-ai/kimi-code
```

Verify the installation and configuration:

```bash
kimi --version
kimi doctor
```

Claude Code CLI is needed only for optional peer sessions where Kimi calls Claude. It is not a prerequisite for Kimi Standalone.

## How Kimi Learns the IWE Rules

The file `AGENTS.md` in the repository root is read automatically by Kimi Code when the repository is opened. It contains:

- WP Gate — the opening ritual for each work item.
- Git staging rules.
- Response style for the pilot.
- Commit rules when Kimi is involved.
- Coordination through the MCP Gateway when several agents edit shared files.

Kimi-specific customizations go into `extensions/` or `AGENTS-agent-blocks.md`.

## Configuring IWE Skills

Kimi resolves its data directory from `KIMI_CODE_HOME`, falling back to `~/.kimi-code`. Its standard skill locations are `$KIMI_CODE_HOME/skills/`, `~/.agents/skills/`, and project-level `.kimi-code/skills/` and `.agents/skills/`.

IWE skills live in `.claude/skills/` inside the IWE repository, so register that directory in `$KIMI_CODE_HOME/config.toml` (by default `~/.kimi-code/config.toml`):

```toml
merge_all_available_skills = true
extra_skill_dirs = [
  "/path/to/IWE/.claude/skills",
  # other paths, if any already exist
]
```

Important notes:

- `config.toml` is a personal file on your machine. **Do not commit it** to the repository.
- If `extra_skill_dirs` already contains paths, **add** the IWE path instead of replacing existing entries.
- The array must point to `.claude/skills` of **your IWE repository**, not to `.kimi/skills` or `.kimi-code/skills`.
- Validate the edit with `kimi doctor config /path/to/config.toml`.
- Skill changes are watched automatically. If the catalog does not refresh, run `/reload` or start a new Kimi session.

## Configuring the Aisystant Knowledge Base

Kimi reads MCP servers from `$KIMI_CODE_HOME/mcp.json` (user level), the project-root `.mcp.json`, and `.kimi-code/mcp.json` (project-local overrides). For this installation, keep the active entry in the user-level file and avoid declaring the same server in several files.

User-level example:

```json
{
  "mcpServers": {
    "iwe-knowledge": {
      "url": "https://mcp.aisystant.com/mcp",
      "deferred": true
    }
  }
}
```

For a server with many tools, enable on-demand loading in `config.toml` so that Kimi exposes `select_tools` instead of collapsing the MCP tool list:

```toml
[experimental]
tool-select = true
```

A single Kimi agent does not need the local file-lock gateway. Configure `iwe-local-gateway` only when several agents edit shared files concurrently, with a unique `IWE_AGENT_ID` for each agent.

For OAuth, open the Kimi Code **terminal UI (TUI)**, not the VS Code chat:

```bash
cd /path/to/IWE
kimi
```

When the TUI is idle, run:

```text
/mcp-config login iwe-knowledge
```

If `/mcp-config` is entered in a chat where it is not registered as a built-in command, it is sent to the agent as ordinary text and does not start OAuth. Complete authorization in the browser, then start a new Kimi session. A server added by editing `mcp.json` is not registered in an already-open session. Use `/mcp` to inspect the connection and `get_instructions(level="experienced")` for the IWE instruction level recommended for this workspace.

## Verifying the Connection (Smoke Test)

1. **Kimi CLI and configuration are valid**:

   ```bash
   kimi --version
   kimi doctor
   ```

2. **IWE skill files are in place**:

   ```bash
   ls /path/to/IWE/.claude/skills/kimi-peer-writer/SKILL.md
   ls /path/to/IWE/.claude/skills/peer-conversation/SKILL.md
   ```

3. **The skill responds in Kimi Code**:

   ```text
   /skill:kimi-peer-writer --list
   ```

   This mode only reads `sessions/00-index.md` and does not call Claude. Before the first peer session the index may not exist; that means there is no peer-session history yet, not that skill loading failed.

4. **Kimi Standalone reads the IWE rules**:

   ```bash
   kimi -p "Read AGENTS.md, list three IWE rules, and do not modify files"
   ```

If a skill does not appear, verify `extra_skill_dirs`, run `kimi doctor`, and inspect `~/.kimi-code/logs/kimi-code.log` for invalid skill frontmatter.

## Operating Modes

### Kimi Standalone (Without Claude)

Kimi works alone for interactive tasks, manual standalone sessions, and scheduled headless tasks.

Load the installation paths and open a Session before edits:

```bash
source .iwe-paths
bash "$IWE_SCRIPTS/session-guard.sh" open --wp WP-N --task "..." --agent kimi
bash "$IWE_SCRIPTS/kimi-standalone-preflight.sh"
```

- `session-guard.sh open` — required Session opening before edits.
- `kimi-standalone-preflight.sh` — hard gate: checks that a Session is open and not stale (threshold `IWE_SESSION_STALE_THRESHOLD`, default 30 min).
- `kimi-auto-heartbeat.sh --interval 120` — background heartbeat for long-running headless sessions, so that `kimi-session-watchdog.sh` does not treat the Session as hung.
- `kimi-whisper-safe.sh` — safe audio transcription wrapper (see `docs/PLATFORM-COMPAT.md`, optional dependencies `ffmpeg`/`openai-whisper`).

### Optional: Kimi = Writer, Claude = Peer Partner

Skill: `/skill:kimi-peer-writer` (`.claude/skills/kimi-peer-writer/SKILL.md`). This mode requires an installed and accessible Claude Code CLI.

Kimi initiates the Session, writes the opening position, calls Claude via `scripts/claude-peer-adapter.sh`, runs the turn-loop until consensus, and — on the pilot's decision — implements the result.

### Optional: Kimi = Peer Partner, Claude = Writer

Skill: `/skill:peer-conversation` (`.claude/skills/peer-conversation/SKILL.md`). Claude initiates the Session and calls Kimi via `scripts/kimi-peer-adapter.sh`.

## Handoff With Claude

When a task is transferred between Kimi and Claude, use one of the mechanisms from `docs/inter-agent-handoff.md`:

- **Git commits + `Co-Authored-By`** — for tasks longer than 30 minutes:

  ```bash
  git commit -m "feat: ..." \
    --trailer "Co-Authored-By: Kimi <noreply@moonshot.ai>" \
    --trailer "Co-Authored-By: Claude <noreply@anthropic.com>"
  ```

- **`.handoff.md` bridge file** — for fast Iteration cycles of 5–15 minutes.
- **Branch-based relay** — for complex tasks involving multiple agents.

## Troubleshooting

1. Check that `extra_skill_dirs` points to the IWE `.claude/skills` directory.
2. Verify `merge_all_available_skills = true` and run `kimi doctor`.
3. Inspect `~/.kimi-code/logs/kimi-code.log` for skill parsing and MCP connection errors.
4. For knowledge-base authorization, run `/mcp-config login iwe-knowledge` and then start a new session.
5. For peer sessions only, confirm that the `claude` CLI is installed and accessible.
6. Check the peer session log: `sessions/00-index.md` after the first peer session.

## Related Documents

- `AGENTS.md` — rules for all agents.
- `docs/inter-agent-handoff.md` — context transfer between agents.
- `.claude/skills/kimi-peer-writer/SKILL.md` — Kimi as writer.
- `.claude/skills/peer-conversation/SKILL.md` — Kimi as Peer partner.
- `docs/skills-catalog.md` and `docs/scripts-catalog.md` — skill and Script catalogs.
