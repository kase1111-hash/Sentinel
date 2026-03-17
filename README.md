# Sentinel

**Know what's running on your machine.**

Sentinel is an open-source kernel activity monitor for Windows that logs exactly what ring-0 drivers are doing on your hardware — what memory they read, what processes they inspect, what data they send home, and when. Built for gamers, researchers, and anyone who believes that owning a computer means knowing what it does.

---

## The Problem

Modern anti-cheat software operates at the kernel level. Vanguard, EasyAntiCheat, Ricochet, BattlEye, and others install drivers that load before your desktop, run with the highest system privileges, and have unrestricted access to your entire machine. They can read any process memory, scan any file, log any keystroke, and transmit data to remote servers — all while providing zero visibility into what they're actually doing.

You are told this is necessary to prevent cheating. You are not told what else it does.

There is currently no tool that lets an ordinary user answer basic questions like:

- What processes did Vanguard inspect in the last hour?
- How much data did EasyAntiCheat send to its servers today?
- Did BattlEye access any files outside the game directory?
- Is the anti-cheat still running after I closed the game?
- What registry keys did it read or modify?

Sentinel answers these questions.

---

## What Sentinel Does

Sentinel is a **passive observer**. It does not block, modify, bypass, or interfere with anti-cheat software in any way. It watches and logs. That's it.

### Kernel Activity Logging

Sentinel uses Event Tracing for Windows (ETW) and, where deeper visibility is needed, a minimal signed filter driver to capture:

- **Process inspection events** — which processes a kernel driver reads memory from, and when
- **File access events** — what files a driver opens, reads, or writes outside its own directory
- **Registry access events** — what registry keys a driver queries or modifies
- **Network activity** — outbound connections, destination IPs, data volume, timing
- **Driver lifecycle** — when a kernel driver loads, runs, and unloads
- **Persistence behavior** — whether a driver remains active after its associated application closes

All events are timestamped and attributed to the specific driver that generated them.

### Audit Reports

Raw logs are structured and queryable. Sentinel generates human-readable audit reports per driver:

```
── Vanguard (vgk.sys) ── Session: 2026-03-16 14:22:00 → 18:47:33 ──

  Loaded at:        14:22:00 (system boot — runs before desktop)
  Unloaded at:      STILL RUNNING (game closed at 18:47:33)

  Processes scanned: 847
    Top targets:     chrome.exe (312), discord.exe (198), explorer.exe (94)
    Game process:    VALORANT.exe (41)

  Files accessed:    23 outside game directory
    Notable:         %APPDATA%\Discord\Local Storage (read)
                     %USERPROFILE%\Documents\*.pdf (enumerated)

  Network activity:  142 connections to 4 unique IPs
    Data sent:       2.4 MB    Data received: 890 KB
    Destinations:    Riot Games CDN (2), AWS us-east-1 (1), unknown (1)

  Registry reads:    67
    Notable:         HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall (full enum)
```

### Real-Time Dashboard

A lightweight Tauri-based UI shows live kernel activity as it happens:

- Active kernel drivers and their current state
- Real-time event stream (filterable by driver, event type, target)
- Network traffic graph per driver
- Process access heatmap — which of your running applications are being inspected most frequently
- Alert rules — get notified when a driver accesses something unexpected

---

## What Sentinel Does Not Do

**Sentinel is not a cheat tool.** It does not:

- Modify, patch, or tamper with anti-cheat drivers
- Hide processes, files, or memory regions from anti-cheat
- Inject code into games or anti-cheat processes
- Bypass, circumvent, or interfere with anti-cheat detection
- Provide any competitive advantage in any game

Sentinel is a system transparency tool. It treats anti-cheat drivers the same way it treats any other kernel-mode software: as something the machine owner has the right to observe.

---

## Architecture

```
sentinel/
├── driver/                 # Minimal kernel-mode filter driver (signed)
│   ├── sentinel-filter/    # ETW provider + minifilter for file/registry/process callbacks
│   └── certs/              # Driver signing infrastructure
├── service/                # User-mode collection service
│   ├── src/
│   │   ├── main.rs         # Service entry point
│   │   ├── etw_consumer.rs # ETW real-time event consumer
│   │   ├── log_store.rs    # SQLite event storage + rotation
│   │   ├── analyzer.rs     # Event correlation + report generation
│   │   └── alert.rs        # Rule-based alerting engine
│   └── Cargo.toml
├── dashboard/              # Tauri desktop UI
│   ├── src-tauri/          # Rust backend — queries log store, streams live events
│   └── src/                # Web frontend — real-time charts, event tables, reports
├── cli/                    # Command-line query tool
│   └── src/
│       └── main.rs         # sentinel query, sentinel report, sentinel status
├── reports/                # Report templates and export formats
├── docs/
│   ├── METHODOLOGY.md      # How Sentinel captures events, what it can and cannot see
│   ├── ANTI-CHEAT-MAP.md   # Known anti-cheat drivers, behaviors, and load patterns
│   ├── LEGAL.md            # User rights, CFAA considerations, ToS analysis
│   └── DRIVER-SIGNING.md   # How to get the filter driver signed
└── README.md
```

### Components

**Filter Driver** — A minimal kernel-mode minifilter driver that registers callbacks for process, file, and registry operations. It does not modify any operations — it only observes and forwards events to the ETW pipeline. The driver must be signed with a valid EV certificate for Windows to load it.

**Collection Service** — A user-mode Windows service that consumes ETW events in real time, enriches them with process metadata (name, path, publisher), and writes them to a local SQLite database. Handles log rotation and configurable retention periods.

**Dashboard** — A Tauri application providing real-time visualization of kernel activity. Connects to the collection service for live streaming and queries the SQLite database for historical analysis.

**CLI** — A command-line interface for scripted queries, report generation, and integration with other tools. Designed for power users and automated auditing pipelines.

---

## Technical Approach

### ETW (Event Tracing for Windows)

The primary collection mechanism. ETW is Microsoft's built-in kernel instrumentation framework — it's what Process Monitor, Windows Performance Analyzer, and Microsoft's own security tools use. Sentinel subscribes to:

- `Microsoft-Windows-Kernel-Process` — process creation, termination, image loads
- `Microsoft-Windows-Kernel-File` — file create, read, write, delete
- `Microsoft-Windows-Kernel-Registry` — registry open, query, set, delete
- `Microsoft-Windows-Kernel-Network` — TCP/UDP connection events
- `Microsoft-Windows-Kernel-Audit-API-Calls` — sensitive API usage

### Minifilter Driver (Optional, Deeper Visibility)

For events that ETW alone cannot capture at sufficient granularity (such as which specific driver initiated a read, rather than just which process), Sentinel includes an optional minifilter driver. This driver:

- Registers with the Filter Manager (standard Windows mechanism)
- Receives pre/post callbacks for file, registry, and process operations
- Tags each event with the originating driver's identity
- Forwards events to the user-mode service via a communication port
- **Never blocks, modifies, or delays any operation**

The minifilter is optional. Sentinel produces useful output from ETW alone — the minifilter adds driver-level attribution.

### Driver Signing

Windows requires kernel drivers to be signed. For development, test signing mode can be used. For distribution, the driver requires:

1. An EV (Extended Validation) code signing certificate
2. Submission to Microsoft's Hardware Developer Center for attestation signing
3. WHQL testing (optional but recommended)

See `docs/DRIVER-SIGNING.md` for the full process.

---

## Usage

### Installation

```powershell
# Install the collection service
sentinel install

# (Optional) Install the minifilter driver
sentinel install --driver

# Start monitoring
sentinel start
```

### Live Monitoring

```powershell
# Watch all kernel driver activity in real time
sentinel watch

# Filter to a specific driver
sentinel watch --driver vgk.sys

# Filter to network activity only
sentinel watch --events network

# Open the dashboard
sentinel dashboard
```

### Reports

```powershell
# Generate a report for today's Vanguard activity
sentinel report --driver vgk.sys --today

# Generate a report for all anti-cheat drivers over the past week
sentinel report --category anticheat --days 7

# Export as JSON for further analysis
sentinel report --driver vgk.sys --format json --output vanguard-audit.json

# Compare two sessions
sentinel diff --before "2026-03-10" --after "2026-03-15" --driver vgk.sys
```

### Queries

```powershell
# What files did EasyAntiCheat access outside its game directory?
sentinel query files --driver EasyAntiCheat.sys --exclude "C:\Program Files\EpicGames\*"

# What processes did BattlEye scan?
sentinel query processes --driver BEDaisy.sys --sort-by access_count

# How much data did Vanguard send today?
sentinel query network --driver vgk.sys --direction outbound --today

# Is any anti-cheat still running right now?
sentinel query drivers --category anticheat --status active
```

---

## Known Anti-Cheat Drivers

| Game | Anti-Cheat | Driver | Load Behavior |
|------|-----------|--------|---------------|
| Valorant | Vanguard | `vgk.sys` | Boot-start — loads before desktop, persists after game closes |
| Fortnite | EasyAntiCheat | `EasyAntiCheat.sys` | On-demand — loads with game, should unload on exit |
| Call of Duty | Ricochet | `atvi-reven.sys` | On-demand — kernel driver loads with game |
| PUBG / DayZ / Ark | BattlEye | `BEDaisy.sys` | On-demand — loads with game |
| Apex Legends | EasyAntiCheat | `EasyAntiCheat.sys` | On-demand — shared with other EAC titles |
| Genshin Impact | mhyprot2 | `mhyprot2.sys` | On-demand — has been exploited as an attack vector by malware |

Sentinel maintains a database of known anti-cheat driver signatures and behaviors in `docs/ANTI-CHEAT-MAP.md`. Community contributions welcome.

---

## FAQ

**Will this get me banned?**
Sentinel is a passive system monitor. It does not interact with game processes or anti-cheat software. However, some anti-cheat systems detect the presence of *any* kernel-level monitoring tool and may flag it. Use at your own risk, and consult the specific game's Terms of Service. Sentinel's authors are not responsible for account actions taken by game publishers.

**Is this legal?**
Monitoring software running on your own hardware is legal in most jurisdictions. Sentinel does not reverse-engineer, decompile, or modify any anti-cheat software — it observes system-level events that Windows exposes through documented APIs. See `docs/LEGAL.md` for detailed analysis.

**Why not just use Process Monitor?**
Process Monitor (ProcMon) is excellent but wasn't designed for persistent auditing. It's a real-time debugging tool — you have to be running it at the right time, it doesn't filter well by driver origin, and it doesn't generate reports or maintain history. Sentinel runs as a background service and builds a searchable audit trail over time.

**Does this work without the kernel driver?**
Yes. ETW-only mode captures the majority of useful events. The minifilter driver adds driver-level attribution (knowing *which* kernel driver initiated an operation, not just which process) and catches events in code paths that bypass ETW.

**Can I use this to audit non-gaming software?**
Absolutely. Sentinel monitors all kernel drivers, not just anti-cheat. DRM systems, hardware vendor drivers, security software, enterprise management tools — anything running in kernel mode is observable.

---

## Requirements

- Windows 10 21H2+ or Windows 11
- Administrator privileges (for ETW kernel sessions)
- (Optional) Test signing mode enabled for development driver builds
- Rust toolchain for building from source

---

## Contributing

Sentinel's effectiveness depends on community knowledge about anti-cheat driver behaviors. Contributions welcome:

- **Driver behavior profiles** — document what a specific anti-cheat driver does on your machine
- **ETW event coverage** — identify events that Sentinel should capture but doesn't
- **Report templates** — design useful audit report formats for specific use cases
- **Legal research** — ToS analysis for specific games regarding system monitoring tools

---

## Author

**Kase** — [github.com/kase1111-hash](https://github.com/kase1111-hash)

---

## License

MIT — Transparency is not a privilege. It's a property right.

---

*"You bought the hardware. You have the right to know what it's doing."*
