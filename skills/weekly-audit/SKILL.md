---
name: weekly-audit
description: Weekly recap (runs headless on a schedule, also invokable on demand). A LEAD agent fans out to several subagents and synthesizes a full weekly recap — desktop+VPS health, a projects roundup, an analysis of how you used Claude Code this week, and coaching on what to do differently: paradigms to try, agent-orchestration and infra improvements, and skills/hooks/workflows worth building. Writes a markdown recap to disk and (optionally) posts a digest somewhere.
---

# Weekly Recap

You are the **LEAD** agent producing a weekly recap. This is a **multi-agent job**: you
dispatch several subagents (the Task/Agent tool), then synthesize their output. Runs
headless on a schedule (e.g. a systemd timer or cron); also invokable on demand.

## THE ONE HARD RULE — READ-ONLY
You and every subagent observe and report only. **Never** modify state: no
installs/removals, `rm`, package-cache clears, service enable/disable/restart, config
edits, git writes, or any privileged write. Notice something worth fixing → write it as
a recommendation. The only files created are the recap and scratch files.

## 1. Setup
```bash
DATE=$(date +%F)
mkdir -p ~/.local/share/audits
REPORT=~/.local/share/audits/$DATE.md
```
If you keep a daily usage-extract job, pull the last 7 days of that data here too.

## 2. Dispatch the subagents
Spawn A, B, C **in parallel** (independent). Then run D (it needs C's output). Give
each the READ-ONLY rule. Have each **return structured findings**, not prose.

### Agent A — Ops (desktop + any servers you run)
Run and report, grouped 🔴/🟡/🟢 with the fix command as a *suggestion*:
- Desktop: disk usage, orphaned packages, package cache size, pending updates, failed
  systemd units, CPU governor, failed auth attempts in the last 7 days.
- Any VPS/server via SSH: disk (space + inodes), memory, failed units, per-service
  health for whatever you run in production, pending OS updates, reboot-required flag,
  failed SSH logins, firewall rules with no matching listener, fail2ban status,
  disk-hog directories, any secrets file that's world/group-readable (flag as 🔴).
- TLS expiry for your domains — flag anything within 21 days.
- Backup freshness — flag if the latest backup is older than expected.
- If you track metrics over time, report the week's trend (disk %, memory, whatever
  you track).

### Agent B — Projects roundup
For each git repo you maintain: commits in the last 7 days, last-commit age, current
branch, dirty/unpushed status, missing remote. Flag ⚠️ for anything dirty/unpushed/
off-branch, 💤 for anything untouched 30+ days. One line per repo, most-active first.

### Agent C — Claude Code usage + shell habits (this week)
**Claude Code:** aggregate whatever usage data you log — sessions, tool-call mix,
subagents vs. workflows, skills invoked, MCP calls, permission denials, models used,
recurring friction. Call out the 3-4 most decision-relevant facts.

**Shell habits** (if you use a shell-history tool with a queryable DB, e.g. atuin):
top commands, failure rate, busiest directories, time-of-day rhythm. Feed anything
notable (a command run constantly that should be an alias/script) to Agent D.

### Agent D — Coach (run AFTER C — pass it C's summary)
First inventory what you already have (skills, commands, hooks, permissions,
workflows, MCP servers). THEN, using C's usage summary + that inventory + the
projects context, produce **prioritized, concrete** coaching — every point grounded
in an observed number from C:
- Under-used paradigms (heavy manual fan-out but no Workflows, repeated manual
  command sequences that want a skill/hook, permission-denial hotspots that want an
  allowlist rule, sessions that want subagent-driven-development or plan mode).
- Agent-orchestration / infra restructuring ideas.
- A build-this-week list: 1-4 named artifacts, each with a one-line why tied to data.
No generic advice — if it isn't tied to something in the numbers, cut it.

## 3. Synthesize the recap → `$REPORT`
Markdown: week-at-a-glance (1-2 lines), this week's highlights, system health
(grouped, fixes as suggestions), projects, Claude Code usage, coaching (give this the
most room), and a diff against last week's recap if one exists.

## 4. Post a digest (optional)
If you have a notification channel, build a short digest (≤ ~1800 chars) leading with
2-3 coaching highlights, then one-liners for health/projects/usage, then the report
path. Severity = worst finding across the week.

## 5. Finish
Print the week-at-a-glance line + the report path. Leave no background processes
running.
