# SimCity 4 Deluxe Edition Launcher — Analysis

## Overview

This binary is **Aspyr's Game Guide 3 (GG3)** launcher framework. It's a generic game launcher used by Aspyr for multiple Steam and Mac App Store titles, not code specific to SimCity 4. All game-specific configuration (executable name, bundle ID, Steam app ID, etc.) comes from the app's `Info.plist`.

The launcher's purpose: display a branded WebKit-based launcher window, handle Steam/MAS DLC, and then **replace itself** with the actual game process.

## How the Launch Works

When the user clicks the **Launch** button:

1. `launchGame:` is called (at `0x1003695c0`). It sets a launching-in-progress flag, then either calls `finishLaunch` immediately or schedules it with a brief delay via `performSelector:withObject:afterDelay:`.

2. `finishLaunch` (at `0x100369640`) executes the actual launch:

   - Calls `perAppFinishLaunch` and `closeDown` (closes the launcher UI)
   - Reads `Info.plist` for the game's launch configuration
   - **Branches on whether this is an App Store build or Steam build**

### Steam Build Path (default)

The launcher constructs the game executable path as:

```
[NSBundle mainBundle].bundlePath + "/Contents/MacOS/" + <GameGuide.SteamDefaultPlayExecutable>
```

The `SteamDefaultPlayExecutable` value comes from `Info.plist` under the key path `GameGuide.SteamDefaultPlayExecutable`. For SimCity 4 Deluxe Edition, this would be something like `"SimCity 4"` or the game's actual executable name.

**Command-line arguments:**
- The launcher takes `NSProcessInfo.processInfo.arguments` (i.e., its own `argv`)
- If `argv` has 2+ entries, it strips `argv[0]` (its own path) and passes `argv[1...]` through as the game's arguments
- It then searches the remaining arguments for any that match the Steam bundle path — if found, the matching argument is used as an alternative executable path
- It builds a C `execv`-compatible `argv` array and calls **`execv()`** (at `0x100369a97`), which **replaces the launcher process entirely** with the game process

This means: **any command-line arguments passed to the launcher are forwarded to the game binary**, which is how Steam passes its own launch parameters.

There is also support for `GameGuide.SteamAltPlayExecutable` and `GameGuide.SteamAltPlayLaunchParams` — these allow specifying a different executable to launch with custom parameters. When the flag at instance offset `0x191` is set, the alt executable is inserted as `argv[1]` in the argument list.

### Mac App Store Build Path

When `isAppStoreBuild` is true:
- Reads `GameGuide.AppStorePlayExecutable` from Info.plist
- Creates an **`NSTask`** instead of using `execv`
- Sets the launch path to the app store executable
- Arguments are constructed as an `NSArray` with `arrayWithObjects:`
- Calls `[NSTask launch]` then `exit(0)`

### Script-Based Launch

If `GameGuide.launchUsingScript` is set in Info.plist, the launcher uses `/bin/sh` as the launch path and constructs a shell command with the game executable as an argument.

## Info.plist Keys the Launcher Reads

| Key | Purpose |
|-----|---------|
| `GameGuide.BundleID` | Game's bundle identifier |
| `GameGuide.SteamDefaultPlayExecutable` | Game executable name (Steam build) |
| `GameGuide.SteamAltPlayExecutable` | Alternative executable path |
| `GameGuide.SteamAltPlayLaunchParams` | Custom launch params for alt executable |
| `GameGuide.AppStorePlayExecutable` | Game executable name (MAS build) |
| `GameGuide.AppStoreBundleID` | MAS bundle ID |
| `GameGuide.SteamBundleID` | Steam bundle ID |
| `GameGuide.SteamAppID` | Steam application ID (`steam_appid.txt`) |
| `GameGuide.launchUsingScript` | If true, launch via `/bin/sh` |
| `GameGuide.PreferenceWindowedKey` | User default key for windowed mode |
| `GameGuide.FullScreenValue` | Fullscreen preference value |
| `GameGuide.WindowedValue` | Windowed preference value |
| `GameGuide.DisplayName` | Display name for the launcher title |

## Steam Integration

The `SteamManager` class handles:
- **`checkForSteam`** — detects if Steam is running
- **`initSteam`** — initializes the Steamworks API (`SteamUser021`, `STEAMAPPS_INTERFACE_VERSION008`)
- **`closeSteam`** — shuts down Steamworks
- **`GetSteamUserID`** — returns the Steam user ID
- **`hasDLCwithID:`** — checks DLC ownership
- **`GetDLCSteamID:`** — converts DLC index to Steam item ID

The `steam_appid.txt` file is read from `Contents/MacOS/steam_appid.txt` in the app bundle.

## Windowed / Fullscreen Preference

The launcher communicates the windowed/fullscreen choice to the game **via NSUserDefaults**, not via command-line arguments.

### The flow:

**1. User selects (clicking radio buttons):**
- `windowed:` sender calls:
  ```
  [[NSUserDefaults standardUserDefaults] setObject:<WindowedValue> forKey:<PreferenceWindowedKey>]
  ```
- `fullscreen:` sender calls:
  ```
  [[NSUserDefaults standardUserDefaults] setObject:<FullScreenValue> forKey:<PreferenceWindowedKey>]
  ```
  Both call `synchronize` immediately.

**2. The key and values come from Info.plist:**

| Info.plist Key | Purpose |
|---|---|
| `GameGuide.PreferenceWindowedKey` | The `NSUserDefaults` key name (e.g. `"windowed"` or `"FullScreen"`) |
| `GameGuide.FullScreenValue` | Value to store for fullscreen (defaults to `@(0)` i.e. `NO`) |
| `GameGuide.WindowedValue` | Value to store for windowed (defaults to `@(1)` i.e. `YES`) |

**3. Preferences survive `execv`** — Since `finishLaunch` calls `execv()` to replace the process with the game binary, and `NSUserDefaults` writes to `~/Library/Preferences/<bundle-id>.plist` on disk, the game binary reads the same key back after launch via `[[NSUserDefaults standardUserDefaults] objectForKey:<PreferenceWindowedKey>]`.

### Summary

The chain is: **launcher UI → NSUserDefaults → execv → game reads NSUserDefaults**. No command-line argument is involved. The Info.plist keys parameterize the exact key and values so each game can define its own convention.

> **Note on bundle ID:** The NSUserDefaults domain depends on the distribution channel. Steam uses `com.aspyr.simcity4.steam`, while the Mac App Store version likely uses `com.aspyr.simcity4` (untested). Substitute the correct domain when running `defaults` commands.

## DLC Management

- `MASContentCheck` class handles Mac App Store receipt validation and DLC purchases
- Steam DLC is checked via `SteamApps` API
- DLC purchase flow uses `SKPaymentQueue` (StoreKit) on the MAS side

## Launcher UI

- WebKit-based window (`WebView`)
- Loads remote content from `https://gameguide3.aspyr.com`
- Falls back to a local menu from `gg_fallbackMenu.plist` if offline
- Shows company logos, game logo, social media links (Twitter, Facebook)
- Preferences window for windowed/fullscreen toggle

## Key Technical Details

- **C++17 / Objective-C++** binary (uses `std::__1` libc++)
- **SDL 2.0.16** for window management and input
- **OpenGL** and **Metal** rendering backends
- **SQLite 3.36.0** embedded (likely used by the ASL framework for data storage)
- **nlohmann/json** for JSON handling
- **OpenSSL 1.1.1f** for HTTPS/certificate handling
- Compiled from Perforce paths: `/Volumes/Stuff/P4/WIP/ASL.AGen/` and `/Users/molloy/Perforce/planetcoaster/WIP/`

## macOS vs Windows — Preference System (Game Binary Analysis)

### How the Game Reads Settings

The game binary (`Sim City 4 Deluxe Edition_Child`) reads configuration settings using its own C++ preference reader functions at `0x1005c7bc9` (integer) and `0x1005c7d15` (boolean). These take a C string key name and a default value. **The reader functions internally use NSUserDefaults** (`registerDefaults:`, `objectForKey:`) to persist and read settings.

The preference keys found in the child binary are distinct from the launcher's keys:

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `intro-movie` | bool | true | Play intro video |
| `fullscreen-mode` | bool | true | Start in fullscreen |
| `custom-resolution` | bool | false | Enable custom resolution |
| `custom-width` | int | 1024 (0x400) | Custom width |
| `custom-height` | int | 768 (0x300) | Custom height |
| `display.index` | int | — | Monitor selection |
| `display.rect` | rect | — | Display rect |
| `display.remember` | bool | false | Remember display settings |

### Startup Relaunch Mechanism

At launch, the game binary reads these preferences and builds a **command-line string** identical to what the Windows version accepts. The logic (at `0x1000045a0`) works as follows:

```
# Read "fullscreen-mode" (default: true)
if fullscreen-mode == true → append "-f"
else → append "-w"

# Read "intro-movie" (default: true)
if intro-movie == false → append "-intro:off"

# Read "custom-resolution" (default: false)
if custom-resolution == true:
    custom-width = read("custom-width", 1024)
    custom-height = read("custom-height", 768)
    append "-r{custom-width}x{custom-height}x32"
    append "-CustomResolution:enabled"
```

After building the command line, the game **relaunches itself** (via `getpid()`/`kill()` + exec) with the new arguments, which means the second invocation processes the Windows-style command-line flags normally. This is the bridge mechanism: the macOS preference system generates the Windows command-line syntax and passes it to the native engine.

### Summary

The launch chain is:

1. **Launcher** writes windowed/fullscreen to NSUserDefaults
2. **Launcher** calls `execv()` → **Game binary (Child)** starts
3. **Game** reads its preferences (NSUserDefaults-backed)
4. **Game** constructs Windows-style CLI args and relaunches itself
5. **Second invocation** parses `-f`, `-w`, `-r1920x1080x32`, etc. natively

This means:
- `defaults write` can set game preferences directly (e.g., `defaults write com.aspyr.simcity4 fullscreen-mode 0`)
- Command-line args from Steam launch options are forwarded through by the launcher AND processed natively by the game on its second invocation
- The macOS port does NOT use INI files (`SimCity 4.ini`, `GZGraphic2.ini`, `SimCity 4.cfg` appear in the binary but are legacy references from the Windows engine code)
