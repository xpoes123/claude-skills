Help configure, troubleshoot, or reload Hyprland window manager settings.

David is on CachyOS with Hyprland as his window manager. Config lives at `~/.config/hypr/hyprland.conf`. He is a Linux beginner — explain what things do before changing them.

## If the user provides arguments: $ARGUMENTS
Treat as what they want to do (e.g. "add a keybind", "change wallpaper", "fix my monitor").

## Common tasks

### Reload config (no restart needed)
```
hyprctl reload
```
Most changes to hyprland.conf apply immediately after this. Tell the user to run it after any edit.

### Check what's running / connected
```
hyprctl monitors    # display info
hyprctl clients     # open windows
hyprctl workspaces  # workspace list
```

### Add a keybind
Keybinds go in hyprland.conf under the `### KEYBINDINGS ###` section:
```
bind = $mainMod, KEY, exec, command
```
Example — open Brave with Super+B:
```
bind = $mainMod, B, exec, brave
```

### Wallpaper (awww)
awww is installed. To set a wallpaper:
```
awww img /path/to/image.jpg
```
For a transition effect:
```
awww img /path/to/image.jpg --transition-type wipe --transition-duration 1
```
To set wallpaper on startup, add to hyprland.conf exec-once:
```
exec-once = awww-daemon && awww img /path/to/image.jpg
```

### Waybar
Config lives at `~/.config/waybar/config.jsonc` and `~/.config/waybar/style.css`.
Restart waybar after changes:
```
pkill waybar; waybar &
```

### Check Hyprland logs for errors
```
journalctl --user -u hyprland --since today | tail -50
```
Or check the runtime log:
```
cat /tmp/hypr/$(ls /tmp/hypr/)/hyprland.log | tail -50
```

## Notes
- $mainMod is set to SUPER (the Windows key) in David's config
- David has a 4K monitor (3840x2160) at 120Hz with 1.5x scaling
- Always explain what a config change does before making it
