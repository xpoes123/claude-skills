# claude-skills

Personal [Claude Code](https://claude.com/claude-code) skills and slash-commands, pulled out of my
`~/.claude` config because they turned out useful beyond my own machine.

Skills (`skills/*/SKILL.md`) are auto-invoked by Claude Code when their description matches what
you're doing. Commands (`commands/*.md`) are explicit slash-commands (`/name`).

**These reference my machine (CachyOS/Hyprland), my VPS, and my domains as examples.** Swap paths,
hostnames, and service names for your own before using — a couple of the infra-heavy ones
(`vps`, `weekly-audit`, `share`) had specific hostnames/IPs/service maps stripped and replaced
with placeholders on purpose; fill in your own.

## Skills

| Skill | What it does |
|---|---|
| `memory-refresh` | Headless weekly sweep of Claude Code session transcripts to catch memory-worthy facts that didn't get saved live. Fans out one subagent per active project dir, synthesizes candidates against an existing memory index. |
| `notify` | Ping a Discord bot (or any webhook) when a task finishes — quick ping vs. full brief, with rules for when NOT to notify (most turns shouldn't). |
| `propose` | Instead of dumping a design/spec/options wall-of-text in chat, render an interactive reviewable HTML page (card-per-decision, react + copy-feedback) and link it. |
| `weekly-audit` | Multi-agent weekly recap: parallel subagents for ops health, project activity, Claude Code usage stats, and a "coach" agent that turns the usage data into concrete next-step recommendations. |

## Commands

| Command | What it does |
|---|---|
| `pkg` | Install/search/remove packages on an Arch-based system (pacman + AUR helper). |
| `hypr` | Configure/troubleshoot Hyprland (Wayland compositor). |
| `bookmark` | Append an entry to a rofi bookmarks launcher file. |
| `run` | Turn a described task into a one-liner and add it to a rofi command menu. |
| `transcript` | Pull + clean a YouTube video's transcript via yt-dlp. |
| `vault` | Get/search/add/edit entries in a Bitwarden vault via `rbw`. |
| `vps` | Template for VPS ops (status/logs/deploy/add-subdomain) via SSH + Caddy + systemd. Fill in your own host and service table. |
| `share` | Template for a git-backed "publish a page to my own domain" pull-to-deploy flow. |

## License

MIT — take whatever's useful.
