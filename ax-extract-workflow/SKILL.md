---
name: ax-extract-workflow
description: >-
  Reconstruct the workflow behind a delivered artifact using local ax data.
  Use when someone asks what made a feature, commit, demo, writeup, or shipped
  result work, or asks to recover the sessions, commits, skills, tool calls,
  and subagent activity behind it.
---

# Ax Extract Workflow

Reconstruct how a past result was produced by reading the local ax graph. Use
it to turn scattered session history, commits, skill usage, tool calls, and
subagent work into an ordered workflow narrative.

This skill is read-only by default. Do not edit repositories, create files,
commit changes, or publish reports unless the user explicitly asks for that as
a separate follow-up.

This skill assumes `ax` is installed and has already ingested the relevant
local sessions. If ax or its database is unavailable, tell the user what is
missing and stop.

## When to Use This Skill

- The user asks "how did we build this?", "what made this work?", or "how did
  this ship?"
- The user gives a commit SHA, date, topic, artifact name, feature name, demo,
  or writeup and wants the process behind it.
- The user wants to identify which skills, decisions, tools, and subagents
  contributed to a delivered result.
- The user wants a reusable recipe based on actual past work, not a
  speculative plan.

Do not use this for generic recent-activity summaries unless the user is trying
to reconstruct a specific result.

## What This Skill Does

1. **Resolve the anchor**: Start from a commit SHA, date, topic, or current
   repository window.
2. **Find relevant sessions**: Use ax session and recall queries to locate the
   work around that anchor.
3. **Inspect the evidence**: Read session JSON, role-grouped views, tool calls,
   commits, skill usage, and subagent activity.
4. **Narrate the workflow**: Present the ordered arc that produced the
   artifact, including key decisions and verification steps.
5. **Keep provenance visible**: Cite session ids, commit references, and
   subagent UUIDs where they support the reconstruction.

## How to Use

### Resolve the Anchor

Pick the narrowest reliable anchor the user provides:

- Commit SHA:

  ```bash
  ax sessions near <sha> --json --no-stale-check
  ```

- Date:

  ```bash
  ax sessions around <YYYY-MM-DD> --days=3 --json
  ```

- Current repository, recent work:

  ```bash
  ax sessions here --days=14 --json --no-stale-check
  ```

- Topic, feature, artifact, or phrase:

  ```bash
  ax recall "<topic>" --sources=turn,commit --json
  ```

If the topic search returns several plausible anchors, ask the user to choose
before reconstructing the workflow.

### Inspect the Sessions

For each likely session, inspect the structured record:

```bash
ax sessions show <id> --json
```

Use the JSON to identify:

- Commits and files touched
- Skill usage and role ordering
- Tool calls and verification commands
- User steering points and constraints
- Subagent dispatches, descriptions, and UUIDs

Then inspect skill ordering by role:

```bash
ax sessions show <id> --by-role
```

If a subagent appears central to the artifact, expand it:

```bash
ax sessions show <id> --expand=<subagent-uuid>
```

## Example

User prompt: "What made the dashboard live ingest work?"

Process:

```bash
ax recall "dashboard live ingest" --sources=turn,commit --json
ax sessions near <sha> --json --no-stale-check
ax sessions show <id> --json
ax sessions show <id> --by-role
ax sessions show <id> --expand=<subagent-uuid>
```

Output:

```text
Workflow behind dashboard live ingest

Anchor: commit <sha>
Sessions: <id>, <id>

1. Framing: identified live ingest as a same-origin dashboard workflow.
2. Planning: split work into server stream, browser subscription,
   and fallback behavior.
3. Implementation: added the ingest stream path and wired progress
   events through the dashboard API.
4. Subagent work: expanded <subagent-uuid> to recover browser-side
   validation.
5. Verification: confirmed the API path and reconnect behavior with
   recorded test commands.

Key decisions:
- <session id>: Kept CLI ingest behavior unchanged.
- <session id>: Used a durable stream so refreshes can rehydrate progress.
- <session id>: Added polling fallback for builds without live ingest.
```

## Tips

- Prefer commit SHAs over topic search when the user has one; they produce
  tighter session windows.
- Treat ax data as evidence, not memory. Cite the session id or commit behind
  important claims.
- Include negative evidence when useful, such as missing verification,
  abandoned approaches, or subagents that were dispatched but did not affect
  the final artifact.
- Keep the final answer inline unless the user explicitly asks for a file,
  issue, PR body, or other durable artifact.

## Common Use Cases

- Recovering the recipe behind a shipped feature.
- Explaining how a successful demo or writeup was produced.
- Auditing which skills and subagents contributed to a result.
- Turning past agent work into a concise process brief for future runs.
