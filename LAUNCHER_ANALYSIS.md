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

## Summary

The launcher is a thin wrapper: it shows a branded UI, validates DLC, then calls **`execv()`** to replace itself with the SimCity 4 game binary, forwarding all command-line arguments. The actual game executable name is configured in `Info.plist` under `GameGuide.SteamDefaultPlayExecutable` (Steam) or `GameGuide.AppStorePlayExecutable` (MAS). No game-specific command-line flags are hardcoded — the launcher simply passes through whatever `argv` it received.
