---
name: notify
description: Message David on Sage (Discord) when work finishes — a quick ping for small things, or a brief + share link when there's real detail to read. Trigger proactively when a significant task completes (a feature shipped, a bug fixed, a deploy done, a long-running job finished) or whenever David asks to be notified/messaged/pinged. Skip for small edits, mid-conversation answers, or read-only investigation — most turns should NOT trigger this.
---

# Notify via Sage

David wants Claude Code to reach him through Sage (his Discord bot), not just end
the response silently. Pick one of two shapes based on how much there is to say.

## Quick ping — nothing more to read

The message *is* the whole update: a one-line status, a deploy confirmation, "done,
nothing broke." No share page needed.

```bash
notify-sage "TITLE" "one or two sentence summary" info
```

`notify-sage` is on PATH (`~/.local/bin/notify-sage`). Args: TITLE, BODY, LEVEL.
LEVEL is `info` for routine completions — reserve `warn`/`crit` for things that
actually need David's attention (a failure, something broken), not normal "done."

## Full brief — there's real detail (multi-step session, decisions made, things that
failed, a diff worth reviewing)

Run the **share** command (`/share`) end to end. Its last step already posts the
TL;DR + public URL to Sage via `notify-sage` — don't call `notify-sage` again
yourself after running it, that would double-post.

## When to trigger

Proactively, at the end of work David would otherwise have to come back and check
on: a feature shipped, a bug fixed, a deploy or long-running build finished. Also
whenever David explicitly asks to be notified/messaged/pinged.

Do NOT trigger for: small edits, mid-conversation answers, read-only investigation,
or anything still in progress. When unsure whether it's "significant," default to
not sending — false pings erode trust in the channel faster than a missed one does.
