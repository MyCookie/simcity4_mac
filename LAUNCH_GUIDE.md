# SimCity 4 Deluxe (Mac Aspyr Port) — Launch Options Guide

The Aspyr Mac port replaces the Windows INI-file and CLI-argument system with **NSUserDefaults** (macOS's built-in preference system). Preferences are stored in `~/Library/Preferences/com.aspyr.simcity4.plist` and can be set via the `defaults` command.

## How Launch Works

The launch chain is:

1. **Aspyr GG3 Launcher** calls `execv()` → game binary starts
2. **Game binary** reads NSUserDefaults, builds a Windows-style argument string
3. **Game binary relaunches itself** so the native engine processes the CLI flags

This means: most Windows command-line flags work, but they reach the engine through the preference bridge (step 2) rather than being parsed on first launch. You can also pass arguments directly via Steam launch options, which are forwarded by the launcher through `execv()`.

## NSUserDefaults Preference Keys

Set these with `defaults write com.aspyr.simcity4 <key> <value>`:

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `fullscreen-mode` | bool | `1` | `1` = fullscreen, `0` = windowed |
| `intro-movie` | bool | `1` | `0` = skip intro videos |
| `custom-resolution` | bool | `0` | `1` = enable custom resolution |
| `custom-width` | int | `1024` | Custom width in pixels |
| `custom-height` | int | `768` | Custom height in pixels |
| `display.index` | int | — | Monitor selection |
| `display.rect` | rect | — | Display bounding rect |
| `display.remember` | bool | `0` | Remember display settings |

## Examples

### Windowed mode
```bash
defaults write com.aspyr.simcity4 fullscreen-mode 0
```

### Skip intro videos
```bash
defaults write com.aspyr.simcity4 intro-movie 0
```

### Custom resolution (e.g. 1920x1080)
```bash
defaults write com.aspyr.simcity4 custom-resolution 1
defaults write com.aspyr.simcity4 custom-width 1920
defaults write com.aspyr.simcity4 custom-height 1080
```

The game appends `-r1920x1080x32` and `-CustomResolution:enabled` to its relaunch arguments, same as the Windows version.

### Reset to defaults
```bash
defaults delete com.aspyr.simcity4 fullscreen-mode
defaults delete com.aspyr.simcity4 custom-resolution
defaults delete com.aspyr.simcity4 custom-width
defaults delete com.aspyr.simcity4 custom-height
defaults delete com.aspyr.simcity4 intro-movie
```

### View all current settings
```bash
defaults read com.aspyr.simcity4
```

## Steam Launch Options

Steam launch options are forwarded by the launcher via `execv()`. The launcher strips its own `argv[0]` and passes everything else through to the game binary. The game's second-invocation engine then parses these flags normally.

Example Steam launch options that work:
```
-w -intro:off -CustomResolution:enabled -r1920x1080x32
```

Note that `-f` and `-w` set by Steam launch options may be overridden by the NSUserDefaults `fullscreen-mode` key, since the preference bridge runs before the relaunch. For reliable windowed/fullscreen control, use `defaults write` instead.

## Working Windows CLI Flags

These flags from the Windows version are processed natively by the engine on the second invocation and should work on macOS when passed as Steam launch options:

- `-f` : Fullscreen
- `-w` : Window mode
- `-Intro:off` : Skip intro videos
- `-r{WIDTH}x{HEIGHT}x32` : Custom resolution
- `-CustomResolution:enabled` : Enable custom resolution support
- `-CPUCount:X` : Limit CPU cores used
- `-CPUPriority:low/high` : CPU thread priority
- `-Cursors:disabled/bw/color16/color256/fullcolor` : Cursor color depth
- `-d:Software/OpenGL` : Rendering mode (macOS likely ignores `DirectX`)
- `-audio:off` : Disable audio
- `-BackgroundLoader:on/off` : Background loading
- `-ExceptionHandling:off` : Disable exception handling
- `-gp` : Pause when backgrounded
- `-IgnoreMissingModelDataBugs:on/off` : Brown box removal
- `-IME:enabled/disabled` : Input method editor
- `-l:english` : Language
- `-UserDir:path` : Custom user directory path (use `/`, not `\`, on macOS)
- `-WriteLog:on/off` : Write config log

## Key Differences from Windows

| Feature | Windows | Mac |
|---------|---------|-----|
| Settings storage | `SimCity 4.ini`, `GZGraphic2.ini`, registry | `NSUserDefaults` (`~/Library/Preferences/com.aspyr.simcity4.plist`) |
| Custom resolution | `-rWxHx32` on command line | `defaults write` keys or Steam launch options |
| Windowed mode | `-w` flag | `defaults write com.aspyr.simcity4 fullscreen-mode 0` |
| Rendering mode | `-d:DirectX/Software/OpenGL` | OpenGL only; `DirectX` ignored |
| Plugins path | `~/Documents/SimCity 4/Plugins/` | Same location (`~/Documents/SimCity 4/Plugins/`) |
| Music path | `Radio/Stations/Region/Music` | Same, inside the app bundle's `Contents/Resources` |
| Startup behavior | Parses CLI args directly | Self-relaunches: NSUserDefaults → CLI args → exec |

## Cheat Codes

Same as Windows — `Cmd+X` opens the cheat console:

- `dollyllama`: Advisors as llamas
- `fightthepower`: No power requirement
- `howdryiam`: No water requirement
- `moneyeo`: 10,000,000§
- `moolah`: X§ (add amount after space)
- `weaknesspays`: 1,000§
- `stopwatch`: Pause clock
- `whatimeizit`: Set time
- `whererufrom`: Change city name
- `hellomynameis`: Change mayor name
- `tastyzots`: Toggle zots
- `zoneria`: Hide empty zone color
- `you don't deserve it`: All rewards
- `recorder`: Start recorder
- `sizeof`: Magnify (1-100)