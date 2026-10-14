# SoundCraft skill for Claude

SoundCraft (ArtCraft suite, storytold/soundcraft) is an open-source DAW in pure Rust, a clean-room reimplementation of **Avid Pro Tools**: Edit window, Mix window, Shuffle/Slip/Spot/Grid modes, Smart/Selector/Grabber/Trim tools, playlists, Memory Locations, AudioSuite, MIDI, automation, bounce. Status: **pre-alpha** (recording from multichannel interfaces, MIDI from controllers and AAF/OMF are missing).

This repository packages a **skill** that teaches Claude to drive SoundCraft (part of the [ArtCraft suite](https://github.com/storytold/soundcraft) by storytold) and that **knows how to create the MCP server and guide you through it**, on **Windows and macOS**. It is unofficial and not affiliated with storytold.

> Part of the [ArtCraft Claude skills hub](https://github.com/Vinciux99/artcraft-claude-skills): all 13 skills, a picker and a one-command installer.

## What it does
- Operating manual for SoundCraft: connection modes, ground rules, every MCP tool, recipes, CLI, control channel, command families and a safe workflow (work on copies, verify, save only with your OK).
- Built-in MCP setup assistant: Claude checks whether the server is connected, then walks you through installing it (automatic script or manual config), the launcher, authentication and verification.
- Troubleshooting table for the usual problems (server "disconnected", ports, macOS quarantine, Windows Store config location...).

## What you can ask Claude
- "Create a new 48 kHz session, import `voice.wav`, trim the first 4.5 seconds and add a 100 ms fade in."
- "Normalize the vocal clip, add a 7-band EQ insert on the bass and set the kick to -8.5 dB."
- "Build a delay aux, route a send from the lead track and set it to -6 dB."
- "Add markers at 0:30 and 1:10 and set the tempo to 110 BPM."
- "Create a MIDI instrument track with a synth and write a C4 quarter note."
- "Bounce a 24-bit WAV of the first three minutes and report peak, true peak and LUFS."
- "Export the stems as FLAC."

## Highlights
- Headless, headless-confined (read/write roots) and live MCP, 17 tools plus resources.
- Position/unit rules (samples vs seconds vs bars|beats), undo, batch and bounce-timeout behavior spelled out.
- Recipes for editing, fades, mixing, sends/aux, automation, MIDI, bounce and stems.
- CLI `run`/`script`/`convert` pipelines and the full control-channel protocol.
- Built-in guidance for creating and troubleshooting the MCP server on Windows and macOS.

## App-specific notes
- Kit port **7801**; live MCP `mcp --connect 7801`; headless `mcp` (or confined with `--automation-read-root` / `--automation-write-root`). Always pass `--port 7801` to `soundcraft-cli app` (its default is 7979).
- No authentication, loopback only. Roots cannot be combined with `--connect`.
- Export is WAV/AIFF/FLAC only (no MP3). Pre-alpha: recording from multichannel interfaces and AAF/OMF are missing.
- `soundcraft-cli run/script --save` overwrites; work on copies.

## Get the skill

| Where | How |
|---|---|
| **Claude app** (claude.ai / Desktop) | Download [`soundcraft-skill.zip](https://github.com/Vinciux99/soundcraft-claude-skill/releases/latest/download/soundcraft-skill.zip)`, then Customize > Skills > "+" > Create skill > Upload a skill. Needs Settings > Capabilities > "Code execution and file creation" on. |
| **Claude Code, plugin** | `/plugin marketplace add Vinciux99/artcraft-claude-skills` then `/plugin install soundcraft@artcraft` |
| **Claude Code, manual** | copy `skills/soundcraft/` to `~/.claude/skills/soundcraft/` (the setup script below does it) |

## Create the MCP server

Prerequisite: install SoundCraft from [upstream releases](https://github.com/storytold/soundcraft/releases) (Windows `.msi`, macOS `.dmg`) and Claude Desktop and/or Claude Code.

**One command (recommended)**

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps soundcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps soundcraft
```
Add `-DryRun` (Windows) or `--dry-run` (macOS) to preview, and `-Mode live|headless|both` / `--mode ...` to choose the mode.

**Or from a clone:** double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time right-click > Open).

**Or just ask Claude:** once the skill is installed, say "set up the SoundCraft MCP server" and Claude will guide you step by step.

After setup: fully quit and reopen Claude, open SoundCraft with the **"SoundCraft (Claude)"** launcher, then ask Claude to use it. If the server shows "disconnected", the app was not running when Claude started: open it with the launcher and re-enable the server, or use headless mode.

What the setup changes (all backed up or reversible): Claude Desktop config (`mcpServers.soundcraft`, backup `<file>.backup-<timestamp>`), the launcher, Claude Code's user-scope MCP list and `~/.claude/skills/soundcraft/`.

## Repository layout
```
skills/soundcraft/SKILL.md      the skill
app.json                   machine-readable settings (ports, paths, MCP args) used by the scripts
scripts/                   artcraft-setup.ps1 (Windows), artcraft-setup.sh (macOS), json-tool.js
setup-windows.cmd          double-click wrapper
setup-macos.command        double-click wrapper
.claude-plugin/plugin.json Claude Code plugin manifest
.github/workflows/ci.yml   validation, setup tests on Windows and macOS, release zip
```

## CI and releases
Every push validates `app.json`, `SKILL.md` and the scripts, then runs the setup scripts against a fake install on **Windows** and **macOS** runners. Every push to `main` builds `soundcraft-skill.zip` and publishes (or updates) the GitHub release `v1.0.0`.

## Status and caveats
- Windows paths and arguments come from a working install. The macOS paths (`/Applications/SoundCraft.app`, CLI location and release asset name) follow the upstream packaging but have **not** been verified on a Mac; the skill marks unverified items and the setup script reports what it could not find.
- SoundCraft is a young open-source project; behaviour can change between releases. `list_commands` / `soundcraft-cli commands` is always the source of truth.
- Unofficial community packaging, MIT licensed (see `LICENSE`).
