# Claude Agents

My portable collection of AI agents — version-controlled so the same agents
work across every device and every project I touch.

The headline idea: **an agent is just a markdown file with frontmatter + a system
prompt.** Nothing magic. Once you internalize that, you can build, version, and
sync them like any other code.

---

## Repo layout

```
.
├── agents/              # one .md file per agent
│   └── e2e-tester.md
├── install.sh           # links agents into ~/.claude/agents on this device
├── templates/
│   └── agent-template.md  # starting point for new agents
└── README.md
```

---

## Why this repo exists (the portability problem)

Claude Code looks for agents in two places on disk:

| Location | Scope | Synced across devices? |
|---|---|---|
| `<project>/.claude/agents/` | that one project | no — lives in the repo |
| `~/.claude/agents/` | every project on **this device** | no — lives on this machine |

Neither follows me to a new laptop, the server, or a client machine. The fix is
to treat agents as **dotfiles**: keep them in this Git repo, then symlink them
into `~/.claude/agents/` on each device. Edit once, `git push`, `git pull`
anywhere, and every machine is updated.

---

## Setup on a new device

```bash
git clone <this-repo-url> claude-agents
cd claude-agents
chmod +x install.sh
./install.sh
```

`install.sh` symlinks every file in `agents/` into `~/.claude/agents/` (user
scope = available in all projects on this device). Because they're symlinks, a
later `git pull` updates the live agents automatically — no re-install needed.

### Per-project alternative

If I want an agent scoped to a single project (committed and shared with a team),
copy it into that project instead:

```bash
mkdir -p .claude/agents
cp ~/claude-agents/agents/e2e-tester.md .claude/agents/
```

---

## Dependencies are per-machine, not in this repo

Agents that use MCP tools depend on those MCP servers being **registered on the
device**. The agent file is portable; the MCP registration is not. `install.sh`
handles the ones I rely on, but the rule to remember:

> The `.md` file says *which* tools the agent may use.
> The MCP registration on the machine decides whether those tools *exist*.

Verify any device with `claude mcp list` or `/mcp` inside a session.

---

## Agents in this repo

### `e2e-tester`

Full end-to-end browser testing of a frontend. Explores the app like a human QA
engineer, exercises every interactive element, runs realistic user scenarios,
and writes a dated bug report (`test-reports/YYYY-MM-DD-e2e-test.md`).

**Framework-agnostic** — detects the stack (React, Vue, Svelte, Next, etc.) in
Phase 1, then tests the rendered browser UI, which is the same regardless of what
built it.

**Requires:** the Playwright MCP server.

```bash
claude mcp add --scope user playwright npx @playwright/mcp@latest
```

(Node 18+. The `executeautomation` package is a different community fork — I use
Microsoft's official `@playwright/mcp`.)

**Usage** — in Claude Code, plain language is enough:

```
Use the e2e-tester agent to do a full e2e test of the frontend.
The dev server is running at http://localhost:5173
```

The main Claude reads the agent's `description`, sees the match, and delegates.
The subagent runs in its own context and hands back only the summary + report
path, keeping the main conversation clean.

---

## How agents work (the model, so future-me remembers)

### Anatomy of an agent file

```markdown
---
name: agent-name
description: When the main model should delegate to this agent. Write it as
             "Use this when..." with concrete triggers — this is the router.
tools: [explicit allow-list of tools the agent may use]
model: sonnet
---

The system prompt. Everything below the frontmatter is the agent's
instructions / personality / process.
```

### The three things that matter

1. **The file is the agent.** Frontmatter + system prompt. Version it like code.

2. **`description` is the router.** It's how delegation gets decided. Vague
   description → never triggers. "Use this when..." with concrete triggers →
   reliable delegation.

3. **Tool restriction is the point.** Give an agent only the tools it needs.
   `e2e-tester` gets Read/Glob/Grep (explore), Write (report), Bash (check the
   server), and the Playwright browser tools — but **not** Edit, because it reads
   and reports, it should never touch source code. Constraining tools is how you
   make an agent safe and focused.

### The reusable skeleton

Every good agent follows the same four-phase bones:

```
1. Understand   — gather context before acting (read code, detect stack, scope)
2. Plan         — produce an explicit checklist / task list first
3. Execute      — do the work methodically, verifying each step
4. Report       — hand back a clean, structured result
```

Swap the domain and the skeleton still holds. "Browser testing" → "security
review" → "dependency audit" → "API contract check" all fit this shape.

---

## Adding a new agent

1. Copy `templates/agent-template.md` to `agents/my-new-agent.md`.
2. Fill in name, description, tools, and the four phases.
3. Register any MCP servers it needs (and add them to `install.sh`).
4. Commit and push. Run `./install.sh` (or just `git pull` if symlinks exist).

---

## Notes to self

- Keep `description` concrete — it's the single biggest factor in whether
  delegation actually fires.
- Restrict `tools` deliberately. Default to least privilege; add Edit/Write
  only when the agent genuinely needs to change things.
- Test a new agent in a throwaway project before trusting it on real work.
- If an agent feels too broad, split it. Two sharp agents beat one vague one.
