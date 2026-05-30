# SimCity 4 Deluxe Edition — macOS Aspyr Port

Documentation and analysis of the Aspyr Game Guide 3 (GG3) launcher and the game binary's NSUserDefaults-based preference system for the macOS port of SimCity 4 Deluxe Edition.

The bundle ID (preferences domain) depends on the distribution channel:

| Channel | Bundle ID |
|---------|-----------|
| Steam | `com.aspyr.simcity4.steam` |
| Mac App Store | `com.aspyr.simcity4` (untested) |

> Substitute the correct bundle ID from the table above whenever you see `com.aspyr.simcity4` in the examples below.

## Translating Windows CLI Flags to macOS

The Windows version uses command-line arguments. The Mac version stores the equivalent settings in NSUserDefaults. Here's how to translate common configurations:

### Windowed mode + skip intro

**Windows:** `SimCity 4.exe -w -intro:off`

**Mac:**
```bash
defaults write com.aspyr.simcity4 fullscreen-mode 0
defaults write com.aspyr.simcity4 intro-movie 0
```

### Custom resolution 1920x1080, windowed

**Windows:** `SimCity 4.exe -w -CustomResolution:enabled -r1920x1080x32`

**Mac:**
```bash
defaults write com.aspyr.simcity4 fullscreen-mode 0
defaults write com.aspyr.simcity4 custom-resolution 1
defaults write com.aspyr.simcity4 custom-width 1920
defaults write com.aspyr.simcity4 custom-height 1080
```

### Fullscreen, no intro, limited to 2 CPU cores

**Windows:** `SimCity 4.exe -f -intro:off -CPUCount:2`

**Mac:**
```bash
defaults write com.aspyr.simcity4 fullscreen-mode 1
defaults write com.aspyr.simcity4 intro-movie 0
```

The CPU count flag can be passed via Steam launch options since the launcher forwards arguments:
```
-CPUCount:2
```

### Reset everything to defaults

```bash
defaults delete com.aspyr.simcity4 fullscreen-mode
defaults delete com.aspyr.simcity4 intro-movie
defaults delete com.aspyr.simcity4 custom-resolution
defaults delete com.aspyr.simcity4 custom-width
defaults delete com.aspyr.simcity4 custom-height
```

See [LAUNCH_GUIDE.md](LAUNCH_GUIDE.md) for the full reference, and [LAUNCHER_ANALYSIS.md](LAUNCHER_ANALYSIS.md) for the detailed technical analysis of the launcher and game binary.
