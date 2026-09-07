Add a bookmark to the rofi bookmarks launcher at ~/.config/rofi/bookmarks.

File format is one bookmark per line: `Name|target`

Target types:
- URL (https://...) — opens in Zen Browser on workspace 9
- `dir:/path/to/dir` — opens a kitty terminal in that directory
- `code:/path/to/dir` — opens Claude Code in that directory

If the user gives just a URL, derive a short friendly name from the domain.
If they give a path, use the folder name as the display name unless they specify one.
Append the new entry to the file and confirm what was added.
