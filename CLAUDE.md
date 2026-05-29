# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains a single pre-compiled universal binary: **Sim City 4 Deluxe Edition Launcher** — the macOS launcher for Sim City 4 Deluxe Edition, ported by Aspyr Media.

## Binary Details

- **File:** `Sim City 4 Deluxe Edition_Launcher`
- **Type:** Mach-O universal binary (x86_64 + arm64)
- **Last modified:** November 19, 2024

## Technology Stack (from binary analysis)

- **Framework:** Aspyr Software Layer (ASL) — `ASL_AppBase`, `ASL_WritableDataStore`, `ASL_ReadOnlyDataStore`, `ASL_EpochTime`
- **Platform:** `Aspyr_AppMac` (macOS-specific Aspyr framework)
- **JSON:** nlohmann/json
- **Database:** SQLite 3.36.0
- **Testing:** Catch (C++ unit test framework)
- **Exception handling:** Boost
- **Graphics:** OpenGL with GLSL shaders (YUV color conversion, ABGR texture handling, etc.)

## Important Notes

- There is no source code in this repository — only the compiled binary.
- The binary was compiled with a C++ toolchain and links against the Aspyr internal SDK.
- This is a game launcher, not the game executable itself.
