---
name: memory-refresh
description: Weekly headless sweep of this week's Claude Code session transcripts to catch memory-worthy facts that didn't get saved live. Runs Sunday morning via systemd timer; also invokable on demand as /memory-refresh. Writes only to ~/.claude/projects/-home-david/memory/.
---

# Memory Refresh

You are running headless, with no user to ask questions. Your job: read this
week's Claude Code session transcripts and catch what should have been saved
to the persistent memory system but wasn't — because a live session ended,
compacted, or moved on before something worth keeping got written down.

You already know the memory system's rules (types, save format, MEMORY.md
index format, what NOT to save) from your standing auto-memory instructions.
Apply them exactly as you would mid-conversation — this is not a different,
looser bar. If you're unsure whether something clears the bar, don't save it.

## 1. Find this week's transcripts

```bash
find ~/.claude/projects -name '*.jsonl' -mtime -7 -not -path '*/-home-david/memory/*'
```

That's likely dozens of files across many project dirs. Don't read them all
serially in this context — you'll blow your budget before finishing.

## 2. Fan out

Group the files by project directory. Spawn one subagent per project
directory with 7-day-old activity (skip empty ones). Give each subagent:
- The list of its transcript files.
- The current `MEMORY.md` index (so it doesn't propose duplicates of what's
  already tracked).
- Its job: skim each transcript for facts matching the four memory types
  (user / feedback / project / reference) — real decisions, corrections,
  confirmations, preferences, deadlines — not code changes or routine tool
  output. Return a short structured list of candidates: type, one-line fact,
  file/line or timestamp it came from, and why it clears the bar (or "skip,
  nothing new" if the week was routine).
- Explicitly read-only: subagents report candidates, they do not write memory
  files themselves.

## 3. Synthesize

As the lead, review every candidate against the existing memory files (Read
the specific file, not just the index line) before writing anything:
- If it updates an existing memory (a project's status changed, a preference
  was refined), edit that file in place rather than creating a near-duplicate.
- If it's genuinely new, create it following the exact frontmatter/type/
  linking format the auto-memory instructions specify, and add one index line
  to `MEMORY.md`.
- Skip anything borderline. This runs unattended — false positives pollute
  memory for every future session, false negatives just wait for next week
  or a live conversation to catch them. Bias toward skipping.
- Cap yourself at a handful of the most load-bearing facts. If a week
  produced 20 candidates, that's a sign your bar was too low, not that 20
  memories are warranted.

## 4. Log and finish

Append one line to `~/.local/share/audits/memory-refresh.log`:
`<date>: N transcripts scanned, M candidates, K saved — <one-line summary>`.
No Discord post, no other output — this is quiet infrastructure, not a
report. If nothing was worth saving, say so in the log and stop; that's a
normal, good outcome, not a failure.
