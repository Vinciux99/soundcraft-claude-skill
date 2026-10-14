---
name: soundcraft
description: Record, edit, mix and bounce audio sessions with SoundCraft (Pro Tools-style DAW, ArtCraft suite) through MCP, the CLI or the control channel, and set up, connect or troubleshoot its MCP server on Windows and macOS.
---

# SoundCraft for Claude

SoundCraft (ArtCraft suite, storytold/soundcraft) is an open-source DAW in pure Rust, a clean-room reimplementation of **Avid Pro Tools**: Edit window, Mix window, Shuffle/Slip/Spot/Grid modes, Smart/Selector/Grabber/Trim tools, playlists, Memory Locations, AudioSuite, MIDI, automation, bounce. Status: **pre-alpha** (recording from multichannel interfaces, MIDI from controllers and AAF/OMF are missing).

**Everything you click is a command with an id + JSON parameters** (`mix.volume`, `edit.fades_create`...), with label, menu, shortcut, parameter doc and "enabled" state. Menus, keyboard, CLI, control channel and MCP execute the same ids. Programmatic calls **never open dialogs**; `session.inspect` / `inspect_session` shows the state so you can verify. Sessions are `.scraft` files, audio in `Audio Files/` next to them (plus `Backups/`). When in doubt `soundcraft-cli --help` / `soundcraft-cli describe <id>` is the source of truth.

## Quick check: is the MCP server connected?

1. Look at the tools you have. If tools from the MCP server `soundcraft` are available (for example `list_commands`, `execute` or `inspect_*` tools) the server is connected: skip to "Operating knowledge" below and use them directly.
2. If they are missing, check whether the user installed it under the name `soundcraft` or `soundcraft-headless`. In Claude Desktop the server may show as **disconnected**.
3. **Most common cause of "disconnected":** the live server (`mcp --connect {port}`) exits when SoundCraft is not running at the moment Claude starts. Fix: open SoundCraft with the **"SoundCraft (Claude)" launcher** (it starts the app with its control channel on), then re-enable the server in Claude's settings or fully quit and reopen Claude.
4. If the user does not need to watch the app live, **headless mode** always starts (no window, works on files). See "Live, headless or both".
5. If nothing is configured yet, follow "Set up the MCP server" below. Ask the user which OS they use if you do not know.

## Set up the MCP server (guide the user)

Walk the user through this step by step. Ask for the OS (Windows or macOS) first. If you have a shell on the user's machine you may run the steps yourself, but **ask before editing any config file and always back it up first**. Never print tokens or keys in chat.

### 1. Prerequisites
- Install SoundCraft from its upstream releases page: https://github.com/storytold/soundcraft/releases (Windows: the `.msi` installer; macOS: the `.dmg`, then drag the app to `/Applications`).
- Install Claude Desktop and/or Claude Code. For Claude Desktop, fully quit and reopen it after any config change.
- The MCP server is the **CLI** that ships with the app: `soundcraft-cli`. Windows installs it next to the app (`C:\Program Files\SoundCraft\soundcraft-cli.exe`, not on `PATH`). On macOS the CLI is a separate release download (asset `soundcraft-cli-<version>-macos-universal.zip`) unless it is bundled in the app (unverified): the setup script below fetches it into `~/.artcraft/bin/` and clears the macOS quarantine flag.
- This kit uses port **7801** (the upstream default), which does not clash with the other apps of the kit.

### 2. Option A: automatic setup (recommended)
The setup script finds the app and CLI, creates the launcher, backs up and updates Claude Desktop's config, registers the server in Claude Code (if the `claude` command exists) and installs this skill. Preview first with a dry run: nothing is changed.

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps soundcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps soundcraft
```
Or clone/download this repository and double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time: right-click, Open). Useful options: `-Mode live|headless|both` / `--mode ...` (default `live`; `both` registers `soundcraft` live and `soundcraft-headless`), `-DryRun` / `--dry-run`, `-NoLauncher`, `-NoClaudeDesktop`, `-NoClaudeCode`, `-NoSkill` (macOS: `--no-launcher`, `--no-claude-desktop`, `--no-claude-code`, `--no-skill`).

What it changes: Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json` plus the Microsoft Store/MSIX copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`; macOS `~/Library/Application Support/Claude/claude_desktop_config.json`; backup saved next to it as `<file>.backup-<timestamp>`), the launcher "SoundCraft (Claude)" (Windows: Desktop shortcut; macOS: `~/Applications/SoundCraft (Claude).app` plus a Desktop alias), Claude Code's user-scope MCP list, and `~/.claude/skills/soundcraft/`.

### 3. Option B: manual setup
**Windows** - edit `%APPDATA%\Claude\claude_desktop_config.json` (if Claude was installed from the Microsoft Store, also the copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`). Keep other servers; add this inside `mcpServers`. Backslashes must be doubled in JSON.
```json
{
  "mcpServers": {
    "soundcraft": {
      "command": "C:\\Program Files\\SoundCraft\\soundcraft-cli.exe",
      "args": [
        "mcp",
        "--connect",
        "7801"
      ]
    },
    "soundcraft-headless": {
      "command": "C:\\Program Files\\SoundCraft\\soundcraft-cli.exe",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**macOS** - edit `~/Library/Application Support/Claude/claude_desktop_config.json`. Use the absolute CLI path (replace `YOUR_NAME`; JSON does not expand `~`). The CLI path is unverified if your install differs: check with `ls ~/.artcraft/bin /Applications/SoundCraft.app/Contents/MacOS`.
```json
{
  "mcpServers": {
    "soundcraft": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/soundcraft-cli",
      "args": [
        "mcp",
        "--connect",
        "7801"
      ]
    },
    "soundcraft-headless": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/soundcraft-cli",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**Launcher** (starts the app with the control channel; needed before the live server can connect):
- Windows: create a Desktop shortcut named "SoundCraft (Claude)" with target `"C:\Program Files\SoundCraft\soundcraft.exe" --control 7801`
- macOS: run `open -n -a "/Applications/SoundCraft.app" --args --control 7801`, or save it as an app with Script Editor/`osacompile` (the setup script does this for you).

**Claude Code** (any OS; run once):
Windows (PowerShell):
```powershell
claude mcp add --scope user soundcraft -- "C:\Program Files\SoundCraft\soundcraft-cli.exe" mcp --connect 7801
claude mcp add --scope user soundcraft-headless -- "C:\Program Files\SoundCraft\soundcraft-cli.exe" mcp
```
macOS (Terminal):
```sh
claude mcp add --scope user soundcraft -- "$HOME/.artcraft/bin/soundcraft-cli" mcp --connect 7801
claude mcp add --scope user soundcraft-headless -- "$HOME/.artcraft/bin/soundcraft-cli" mcp
```
Check with `claude mcp list`.

### 4. Authentication
SoundCraft needs no token: the control channel listens on loopback (`127.0.0.1`) only and has no authentication. Anyone on the machine can drive the app while it is open, so close it when you are done.

### 5. Finish and verify
1. **Fully quit and reopen Claude** (Desktop: quit from the tray/menu bar, not just close the window).
2. Open SoundCraft with the **"SoundCraft (Claude)" launcher** before asking Claude to use it.
3. In Claude, confirm the server `soundcraft` is connected and call its `list_commands` tool (or equivalent). A non-empty command list means it works.

### Live, headless or both
- **Live** (`soundcraft`): drives the open app; the user sees every change and screenshots/UI tools work. Needs the launcher first.
- **Headless** (`soundcraft-headless`): in-process engine, no window; always starts; works on files; UI/screenshot tools are not available.
- **Both**: register both and pick per task. Switch with `-Mode` / `--mode`, or by editing the config entries above.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Server shows "disconnected" right after Claude starts | Live server exits when SoundCraft is not running | Open SoundCraft with the "SoundCraft (Claude)" launcher, then re-enable the server or restart Claude; or use headless mode |
| "Port already in use" / app opens but Claude cannot connect | Another app or an old SoundCraft instance holds port 7801 | Quit all SoundCraft windows, check the launcher uses the kit port, make sure no other ArtCraft app uses the same port |
| `command not found` / spawn error | CLI path wrong or the CLI is not installed (Windows: not on `PATH`; macOS: separate download) | Use the full absolute path in the config; macOS: run the setup script (it downloads the CLI to `~/.artcraft/bin/`) |
| macOS: "cannot be opened because the developer cannot be verified" | Gatekeeper/quarantine on the app or CLI | Right-click the app, Open; for the CLI: `xattr -dr com.apple.quarantine ~/.artcraft/bin/soundcraft-cli` |
| Anyone else can drive the app | The control channel has no authentication (loopback only) | Start the app with the control channel only when needed and quit it when done |
| Config edited but nothing changed (Windows) | Claude installed from the Microsoft Store reads the MSIX copy of the config | Edit/update `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\claude_desktop_config.json` too |
| JSON config error / server not listed | Unescaped backslashes or trailing comma | Double the backslashes on Windows; validate the JSON; restore the `.backup-*` file if needed |
| App crashes on start or window is blank | Graphics backend problem | Windows: set `WGPU_BACKEND=vulkan` (or `dx12`) for one launch; avoid `gl` on some AMD drivers |
| Changes do not appear in the window | You are on the headless server | Use the live server (launcher first) |
| Quotes disappear in PowerShell JSON arguments | Windows PowerShell 5.1 quoting for `.exe` | Use `--script`/files where the CLI supports them, or single-quote the whole JSON |

## Operating knowledge

## App, settings, environment
- **Settings folder:** Windows `%APPDATA%\SoundCraft\` (`ui.json`, `Autosave\`, `Presets\`, `logs\`); macOS location unverified (likely `~/Library/Application Support/SoundCraft`). Move it with `SOUNDCRAFT_CONFIG_DIR`.
- **Log:** `<settings>/logs/soundcraft.log` (previous: `soundcraft.1.log`, `soundcraft.2.log`; max 16 MiB). It also contains the line `control server listening on 127.0.0.1:<port>`. This matters because on Windows the app has no console and the `SOUNDCRAFT_CONTROL_PORT=...` print of `--control 0` is not visible, so **use a fixed port**.
- **App flags:** `soundcraft [--demo] [--control <port>] [--no-audio] [session.scraft | audio files...]`, plus `--version`. `--demo` opens the demo song (synthesized by SoundCraft); `--no-audio` disables audio output (not the recording input).
- Shortcuts: on Windows the docs' `Cmd` = `Ctrl`. Setup > Keyboard Shortcuts lists them all.

| Variable | Effect |
|---|---|
| `SOUNDCRAFT_CONTROL_PORT` | like `--control <port>` for the app; also the default port of `soundcraft-cli app` |
| `SOUNDCRAFT_CONFIG_DIR` | full settings folder (isolated test profile); empty = no folder |
| `SOUNDCRAFT_NO_PREFS` | no preferences read/written, no log file |
| `RUST_LOG` | log levels (e.g. `debug`, `warn,soundcraft_mix=trace`) |
| `WGPU_BACKEND` | graphics backend (`dx12` default on Windows, `vulkan`); avoid `gl` (AMD driver crash) |
| `WGPU_DX12_COMPILER` | `dxc`/`auto` for troubleshooting only (default FXC) |

- **If the app will not open** (flashes in Task Manager and closes), in PowerShell from the exe folder: `$env:WGPU_BACKEND = "vulkan"; & .\soundcraft.exe`, then `Remove-Item Env:WGPU_BACKEND`.

## Connecting

| Mode | Start | What it drives |
|---|---|---|
| MCP headless | `soundcraft-cli mcp` (empty session) or `soundcraft-cli mcp --demo` | in-process engine, no window or audio device |
| MCP headless confined | `soundcraft-cli mcp --automation-read-root DIR --automation-write-root DIR` | as above, files only inside the roots |
| MCP on the live app | `soundcraft-cli mcp --connect 7801` (port or `host:port`) | the app opened with `--control 7801`: same tools + screenshot/click |
| CLI | `soundcraft-cli run ...` / `script ...` (headless), `soundcraft-cli app --port 7801 ...` (live app) | one execution per call |
| Control channel | `soundcraft --control 7801` | line JSON over TCP `127.0.0.1:7801` |

- **Port:** the kit uses **7801**, which does not collide with the other ArtCraft apps (9876, 9877, 7878, 7979-7990). **Warning:** `soundcraft-cli app` without `--port` and without `SOUNDCRAFT_CONTROL_PORT` uses **7979** (VectorCraft's port in the kit), so **always** pass `--port 7801`.
- **Authentication:** **none** (no token or token file). The channel listens on loopback only, max 16 connections, but anyone on the machine can command the app. Start `--control` only when needed and close the app (or `app.quit`) when done. Do not set `SOUNDCRAFT_CONTROL_PORT` system-wide: the app would open the port on every launch.
- **Automation roots** (`--automation-read-root` / `--automation-write-root`, same flags as the other Crafts): every file read (`open_session`, `import_audio`, `file.import_midi`, `file.import_video`, `track.load_preset`, `mix.sampler_load`...) must be inside the read root; every file written (`save_session`, `bounce_mix`, `file.bounce_stems`, `file.export_*`, `track.save_preset`, plus the session's `Audio Files/` and `Backups/`) inside the write root. Omitted root = no access of that kind. Folders must exist. Case-insensitive on Windows, `CON`/`NUL`/`COM1` rejected. A score command without `path` writes to temp and is rejected. **They cannot be combined with `--connect`** (error): the desktop app cannot be confined.

### Choose the channel
1. **SoundCraft MCP server available** -> use it (tools below). Live when the user is watching the app or you need to listen/record/screenshot; headless for file work.
2. **Shell on the user's machine** -> `soundcraft-cli` with full path; live app with `app --port 7801 ...`.
3. **Cloud/Linux sandbox** -> cannot run the user's app or reach their `localhost` ports (not 7801). There you can only **prepare files** (scripts for `soundcraft-cli script`, MIDI, file lists).
4. **Last resort: computer use on the GUI.** Non-US keyboards: shortcuts with `[` `]` (e.g. `Alt+[` Play Edit), `=` (`Ctrl+=` Mix/Edit) and `` ` `` may not arrive correctly; prefer menus. F1-F4 edit modes, F5-F10 tools, Space play/stop.

## Basic rules
- **Positions** (`at`, `start`, `end`, `to`, `from`):
  - integer = **samples** (`96000`). A decimal number is also treated as samples, **not** seconds;
  - `{"seconds": 2.5}`;
  - string with a timebase prefix: `"bars_beats:5|1|000"`, `"min_secs:0:12.500"`, `"timecode:00:00:10:00"`, `"samples:48000"` (also `feet_frames`). Without a prefix the string is read in the main counter's format, then the others.
  - CLI exception: in `soundcraft-cli run --start 12.5 --end 20` numbers are **seconds**.
- **Range:** `start`+`end` or `start`+`length`; omitted = current edit selection.
- **Durations in samples:** e.g. fade `length` and `by` of `edit.move_clips`/`edit.shift` (`edit.shift` also accepts `{seconds}`). At 48 kHz, 0.1 s = 4800 samples.
- **MIDI:** notes in ticks (`start_ticks`, `length_ticks`, default 480); tracks `timebase` `samples|ticks`.
- **Sample rate:** `session.new` defaults to 48000 Hz / 24 bit. For audio at a different rate check `session.audio_health` (sample-rate mismatch, out-of-range sources, missing media).
- **References:** tracks by **name** (`"Kick"`) or numeric id; clips by **id** (from `inspect_session`); several tracks with `tracks: [...]`. Insert/send slots `0..9` or `a..j` (`"c"` = 2). `track.new` returns `{"tracks": [id, ...]}`.
- **Empty params = current selection.** To not depend on what is selected always pass explicit `track`/`tracks`, `clips`, `start`/`end`.
- **Mostly tolerant parameters:** a missing or wrong-typed number often falls back to the default without error. Strict: nonexistent track/clip names, `mix.instrument` (unknown fields = error), `mix.insert_params` (unknown ids or non-numeric values = error), malformed JSON in the CLI. So **always verify afterwards** with `inspect_session`.
- **Undo:** every command is undoable with `edit.undo` (`edit.redo`), except queries, view settings and "noundo" commands (session/saves, bounce, selection, transport, mode/tool, marker recall). `engine.history` shows the history.
- **Batch:** no `$N` references to earlier results: if you need a freshly created id, make separate calls or use names (create tracks with `name`). `batch` stops at the first error (unless `keep_going`) and runs at most 1000 calls.
- **Bounce / long jobs:** `bounce_mix` is synchronous, with no progress or cancel; it returns `path`, `bytes`, `seconds`, `peak_db`, `lufs`, `true_peak_db`. On the live app the channel answers `{"ok":false,"error":"timeout"}` after 90 s (the CLI/MCP client waits up to 120 s): long bounces -> headless, or in pieces. **Without `start`/`end` the bounce exports the selection if there is one**, otherwise from 0 to the end of the content.
- **Formats:** import WAV/BWF/RF64, AIFF/AIFC, FLAC, MP3, OGG, AAC/M4A, ALAC, CAF, audio of MP4/MOV; export **only WAV, AIFF, FLAC** (16/24/32f). **No MP3 export**: needs an external tool.
- **Headless:** no audio device; transport commands are discarded (playhead stays); `ui.*` methods error. Recording only in the desktop app.
- **Saving:** `save_session` does not overwrite another existing file without `replace: true`; **`soundcraft-cli run/script --save` instead saves with `replace: true`**. `open_session` returns `missing` (media not found).

## MCP tools

17 tools (the last 7 only with `--connect`):

| Tool | Arguments | Notes |
|---|---|---|
| `list_commands` | `filter?` (substring of id or label) | id, label, menu, shortcut, parameter doc, enabled + reason |
| `execute` | `command`, `params?` | runs a command |
| `batch` | `calls: [{command, params}]`, `keep_going?` | any error -> `isError: true`; with `keep_going` lists all results |
| `inspect_session` | `detail?: summary\|full` | tracks, clips, mixer, inserts (+params with `full`), sends, automation, markers, selection, transport; `sources[].loaded` |
| `new_session` | `name?`, `sample_rate?`, `template?: blank\|demo` | = `session.new` |
| `open_session` | `path` | `.scraft`; returns `missing` |
| `save_session` | `path?` (required the first time), `replace?` | = `session.save` |
| `import_audio` | `path`, `track?`, `at?` | = `file.import_audio` (which also accepts `paths`, `new_tracks`) |
| `bounce_mix` | `path`, `format?: wav\|aiff\|flac`, `bit_depth?`, `start?`, `end?`, `normalize?` | = `file.bounce_mix`; peak dBFS and integrated LUFS |
| `parity` | none | Pro Tools parity report |
| `screenshot` | `path?` | window PNG (inline image without `path`) |
| `ui_inspect` | none | active window, panels, dialogs, timeline and track rects |
| `ui_set` | `state` | e.g. `{"window": "Mix"}` |
| `menu_invoke` | `path` | like clicking the menu (may open the dialog), e.g. `"Track > New..."` |
| `click` | `x`, `y`, `button?`, `count?` | window-point coordinates |
| `drag` | `x`, `y`, `to_x`, `to_y` | primary button |
| `key` | `key`, `cmd?`, `shift?`, `alt?`, `ctrl?` | `Space`, `Enter`, `A`, `F7`, `=`... |

Resources: `soundcraft://session` (full session JSON), `soundcraft://commands`. No prompts.

## Recipes (MCP; via CLI/control the same ids and JSON)

```jsonc
// New session and first save in a working folder
new_session  {"name": "Podcast 01", "sample_rate": 48000}
save_session {"path": "/path/to/work/Podcast 01/Podcast 01.scraft"}

// Import audio (new track by default) and read the clip ids
import_audio {"path": "/path/to/work/voice.wav", "at": {"seconds": 0}}
inspect_session {"detail": "summary"}

// Move / cut / trim clips
execute {"command": "edit.move_clips", "params": {"clips": [7], "to": {"seconds": 4.0}}}
execute {"command": "edit.separate",   "params": {"tracks": ["voice"], "at": {"seconds": 10}}}
execute {"command": "edit.trim_clip",  "params": {"clip": 7, "start": {"seconds": 4.5}}}
execute {"command": "edit.select", "params": {"tracks": ["Snare"], "start": "bars_beats:6|1|000", "end": "bars_beats:9|1|000"}}
execute {"command": "edit.cut", "params": {}}          // then edit.undo if needed

// Fades: per clip (length in samples) or over a range
execute {"command": "edit.fades_create", "params": {"clip": 7, "side": "in", "length": 4800, "shape": "s-curve"}}
execute {"command": "edit.fades_create", "params": {"tracks": ["voice"], "start": {"seconds": 59}, "end": {"seconds": 61}, "shape": "equal power"}}

// Clip gain and AudioSuite
execute {"command": "clip.gain", "params": {"clips": [7], "db": -3}}
execute {"command": "audiosuite.process", "params": {"process": "normalize", "clips": [7]}}

// Mixer: volume, pan, plugins (ids from `soundcraft-cli plugins` / engine.plugins)
execute {"command": "mix.volume", "params": {"track": "Kick", "db": -8.5}}
execute {"command": "mix.pan",    "params": {"track": "Hats", "pan": -0.5}}
execute {"command": "mix.insert", "params": {"track": "Bass", "slot": "c", "plugin": "eq_7band"}}
execute {"command": "mix.insert_bypass", "params": {"track": "Bass", "slot": 2}}

// Reverb/delay aux fed by a send
batch {"calls": [
  {"command": "track.new",   "params": {"kind": "aux", "format": "stereo", "name": "Delay"}},
  {"command": "track.input", "params": {"track": "Delay", "input": "DelayBus"}},
  {"command": "mix.insert",  "params": {"track": "Delay", "plugin": "mod_delay"}},
  {"command": "mix.send",    "params": {"track": "Lead", "slot": 1, "bus": "DelayBus", "level_db": -6}}]}

// Automation, markers, tempo
execute {"command": "automation.write_range", "params": {"track": "Pad", "param": "volume", "start": {"seconds": 10}, "end": {"seconds": 12}, "value": -20.0}}
execute {"command": "markers.add", "params": {"name": "Bridge", "at": {"seconds": 30.0}}}
execute {"command": "event.tempo", "params": {"bpm": 110.0}}

// MIDI: instrument track, clip, note
execute {"command": "track.new", "params": {"kind": "instrument", "name": "Synth", "instrument": "subtractive_synth"}}
execute {"command": "midi.new_clip", "params": {"track": "Synth", "start": "bars_beats:1|1|000"}}
execute {"command": "midi.note_add", "params": {"clip": 12, "pitch": 60, "start_ticks": 0, "length_ticks": 480, "velocity": 100}}

// Export: mix (explicit range!) and stems
bounce_mix {"path": "/path/to/out/mix.wav", "bit_depth": "24", "start": {"seconds": 0}, "end": {"seconds": 180}}
execute {"command": "file.bounce_stems", "params": {"dir": "/path/to/out/stems", "format": "flac"}}
```

Plugin parameters: `mix.insert` accepts `params: {id: value}`; see the exact ids in `inspect_session {"detail":"full"}` (inserts with parameters) after inserting, then `mix.insert_param {track, slot, param, value}` or `mix.insert_params {track, slot, params}`.

## CLI (`soundcraft-cli`)

```text
soundcraft-cli info FILE                     summary of an audio file or .scraft
soundcraft-cli convert IN OUT [--bit-depth 16|24|32f] [--rate HZ] [--normalize]
soundcraft-cli run [--in S.scraft | --demo] [--cmd ID[=JSON]]... [--bounce OUT] [--start T --end T] [--save S.scraft] [--inspect]
soundcraft-cli script FILE [--in S | --demo] [--save S] [--bounce OUT] [--keep-going]   (one `ID JSON` per line)
soundcraft-cli commands [FILTER]             list commands
soundcraft-cli describe ID                   details of one command (JSON)
soundcraft-cli app [--port P] METHOD [JSON]  call the app's control channel
soundcraft-cli mcp [--connect PORT] [--demo] [--automation-read-root DIR] [--automation-write-root DIR]
soundcraft-cli parity [--write PATH]         parity report
soundcraft-cli plugins                       built-in plugins (id, name, category, instrument)
```

- `run` executes `--cmd` in order, stops at the first error (exit 1), then `--bounce` (= `file.bounce_mix`), `--save` (overwrites!), `--inspect`. Without `--in`/`--demo` it starts empty.
- `script`: lines `id {json}`, `#` comments, blank lines ignored. **On PowerShell prefer it over `--cmd`**: JSON quotes passed to an `.exe` can get altered.
- `app`: a dotted id that is not a control method becomes `engine.execute` (`app --port 7801 transport.play`). `ui.scroll`/`ui.zoom` are not in the CLI's method list: send them as a raw JSON line on the socket.
- `convert`: `--rate` must be an integer > 0, `--bit-depth` 16|24|32f (default 24), `--normalize` to -0.1 dBFS; output format from the extension (wav/aiff/flac).

```powershell
$cli = "C:\Program Files\SoundCraft\soundcraft-cli.exe"
& $cli describe edit.fades_create
& $cli run --in ".\song\song.scraft" --cmd 'mix.volume={"track":"Kick","db":-6}' --bounce ".\out\mix.wav" --start 0 --end 30
& $cli script ".\job.txt" --in ".\song\song.scraft" --bounce ".\out\mix.wav"
& $cli app --port 7801 session.inspect '{"detail":"summary"}'
& $cli app --port 7801 ui.screenshot '{"path":"C:\\Temp\\sc.png"}'
& $cli run --demo --cmd 'session.audio_health={}'
```

## Control channel

- **Transport:** TCP on `127.0.0.1:<port>` only (kit: 7801), newline-delimited JSON, one request and one response per line. Max 16 connections; lines up to ~86 MiB (sized for base64 plugin states); non-UTF-8 bytes close the connection. No authentication.
- **Format:** `{"id":1,"method":"engine.execute","params":{"command":"track.new","params":{"count":2,"format":"Stereo"}}}` -> `{"id":1,"ok":true,"result":{"tracks":[12,13]}}`; error: `{"id":1,"ok":false,"error":"bad parameters for `mix.volume`: no track named `Nope`"}`.
- **Timeout:** if the app does not answer within 90 s -> `{"ok":false,"error":"timeout"}`.

| Method | Parameters | Effect |
|---|---|---|
| `engine.execute` (alias `command`) | `{command, params}` | runs a command, never dialogs |
| `engine.commands` | `{filter?}` | all commands, including UI ones (`window.*`) |
| `engine.parity` | `{}` | parity report |
| `session.inspect` (alias `document.inspect`) | `{detail?: summary\|full}` | the document: tracks, timed clips, inserts, sends, routing, automation, markers, groups, buses, sources, selection, zoom, transport, undo labels |
| `ui.inspect` | `{}` | window/panels/dialogs, sizes, play/position, audio device, `edit_layout` (timeline and track-row rects) |
| `ui.set` | `UiState` fields | e.g. `{"window":"Mix","narrow_mix":true}` |
| `ui.menu.list` / `ui.menu.invoke` | `{}` / `{path}` or `{id}` | catalog menus; invoke may open the dialog |
| `ui.click` | `{x,y,button?,count?,cmd?,shift?,alt?,ctrl?}` | synthetic click (window points) |
| `ui.drag` | `{x,y,to_x,to_y,steps?}` | drag |
| `ui.scroll` | `{x,y,dx?,dy?,unit?: point\|line,steps?,...}` | trackpad swipe; with `cmd` zoom, with `shift` horizontal |
| `ui.zoom` | `{x,y,factor,steps?}` | pinch; `factor` > 1 = zoom in |
| `ui.move` / `ui.key` / `ui.text` | `{x,y}` / `{key,...}` / `{text}` | pointer, key, text into the focused field |
| `ui.screenshot` | `{path?}` | PNG (base64 in `png_base64` without path) |
| `app.quit` | `{}` | closes the app |

- **No UI element ids:** target by coordinates taken from `ui.inspect` (`edit_layout`). Docking panels: `tracks`, `clips`, `midi`, `memory`, `history` (protected center `canvas`), via `engine.execute` with `ui.dock` (`operation: open|close|activate|float|move`, `panel`, ...) and `ui.dock.reset`. Theme: `ui.theme {"mode":"system"}` (default Dark). Audio device: `ui.audio_device`.
- Synthetic input injects egui events, not system input. Automated UI tests: never arm tracks, enable input monitor or record.

PowerShell without the CLI (raw line):

```powershell
$c = New-Object System.Net.Sockets.TcpClient("127.0.0.1", 7801); $s = $c.GetStream()
$w = New-Object System.IO.StreamWriter($s); $r = New-Object System.IO.StreamReader($s); $w.AutoFlush = $true
$w.WriteLine('{"id":1,"method":"session.inspect","params":{"detail":"summary"}}'); $r.ReadLine(); $c.Close()
```

## Commands by family

About 380 engine commands (count from source; tests guarantee > 150) plus UI ones. Full list and parameters: `list_commands {"filter": "fade"}`, `soundcraft-cli commands fade`, `soundcraft-cli describe <id>`. A selection of real ids:

- **session.** (11): `new`, `open`, `close`, `save`, `save_as`, `save_copy`, `save_template`, `revert`, `audio_health`, `inspect`, `format_time` (`{at, format?}`).
- **file.** (17): `import_audio`, `import_midi`, `import_video`, `import_session_data`, `import_clip_groups`, `bounce_mix`, `bounce_stems` (`{dir,...}`), `export_clips`, `export_midi`, `export_session_text`, `export_selected_tracks_as_session`, `export_sibelius`, `send_to_sibelius`, `print_score`, `score_setup`, `export_clip_groups`, `get_info`.
- **track.** (65): `new` (`{count?, format?: mono|stereo|5.1..., kind?: audio|aux|master|midi|instrument|vca|folder, name?, timebase?, instrument?}`), `delete`, `rename`, `duplicate`, `input`, `output`, `inspect`, `color`, `hide`, `height`, `view`, `group`, `ungroup`, `freeze`, `commit`, `bounce`, `make_inactive`, `split_into_mono`, `playlist_new`, `playlist_duplicate`, `playlist_select`, `vca`, `save_preset`, `load_preset`, `bypass_inserts`, `mute_sends`, `elastic`, `timebase`, `extract_midi`...
- **edit.** (93): `undo`, `redo`, `select`, `select_all`, `select_none`, `cut`, `copy`, `paste`, `clear`, `duplicate`, `repeat`, `shift`, `insert_silence`, `separate`, `separate_on_grid`, `separate_at_transients`, `heal`, `consolidate`, `strip_silence`, `trim_to_selection`, `trim_clip`, `move_clips`, `nudge`, `fades_create`, `fades_delete`, `fade_to_start`, `fade_to_end`, `mute_clips`, `mode` (shuffle|slip|spot|grid|relative_grid), `tool`, `snap_to_grid`, `copy_to_new_playlist`, `cut_all_automation`, `paste_clip_gain`...
- **clip.** (31): `gain`, `gain_nudge_up`/`down`, `gain_render`, `gain_bypass`, `rename`, `color`, `inspect`, `loop`, `unloop`, `group`, `ungroup`, `place_source`, `capture`, `clear`, `effects_set`, `effects_bypass`, `effects_render`, `quantize_to_grid`, `elastic_properties`, `conform_to_tempo`, `edit_lock`, `time_lock`...
- **mix.** (28): `volume` (`db` or `delta_db`), `pan`, `mute`, `solo`, `solo_safe`, `clear_solos`, `clear_mutes`, `trim`, `phase_invert`, `automation_mode` (off|read|touch|latch|write|trim), `insert`, `insert_param`, `insert_params`, `insert_remove`, `insert_bypass`, `insert_move`, `send`, `send_level`, `send_remove`, `new_bus`, `instrument`, `instrument_param`, `sampler_load`, `sampler_clear`, `record_arm`, `input_monitor`, `surround_pan`.
- **automation.** (10): `set_point` (`param: volume|pan|mute|send_a_level|plugin:a:<id>`), `write_range`, `clear`, `thin`, `thin_all`, `write_to_current`, `write_to_all`, `volume_to_clip_gain`, `clip_gain_to_volume`, `copy_to_send`.
- **event.** (35): `tempo`, `meter`, `tempo_remove`, `tempo_linear`/`parabolic`/`s_curve`, `insert_time`, `cut_time`, `quantize`, `transpose`, `change_velocity`, `beat_detective`, `identify_beat`, `add_key_change`, `extract_chords`...
- **midi.** (6): `notes`, `new_clip`, `note_add`, `note_edit`, `note_delete`, `notes_move`.
- **markers.** (6): `add`, `delete`, `edit`, `recall`, `next`, `previous`.
- **transport.** (13): `play`, `stop`, `toggle`, `record`, `pause`, `play_selection`, `locate`, `rtz`, `go_to_end`, `status`... (live app only).
- **view.** (29): `zoom_fit`, `zoom_set`, `zoom_to_selection`, `main_counter`, `grid`, `ruler`...
- **options.** (8), **setup.** (20: `session`, `main_format`, `playback_engine`, `io`...), **audiosuite.process** (`process`: plugin id | normalize | reverse | invert | gain | duplicate | time_stretch | pitch_shift | varispeed | signal_generator), **engine.** (`commands`, `history`, `plugins`, `clap_plugins`, `vst3_plugins`, `parity`), **video.** (3), **app.quit**.
- Built-in plugins named in docs/tests: `eq_7band`, `mod_delay`, `gain`, `subtractive_synth`, `drum_synth`; external ones with prefix `clap:`, `vst3:`, `au:`. The rest: `soundcraft-cli plugins`.

## Recommended workflow

1. **Orient:** `inspect_session {"detail":"summary"}` (and `ui_inspect`/`screenshot` on the live app); `session.audio_health` for missing media or different sample rates.
2. **Work on a copy:** open the original and immediately save with `session.save_copy` / `save_as` into a new folder (or use read/write roots), then work on the copy. Never `replace: true` or `run --save` on the original.
3. **Find the commands:** `list_commands {"filter": "..."}` + `soundcraft-cli describe <id>`; do not guess parameters.
4. **Edit in small steps** with explicit references (tracks by name, clips by id, `start`/`end` ranges), one command or a small `batch` at a time; `edit.undo` if the outcome is not what you expected.
5. **Verify:** `inspect_session` after each step (volume_db, inserts, sends, clips, automation, markers); for audio a short test `bounce_mix` into a temp folder, checking `peak_db`, `true_peak_db`, `lufs`; on the live app `screenshot`.
6. **Save/export only with the user's OK**, to new paths; WAV/AIFF/FLAC export with explicit `start`/`end`.
7. **Close:** the control channel has no authentication, so when finished quit the app (`app.quit`, after saving if the user wants) or at least stop using the "(Claude)" launcher and reopen SoundCraft normally.
