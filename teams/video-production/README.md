# Video production team

A team of Claude Code subagents that turns a planned series of short vertical videos
(brief → voice → motion graphics → render → review → scheduled post) into finished,
scheduled posts. Your main session is the **Director**; the agents are the crew.

Built for a Hyperframes (HTML → video) + ElevenLabs + studio-MCP (Instagram/Facebook
scheduling) pipeline, but each agent reads its commands from the project's own runbook,
so the same team works for any project that has one.

## The crew

| Agent | Job | Hands off |
|---|---|---|
| Director (your main session, prompt below) | picks jobs, starts agents, routes fixes, reports | — |
| `video-producer` | reads the brief + runbook, pre-flight checks, writes `JOB.md` | READY / BLOCKED |
| `video-sound` | ElevenLabs voice, caption timings, audio assets | narration.mp3, captions.json |
| `video-motion` | motion-graphics scenes timed to the captions | build script |
| `video-monitor` | joins motion + voice, sync checks, `hyperframes check` gate | READY / BOUNCE |
| `video-render` | final MP4, cover frame, contact sheet | render + stills |
| `video-qa` | visual + compliance review, routes each fix to one role | PASS / FAIL |
| `video-publisher` | uploads and creates IG + FB posts (pending owner approval) | post ids |
| `video-archivist` | per-video docs, series README, backup | DONE |
| `video-analyst` (optional) | reads insights, recommends changes for unbuilt videos | recommendations |

Everything passes through one file per video, `<project folder>/JOB.md`: each agent
appends `| time | role | status | note |` to its handoff log. That makes runs resumable.

## Install

```bash
# user scope (all projects on this device)
mkdir -p ~/.claude/agents && cp agents/video-*.md ~/.claude/agents/
# or project scope
mkdir -p .claude/agents && cp ~/claude-agents/agents/video-*.md .claude/agents/
```

Requires on the device: `ffmpeg`/`ffprobe`, Node (for `npx hyperframes`), Python 3,
an exported `ELEVENLABS_API_KEY`, and the studio MCP server (tools `mcp__studio__*`)
for the publisher and analyst.

The project must provide: a runbook (rules, build/voice/render/stills commands, banned
claims), one brief per video (approved narration, beats, allowed UI labels, captions,
publish times), and a base build to copy.

## Director starter prompt

Save as `.claude/commands/video-team.md` in the project (then run `/video-team brag_11`),
or paste it into a fresh session. Fill the `<…>` placeholders.

```markdown
---
description: Director for the video team. Builds the planned videos with the video-* agents and schedules them.
argument-hint: "[which videos, e.g. 'ep_11' or 'ep_11-ep_14' or 'next']"
---
You are the Director of the video team. You manage; the agents do the work: pick the
jobs, start the right agent at the right time, read their handoff lines, route fixes,
report. Don't build, voice or render yourself unless an agent is unavailable.

Videos this run: $ARGUMENTS (empty = the next calendar video without `archivist | DONE`).

Sources of truth: runbook <RUNBOOK PATH>; calendar <PLAN PATH>; briefs <BRIEFS GLOB>;
project folders <PROJECTS DIR>. Each job's state is <project folder>/JOB.md; re-read it
before every decision.

Order per video: video-producer → video-sound → video-motion → video-monitor →
video-render → video-qa → video-publisher → video-archivist (video-analyst on request).
Give each agent the absolute JOB.md path and a one-line goal. Agents run in the
background: wait for each notification, never guess a result.

Pipelining: one video at a time from motion to archivist; while video N is in
monitor/render/qa you may run producer + sound for N+1. Never two renders at once.

Routing: qa FAIL → the named role, then monitor → render → qa (max 2 loops, then stop
that video). monitor BOUNCE → named role → monitor. BLOCKED → stop that video, report
what the owner must do.

Owner rules: <APPROVAL POLICY, e.g. "no approval stops; narrations approved; push posts
as pending, the owner approves in the studio">. Hard stops: missing API key, voice not
in the account, an account with auto_approve on, repeated QA failure. Never deploy,
never put secrets in files, one TTS call per video unless QA routes an audio fix.

End with a table: video · status · IG id + time · FB id + time · fallbacks used ·
owner to-dos; then the one next action.
```
