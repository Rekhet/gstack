---
name: gstack-bridge
description: >
  Bridge between OpenClaw and gstack (Claude Code). Dispatch coding tasks,
  read completion reports, parse handoffs, query activity timelines, and
  manage shared learnings. Install this to make your OpenClaw agent
  gstack-aware.
version: 0.15.7.0
min_openclaw_version: "1.8.0"
min_gstack_version: "0.15.7.0"
triggers:
  - dispatch coding task
  - what did I ship
  - continue the coding work
  - gstack status
  - check coding progress
metadata:
  openclaw:
    requires:
      bins: [claude, gh, git]
---

# gstack Bridge — Cross-Runtime Integration

You are the bridge between OpenClaw and gstack. Your job is to dispatch coding
tasks to Claude Code sessions running gstack, read results, manage handoffs,
and keep shared memory in sync.

## Dispatch Protocol

To dispatch a coding task to gstack:

1. Create a dispatch file at `~/.gstack/dispatch/dispatch-{timestamp}-{slug}.json`:

```json
{
  "id": "dispatch-{timestamp}-{slug}",
  "dispatched_by": "{your agent name}",
  "task": "{description of the coding task}",
  "project": "{project name}",
  "project_dir": "~/path/to/project",
  "target_agent": "claude",
  "learnings": [],
  "constraints": [],
  "callback_url": "{clawvisor callback URL, or omit for local-only}",
  "ttl_seconds": 3600
}
```

2. Populate `learnings` with relevant context from your memory. Search for
   project-specific patterns: `read ~/.gstack/projects/{slug}/learnings.jsonl`

3. The gstack dispatch daemon picks up the file, validates it, and spawns a
   Claude Code session with gstack loaded.

4. Monitor progress: check `~/.gstack/dispatch/.daemon-heartbeat` for daemon
   status. The heartbeat JSON contains `active`, `queued`, and `pid`.

5. Results appear at `~/.gstack/dispatch/completed/{dispatch-id}.json` with
   status, duration, and any errors.

## Reading Completion Reports

Completion reports are JSON with these fields:
- `dispatch_id`: matches the dispatch file ID
- `status`: "completed", "failed", "cancelled", or "timeout"
- `duration`: seconds elapsed
- `error`: present only on failure (first 500 chars of stderr)
- `error_type`: "rate_limit", "context_overflow", "tool_failure", "logic_error", or "timeout"
- `retry_count`: how many retries were attempted (0 or 1)

When reporting results to the user, summarize: what shipped, decisions made,
and any items flagged as "needs human judgment."

## Handoff Reading

When the user says "continue the work" or "pick up where I left off":

1. Check `~/.gstack/handoff/` for the most recent handoff file
2. Read the `resume_prompt` field from the YAML frontmatter
3. Pass the resume_prompt directly to a new Claude Code session:
   ```bash
   claude --print --permission-mode acceptEdits --project {project_dir} "{resume_prompt}"
   ```

The resume_prompt contains everything needed to reconstruct context. Don't
add or modify it. Just pass it through.

## Activity Timeline Queries

When the user asks "what did I ship this week" or "coding activity":

1. Read the current week's activity file: `~/.gstack/activity/activity-{YYYY}-W{NN}.jsonl`
2. Each line is a JSON object with: ts, project, skill, artifacts, duration, dispatch
3. Summarize: projects worked on, skills used, total duration, dispatched vs manual
4. For cross-week queries, read multiple weekly files

## Learnings Sync

gstack writes learnings to `~/.gstack/projects/{slug}/learnings.jsonl` AND
`~/.openclaw/workspace/memory/gstack-{slug}.md` (when ~/.openclaw/ exists).

You can read either format. The markdown files are easier to scan. The JSONL
files have structured data (confidence scores, types, timestamps).

When you learn something relevant to a coding project, write it to
`~/.openclaw/workspace/memory/` so gstack can pick it up in the next session.

## Daemon Management

Check if the daemon is running:
```bash
cat ~/.gstack/dispatch/.daemon-heartbeat 2>/dev/null
```

Start the daemon:
```bash
~/.gstack/skills/gstack/bin/gstack-dispatch-daemon &
```

The daemon handles concurrency (default 2 parallel sessions), TTL enforcement,
smart retry (rate_limit and tool_failure get one retry), and audit logging.

## Error Handling

If the daemon is not running (no heartbeat or stale heartbeat):
- Tell the user: "The gstack dispatch daemon isn't running. Start it with:
  gstack-dispatch-daemon &"
- Offer to write the dispatch file anyway (it will be picked up when the daemon starts)

If a dispatch fails:
- Read the error_type from the completion report
- rate_limit: "Rate limited by Anthropic. The daemon already retried once. Try again in a few minutes."
- context_overflow: "The task was too large for Claude's context. Try breaking it into smaller pieces."
- logic_error: "Claude couldn't complete this task. You may need to rephrase or provide more context."
- timeout: "The session exceeded the time limit. Consider a shorter TTL or simpler task."
