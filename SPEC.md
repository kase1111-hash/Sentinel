# SPEC.md — Sentinel Technical Specification

**Author:** Kase — github.com/kase1111-hash  
**Version:** 1.0.0  
**Date:** 2026-03-16  
**Purpose:** Complete coding reference for building Sentinel from scratch via Claude Code sessions  

---

## How to Use This Document

This is the engineering blueprint. Every component, every data structure, every API surface, every database schema, and every build step is specified here. Open a Claude Code session, point it at the relevant section, and start building. Each phase produces a working, testable increment.

---

## Build Order

Dependencies flow strictly downward. Do not skip phases.

```
Phase 1: Shared types + database schema (foundation everything else uses)
Phase 2: ETW consumer service (the core — captures events from Windows)
Phase 3: CLI tool (query and report against collected data)
Phase 4: Alert engine (rule-based notifications)
Phase 5: Vulnerable driver check (known-vulnerable / malicious driver lists)
Phase 6: Dashboard (Tauri UI for real-time monitoring)
Phase 7: Minifilter driver (optional — adds driver-level attribution)
Phase 8: Anti-cheat profile database (community knowledge base)
```

---

## Repository Structure

```
sentinel/
├── Cargo.toml                      # Workspace root
├── README.md
├── SPEC.md                         # This document
├── LICENSE
│
├── crates/
│   ├── sentinel-types/             # Shared types, enums, event structs
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   │
│   ├── sentinel-store/             # SQLite storage layer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── schema.rs           # Table definitions, migrations
│   │       ├── writer.rs           # Batch event insertion
│   │       ├── reader.rs           # Query interface
│   │       └── rotation.rs         # Log rotation and retention
│   │
│   ├── sentinel-etw/               # ETW consumer engine
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── session.rs          # ETW trace session management
│   │       ├── providers.rs        # Provider GUIDs and enable flags
│   │       ├── parser.rs           # Raw ETW event → SentinelEvent
│   │       ├── enrichment.rs       # PID → process name/path resolution
│   │       └── driver_catalog.rs   # Known driver identification
│   │
│   ├── sentinel-analyzer/          # Event correlation and report generation
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── correlator.rs       # Group events into driver sessions
│   │       ├── report.rs           # Report generation (text, JSON, HTML)
│   │       ├── diff.rs             # Session comparison
│   │       └── stats.rs            # Aggregate statistics
│   │
│   ├── sentinel-alert/             # Rule engine and notification
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── rules.rs            # Rule definition and matching
│   │       └── notify.rs           # Notification dispatch (toast, log, webhook)
│   │
│   └── sentinel-vulndb/            # Known-vulnerable driver check (Phase 5)
│       ├── Cargo.toml
│       ├── tests/fixtures/         # List fixtures (LOLDrivers JSON, SiPolicy XML)
│       └── src/
│           ├── lib.rs              # VulnDb: list index and lookup
│           ├── hashing.rs          # SHA-1/SHA-256 + Authenticode hashes
│           ├── loldrivers.rs       # LOLDrivers drivers.json parser
│           ├── ms_blocklist.rs     # Microsoft blocklist (SiPolicy XML) parser
│           ├── paths.rs            # Driver path normalization
│           └── scanner.rs          # Loaded / registered / on-disk enumeration (Windows)
│
├── bins/
│   ├── sentinel-service/           # Windows service binary
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs             # Service entry, lifecycle management
│   │       ├── service.rs          # Windows Service Control Manager integration
│   │       ├── pipeline.rs         # ETW → enrichment → store → alert pipeline
│   │       └── ipc_server.rs       # Named pipe server for CLI/dashboard
│   │
│   ├── sentinel-cli/               # Command-line tool
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs             # Argument parsing, subcommand dispatch
│   │       ├── commands/
│   │       │   ├── mod.rs
│   │       │   ├── install.rs      # sentinel install / uninstall
│   │       │   ├── start_stop.rs   # sentinel start / stop / status
│   │       │   ├── watch.rs        # sentinel watch (live stream)
│   │       │   ├── query.rs        # sentinel query (database queries)
│   │       │   ├── report.rs       # sentinel report (generate reports)
│   │       │   ├── diff.rs         # sentinel diff (compare sessions)
│   │       │   └── vulndb.rs       # sentinel vulndb update / status
│   │       └── output.rs           # Terminal formatting and colors
│   │
│   └── sentinel-dashboard/         # Tauri desktop UI
│       ├── Cargo.toml
│       ├── tauri.conf.json
│       ├── src-tauri/src/
│       │   ├── main.rs
│       │   ├── ipc_client.rs       # Connect to service via named pipe
│       │   ├── commands.rs         # Tauri commands for frontend
│       │   └── live_stream.rs      # WebSocket-style event streaming to frontend
│       └── src/
│           ├── index.html
│           ├── style.css
│           ├── app.js
│           └── components/
│               ├── driver-list.js
│               ├── event-stream.js
│               ├── network-graph.js
│               ├── process-heatmap.js
│               ├── report-viewer.js
│               └── alert-rules.js
│
├── driver/                         # Kernel minifilter (Phase 7 — optional)
│   ├── sentinel-filter/
│   │   ├── sentinel-filter.inf     # Driver installation manifest
│   │   ├── sentinel-filter.c       # Minifilter registration and callbacks
│   │   ├── callbacks.c             # Pre/post operation callbacks
│   │   ├── comm.c                  # User-mode communication port
│   │   └── Makefile / build.cmd    # WDK build scripts
│   └── certs/
│       └── README.md               # Signing instructions
│
├── profiles/                       # Anti-cheat driver behavior profiles
│   ├── schema.toml                 # Profile format definition
│   ├── vanguard.toml
│   ├── easyanticheat.toml
│   ├── battleye.toml
│   ├── ricochet.toml
│   └── mhyprot2.toml
│
├── docs/
│   ├── METHODOLOGY.md
│   ├── ANTI-CHEAT-MAP.md
│   ├── LEGAL.md
│   ├── DRIVER-SIGNING.md
│   └── FEASIBILITY.md              # Design review: what works, what can't be observed
│
└── scripts/
    ├── install-service.ps1
    ├── uninstall-service.ps1
    └── enable-test-signing.ps1
```

---

## Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/sentinel-types",
    "crates/sentinel-store",
    "crates/sentinel-etw",
    "crates/sentinel-analyzer",
    "crates/sentinel-alert",
    "crates/sentinel-vulndb",
    "bins/sentinel-service",
    "bins/sentinel-cli",
    "bins/sentinel-dashboard/src-tauri",
]
```

---

## Phase 1 — Shared Types + Database

**Sessions:** 2–3  
**Goal:** Define every data structure the system uses and build the storage layer.

### 1.1 — sentinel-types

The canonical event type that flows through the entire system.

```toml
# crates/sentinel-types/Cargo.toml
[package]
name = "sentinel-types"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
chrono = { version = "0.4", features = ["serde"] }
```

#### Core Event Struct

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// Unique event identifier.
pub type EventId = u64;

/// Process identifier.
pub type Pid = u32;

/// The canonical event type. Every event in the system passes through as this struct.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SentinelEvent {
    /// Auto-assigned by the store on insertion.
    pub id: Option<EventId>,

    /// When the event occurred (UTC).
    pub timestamp: DateTime<Utc>,

    /// What category of event this is.
    pub event_type: EventType,

    /// The process that initiated the operation.
    pub source: ProcessInfo,

    /// The target of the operation (file path, process, registry key, IP, etc.)
    pub target: String,

    /// Additional structured details depending on event_type.
    pub details: EventDetails,

    /// If attributable to a specific kernel driver, its identity.
    pub driver: Option<DriverInfo>,

    /// Which collection mechanism captured this event.
    pub source_provider: Provider,
}

/// High-level event categories.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum EventType {
    ProcessAccess,       // Handle opened to another process (with requested access rights).
                         // Not memory reads: those are not observable (see Driver attribution).
    ProcessCreate,       // New process spawned
    ProcessTerminate,    // Process exited
    ImageLoad,           // DLL or driver loaded into a process/kernel

    FileCreate,          // File opened or created
    FileRead,            // File read operation
    FileWrite,           // File write operation
    FileDelete,          // File deleted
    FileRename,          // File renamed or moved
    DirectoryEnum,       // Directory listing / enumeration

    RegistryOpen,        // Registry key opened
    RegistryQuery,       // Registry value read
    RegistrySet,         // Registry value written
    RegistryDelete,      // Registry key or value deleted

    NetworkConnect,      // Outbound TCP connection initiated
    NetworkAccept,       // Inbound TCP connection accepted
    NetworkSend,         // Data sent (with byte count)
    NetworkReceive,      // Data received (with byte count)
    DnsQuery,            // DNS resolution request

    DriverLoad,          // Kernel driver loaded
    DriverUnload,        // Kernel driver unloaded
}

impl EventType {
    /// Returns the broad category for filtering.
    pub fn category(&self) -> EventCategory {
        match self {
            Self::ProcessAccess | Self::ProcessCreate |
            Self::ProcessTerminate | Self::ImageLoad => EventCategory::Process,

            Self::FileCreate | Self::FileRead | Self::FileWrite |
            Self::FileDelete | Self::FileRename | Self::DirectoryEnum => EventCategory::File,

            Self::RegistryOpen | Self::RegistryQuery |
            Self::RegistrySet | Self::RegistryDelete => EventCategory::Registry,

            Self::NetworkConnect | Self::NetworkAccept |
            Self::NetworkSend | Self::NetworkReceive |
            Self::DnsQuery => EventCategory::Network,

            Self::DriverLoad | Self::DriverUnload => EventCategory::Driver,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum EventCategory {
    Process,
    File,
    Registry,
    Network,
    Driver,
}

/// Information about the process that initiated the event.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProcessInfo {
    pub pid: Pid,
    pub name: String,
    pub path: Option<String>,
    /// The publisher from the digital signature, if available.
    pub publisher: Option<String>,
    /// Parent process ID.
    pub parent_pid: Option<Pid>,
}

/// Information about a kernel driver.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DriverInfo {
    /// Driver filename (e.g., "vgk.sys")
    pub name: String,
    /// Full path on disk
    pub path: Option<String>,
    /// Publisher from digital signature
    pub publisher: Option<String>,
    /// Whether this matches a known anti-cheat profile
    pub profile: Option<String>,
    /// Load address in kernel memory (for correlation)
    pub load_address: Option<u64>,
}

/// Structured detail payload — varies by event type.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "kind")]
pub enum EventDetails {
    ProcessAccess {
        target_pid: Pid,
        target_name: String,
        access_mask: u32,
        /// Human-readable access rights
        access_rights: Vec<String>,
    },
    FileOp {
        operation: String,
        size_bytes: Option<u64>,
        /// Was this inside or outside the driver's expected working directory?
        outside_expected_dir: bool,
    },
    RegistryOp {
        key_path: String,
        value_name: Option<String>,
        value_data: Option<String>,
    },
    NetworkOp {
        local_addr: String,
        local_port: u16,
        remote_addr: String,
        remote_port: u16,
        protocol: String,
        bytes_transferred: Option<u64>,
    },
    DnsOp {
        query_name: String,
        query_type: String,
        response: Option<String>,
    },
    DriverLifecycle {
        action: String,      // "load" or "unload"
        image_size: Option<u64>,
    },
    ImageLoad {
        image_name: String,
        image_path: String,
        image_size: u64,
    },
    /// Fallback for events with no structured details.
    Generic {
        description: String,
    },
}

/// Which subsystem captured this event.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Provider {
    /// Standard ETW kernel providers.
    EtwKernel,
    /// Sentinel's custom minifilter driver.
    Minifilter,
}

/// Driver session — a contiguous period where a driver was loaded.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DriverSession {
    pub driver: DriverInfo,
    pub loaded_at: DateTime<Utc>,
    pub unloaded_at: Option<DateTime<Utc>>,
    pub event_count: u64,
    pub process_access_count: u64,
    pub file_access_count: u64,
    pub registry_access_count: u64,
    pub network_connection_count: u64,
    pub bytes_sent: u64,
    pub bytes_received: u64,
}

/// A filter predicate for querying events.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct EventFilter {
    pub driver_name: Option<String>,
    pub event_types: Option<Vec<EventType>>,
    pub categories: Option<Vec<EventCategory>>,
    pub target_pattern: Option<String>,       // Glob pattern against target field
    pub exclude_pattern: Option<String>,      // Exclude targets matching this glob
    pub source_pid: Option<Pid>,
    pub source_name: Option<String>,
    pub after: Option<DateTime<Utc>>,
    pub before: Option<DateTime<Utc>>,
    pub limit: Option<u64>,
    pub sort_by: Option<SortField>,
    pub sort_desc: bool,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum SortField {
    Timestamp,
    EventType,
    Target,
    SourceName,
    DriverName,
}

/// Anti-cheat driver profile loaded from TOML (see Phase 8 for the file format).
/// Mirrors the TOML layout so `toml::from_str::<DriverProfile>(..)` works directly.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DriverProfile {
    pub driver: ProfileDriver,
    pub games: ProfileGames,
    pub expected_behavior: ExpectedBehavior,
    #[serde(default)]
    pub known_activity: KnownActivity,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProfileDriver {
    pub name: String,
    pub vendor: String,
    /// Kernel driver and user-mode component filenames (e.g., "vgk.sys", "vgc.exe").
    pub filenames: Vec<String>,
    pub publishers: Vec<String>,
    pub load_behavior: LoadBehavior,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProfileGames {
    pub titles: Vec<String>,
    /// Game executables. Needed to detect "game exited" for PersistenceAfterExit.
    pub executables: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExpectedBehavior {
    pub install_directories: Vec<String>,
    /// Hostname globs. Network events carry IPs, so matching requires correlating
    /// with DNS-Client events from the same process.
    pub network_destinations: Vec<String>,
}

/// Community-reported behavior. Sentinel does not verify these claims — several
/// (e.g., memory reads) are outside what it can observe. Each claim needs a source.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct KnownActivity {
    pub scans_all_processes: Option<bool>,
    pub scans_at_boot: Option<bool>,
    pub persists_after_game_exit: Option<bool>,
    pub reads_browser_memory: Option<bool>,
    pub enumerates_installed_software: Option<bool>,
    pub checks_hypervisor_presence: Option<bool>,
    #[serde(default)]
    pub notes: String,
    /// URLs or citations backing the claims above.
    #[serde(default)]
    pub sources: Vec<String>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LoadBehavior {
    /// Loads at boot, persists regardless of game state.
    BootStart,
    /// Loads when game starts, should unload when game exits.
    OnDemand,
    /// Loads when game starts, may persist after game exits.
    OnDemandPersistent,
}

// ── Known-vulnerable driver check (Phase 5) ──

/// Hashes of a driver image. Lowercase hex.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct DriverHashes {
    pub sha1: String,
    pub sha256: String,
    /// Authenticode hashes exclude the PE checksum and signature, so they survive
    /// re-signing. None if the file isn't a parseable PE image.
    pub authentihash_sha1: Option<String>,
    pub authentihash_sha256: Option<String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum HashKind {
    Sha1,
    Sha256,
    AuthentihashSha1,
    AuthentihashSha256,
}

/// Which list an entry came from.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum VulnSource {
    /// loldrivers.io — community list of vulnerable and malicious drivers (Apache-2.0).
    LolDrivers,
    /// Microsoft's recommended driver block rules (the Vulnerable Driver Blocklist).
    MicrosoftBlocklist,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum VulnCategory {
    /// Legitimately signed driver with an exploitable flaw (e.g., mhyprot2.sys).
    Vulnerable,
    /// Driver written or signed for malicious use.
    Malicious,
    /// Listed without a stated category (Microsoft's blocklist doesn't say which).
    Unknown,
}

/// One list entry that matched a driver's hashes.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VulnMatch {
    pub source: VulnSource,
    /// LOLDrivers `Id` (UUID) or the blocklist rule `ID`.
    pub entry_id: String,
    pub category: VulnCategory,
    pub matched_on: HashKind,
    /// LOLDrivers sample filename, or the blocklist rule's FriendlyName.
    pub name: String,
    /// CVEs and reference URLs, where the list provides them.
    pub references: Vec<String>,
}

/// A kernel or file-system driver registered as a service: loadable later even if not loaded now.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DriverService {
    pub name: String,
    pub start: ServiceStart,
}

/// The service `Start` value.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ServiceStart {
    Boot,     // 0
    System,   // 1
    Auto,     // 2
    Demand,   // 3
    Disabled, // 4
}

/// A driver file on this machine that matches a known-vulnerable or malicious driver list.
/// One finding per (path, sha256).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VulnerableDriverFinding {
    pub path: String,
    pub hashes: DriverHashes,
    /// Loaded in the kernel as of `last_seen`.
    pub loaded: bool,
    /// Registered as a driver service as of `last_seen`.
    pub service: Option<DriverService>,
    pub matches: Vec<VulnMatch>,
    pub first_seen: DateTime<Utc>,
    pub last_seen: DateTime<Utc>,
    /// Set when a full scan no longer finds the file, the service, or the loaded image.
    pub resolved_at: Option<DateTime<Utc>>,
}

/// Metadata for one loaded list.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VulnDbSourceInfo {
    pub source: VulnSource,
    pub retrieved_at: DateTime<Utc>,
    pub entries: u64,
    pub hashes: u64,
}

/// Result of the most recent scan, returned alongside findings.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct VulnScanSummary {
    pub scanned_at: Option<DateTime<Utc>>,
    pub files_checked: u64,
    /// Loaded drivers whose image file couldn't be read (deleted after loading, access
    /// denied). Sentinel can't check these, and a deleted image is itself suspicious.
    pub unverifiable_loaded: Vec<String>,
    pub sources: Vec<VulnDbSourceInfo>,
    /// Microsoft's Vulnerable Driver Blocklist setting: Some(on/off), or None when the
    /// registry value is absent (Windows' default applies) or unreadable.
    pub blocklist_enabled: Option<bool>,
}
```

#### IPC Message Types (service ↔ CLI/dashboard)

```rust
/// Messages sent to the sentinel service via named pipe.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum ServiceRequest {
    /// Get current service status.
    Status,
    /// Subscribe to live event stream (returns events until disconnect).
    Subscribe { filter: EventFilter },
    /// Query historical events from the database.
    Query { filter: EventFilter },
    /// Generate a report.
    Report { driver_name: Option<String>, days: u32, format: ReportFormat },
    /// Compare two time periods.
    Diff { driver_name: String, before: DateTime<Utc>, after: DateTime<Utc> },
    /// List all known drivers currently loaded or historically seen.
    ListDrivers { active_only: bool },
    /// Get alert rules.
    GetAlertRules,
    /// Set alert rules.
    SetAlertRules { rules: Vec<AlertRule> },
    /// Known-vulnerable driver findings. `rescan` runs a full scan first.
    VulnerableDrivers { rescan: bool, include_resolved: bool },
    /// Reload the vulnerable-driver lists from disk (sent by `sentinel vulndb update`).
    ReloadVulnDb,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum ServiceResponse {
    Status {
        running: bool,
        uptime_seconds: u64,
        events_collected: u64,
        events_today: u64,
        active_drivers: Vec<DriverInfo>,
        db_size_mb: f64,
        etw_sessions_active: u32,
        /// Unresolved known-vulnerable driver findings.
        vulnerable_drivers: u32,
    },
    Events { events: Vec<SentinelEvent>, total: u64 },
    Report { content: String, format: ReportFormat },
    Drivers { drivers: Vec<DriverSession> },
    AlertRules { rules: Vec<AlertRule> },
    VulnerableDrivers { findings: Vec<VulnerableDriverFinding>, summary: VulnScanSummary },
    Error { message: String },
    Ok,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ReportFormat {
    Text,
    Json,
    Html,
}

/// A single alert rule.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AlertRule {
    pub id: String,
    pub name: String,
    pub enabled: bool,
    pub condition: AlertCondition,
    pub action: AlertAction,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "kind")]
pub enum AlertCondition {
    /// Trigger when a specific driver accesses a file matching a pattern.
    FileAccess { driver_pattern: String, path_pattern: String },
    /// Trigger when a driver accesses a process matching a pattern.
    ProcessAccess { driver_pattern: String, process_pattern: String },
    /// Trigger when network bytes exceed threshold in a time window.
    NetworkThreshold { driver_pattern: String, bytes_per_minute: u64 },
    /// Trigger when a driver is still loaded N minutes after its game exits.
    PersistenceAfterExit { driver_pattern: String, minutes_after: u32 },
    /// Trigger when any new kernel driver loads.
    NewDriverLoad,
    /// Trigger when a driver on a vulnerable/malicious driver list is found
    /// (new finding) or loaded. `loaded_only` ignores drivers that are only
    /// registered or on disk.
    VulnerableDriver { loaded_only: bool },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "kind")]
pub enum AlertAction {
    Log,
    Toast { title: String, body: String },
    Webhook { url: String },
}
```

### 1.2 — sentinel-store

SQLite database with event storage, querying, and rotation.

```toml
# crates/sentinel-store/Cargo.toml
[package]
name = "sentinel-store"
version = "0.1.0"
edition = "2021"

[dependencies]
sentinel-types = { path = "../sentinel-types" }
rusqlite = { version = "0.31", features = ["bundled", "vtab"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["serde"] }
```

#### Database Schema

```sql
-- Core event log. This is the primary table — everything flows here.
CREATE TABLE events (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   TEXT NOT NULL,               -- ISO 8601 UTC
    event_type  TEXT NOT NULL,               -- EventType enum serialized as string
    category    TEXT NOT NULL,               -- EventCategory for fast filtering

    -- Source process
    source_pid      INTEGER NOT NULL,
    source_name     TEXT NOT NULL,
    source_path     TEXT,
    source_publisher TEXT,

    -- Target (file path, process name, registry key, IP:port, etc.)
    target      TEXT NOT NULL,

    -- Driver attribution (nullable — ETW-only events may not have this)
    driver_name     TEXT,
    driver_path     TEXT,
    driver_publisher TEXT,
    driver_profile  TEXT,                    -- Profile name if matched

    -- Event-specific details as JSON blob
    details     TEXT NOT NULL,               -- JSON serialization of EventDetails

    -- Collection source
    provider    TEXT NOT NULL DEFAULT 'etw'  -- 'etw' or 'minifilter'
);

-- Indexes for common query patterns
CREATE INDEX idx_events_timestamp ON events(timestamp);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_category ON events(category);
CREATE INDEX idx_events_driver ON events(driver_name);
CREATE INDEX idx_events_source ON events(source_name);
CREATE INDEX idx_events_target ON events(target);

-- Compound index for the most common query: "what did driver X do today?"
CREATE INDEX idx_events_driver_time ON events(driver_name, timestamp);

-- FTS5 for text search across targets and details
CREATE VIRTUAL TABLE events_fts USING fts5(
    target,
    source_name,
    driver_name,
    content='events',
    content_rowid='id',
    tokenize='unicode61'
);

-- FTS sync triggers
CREATE TRIGGER events_fts_insert AFTER INSERT ON events BEGIN
    INSERT INTO events_fts(rowid, target, source_name, driver_name)
    VALUES (new.id, new.target, new.source_name, new.driver_name);
END;

CREATE TRIGGER events_fts_delete AFTER DELETE ON events BEGIN
    INSERT INTO events_fts(events_fts, rowid, target, source_name, driver_name)
    VALUES ('delete', old.id, old.target, old.source_name, old.driver_name);
END;

-- Without this, any UPDATE (e.g., backfilling driver attribution) desyncs the
-- external-content FTS index and its integrity-check reports corruption.
CREATE TRIGGER events_fts_update AFTER UPDATE ON events BEGIN
    INSERT INTO events_fts(events_fts, rowid, target, source_name, driver_name)
    VALUES ('delete', old.id, old.target, old.source_name, old.driver_name);
    INSERT INTO events_fts(rowid, target, source_name, driver_name)
    VALUES (new.id, new.target, new.source_name, new.driver_name);
END;

-- Driver sessions table — aggregated view of driver activity periods
CREATE TABLE driver_sessions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    driver_name     TEXT NOT NULL,
    driver_path     TEXT,
    driver_publisher TEXT,
    driver_profile  TEXT,
    loaded_at       TEXT NOT NULL,
    unloaded_at     TEXT,                   -- NULL if still loaded
    event_count     INTEGER DEFAULT 0,
    process_access_count INTEGER DEFAULT 0,
    file_access_count    INTEGER DEFAULT 0,
    registry_access_count INTEGER DEFAULT 0,
    network_connection_count INTEGER DEFAULT 0,
    bytes_sent      INTEGER DEFAULT 0,
    bytes_received  INTEGER DEFAULT 0
);

CREATE INDEX idx_sessions_driver ON driver_sessions(driver_name);
CREATE INDEX idx_sessions_time ON driver_sessions(loaded_at);

-- Known drivers — persisted catalog of every driver ever seen
CREATE TABLE known_drivers (
    name            TEXT PRIMARY KEY,
    path            TEXT,
    publisher       TEXT,
    first_seen      TEXT NOT NULL,
    last_seen       TEXT NOT NULL,
    profile         TEXT,                   -- Matched profile name
    total_events    INTEGER DEFAULT 0,
    is_anticheat    INTEGER DEFAULT 0       -- Boolean: matched an anti-cheat profile
);

-- Alert rules (persisted)
CREATE TABLE alert_rules (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    enabled     INTEGER NOT NULL DEFAULT 1,
    condition   TEXT NOT NULL,               -- JSON serialization of AlertCondition
    action      TEXT NOT NULL                -- JSON serialization of AlertAction
);

-- Known-vulnerable driver findings (Phase 5). One row per (path, sha256).
CREATE TABLE vulnerable_driver_findings (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    path        TEXT NOT NULL,
    sha256      TEXT NOT NULL,
    hashes      TEXT NOT NULL,               -- JSON serialization of DriverHashes
    loaded      INTEGER NOT NULL DEFAULT 0,  -- Boolean, as of last_seen
    service     TEXT,                        -- JSON DriverService, NULL if not registered
    matches     TEXT NOT NULL,               -- JSON Vec<VulnMatch>
    first_seen  TEXT NOT NULL,
    last_seen   TEXT NOT NULL,
    resolved_at TEXT,                        -- NULL while the driver is still present
    UNIQUE (path, sha256)
);

CREATE INDEX idx_vuln_unresolved ON vulnerable_driver_findings(resolved_at);

-- Hash cache so scheduled scans only re-hash files whose size or write time changed.
CREATE TABLE file_hash_cache (
    path        TEXT PRIMARY KEY,
    size        INTEGER NOT NULL,
    modified    TEXT NOT NULL,               -- Last-write time (ISO 8601 UTC)
    hashes      TEXT NOT NULL                -- JSON serialization of DriverHashes
);

-- Metadata table for schema versioning and config
CREATE TABLE metadata (
    key     TEXT PRIMARY KEY,
    value   TEXT NOT NULL
);

INSERT INTO metadata (key, value) VALUES ('schema_version', '1');
INSERT INTO metadata (key, value) VALUES ('created_at', datetime('now'));
```

#### Storage API Surface

```rust
pub struct Store {
    conn: rusqlite::Connection,
}

impl Store {
    /// Open or create the database at the given path.
    pub fn open(path: &Path) -> Result<Self>;

    /// Open at the default location: %ProgramData%\Sentinel\data\sentinel.db
    pub fn open_default() -> Result<Self>;

    // ── Writing ──

    /// Insert a batch of events. Uses a transaction for performance.
    /// Called by the pipeline after enrichment.
    /// Batch size recommendation: 100-500 events per call.
    pub fn insert_events(&self, events: &[SentinelEvent]) -> Result<u64>;

    /// Create or update a driver session.
    pub fn upsert_driver_session(&self, session: &DriverSession) -> Result<()>;

    /// Register a driver in the known_drivers catalog.
    pub fn register_driver(&self, driver: &DriverInfo) -> Result<()>;

    // ── Reading ──

    /// Query events using a filter. Returns matching events and total count.
    pub fn query_events(&self, filter: &EventFilter) -> Result<(Vec<SentinelEvent>, u64)>;

    /// Get all driver sessions, optionally filtered to active-only.
    pub fn get_driver_sessions(&self, active_only: bool) -> Result<Vec<DriverSession>>;

    /// Get sessions for a specific driver.
    pub fn get_sessions_for_driver(&self, driver_name: &str) -> Result<Vec<DriverSession>>;

    /// Get all known drivers.
    pub fn get_known_drivers(&self) -> Result<Vec<DriverInfo>>;

    /// Full-text search across events.
    pub fn search_events(&self, query: &str, limit: u64) -> Result<Vec<SentinelEvent>>;

    /// Get aggregate stats for a driver in a time range.
    pub fn get_driver_stats(
        &self,
        driver_name: &str,
        after: DateTime<Utc>,
        before: DateTime<Utc>,
    ) -> Result<DriverStats>;

    /// Get top N processes accessed by a driver.
    pub fn get_top_process_targets(
        &self,
        driver_name: &str,
        limit: usize,
    ) -> Result<Vec<(String, u64)>>;

    /// Get files accessed outside expected directories.
    pub fn get_unexpected_file_access(
        &self,
        driver_name: &str,
        expected_dirs: &[String],
    ) -> Result<Vec<SentinelEvent>>;

    /// Get network destinations for a driver.
    pub fn get_network_destinations(
        &self,
        driver_name: &str,
    ) -> Result<Vec<NetworkDestination>>;

    /// Get today's event count.
    pub fn count_events_today(&self) -> Result<u64>;

    /// Get total event count.
    pub fn count_events_total(&self) -> Result<u64>;

    /// Get database file size in MB.
    pub fn db_size_mb(&self) -> Result<f64>;

    // ── Maintenance ──

    /// Delete events older than the given number of days.
    pub fn rotate(&self, retention_days: u32) -> Result<u64>;

    /// Vacuum the database to reclaim space after rotation.
    pub fn vacuum(&self) -> Result<()>;

    // ── Alert Rules ──

    pub fn get_alert_rules(&self) -> Result<Vec<AlertRule>>;
    pub fn set_alert_rules(&self, rules: &[AlertRule]) -> Result<()>;

    // ── Vulnerable Drivers (Phase 5) ──

    /// Insert a finding, or update loaded/service/matches/last_seen on an existing
    /// (path, sha256) and clear resolved_at.
    pub fn upsert_vuln_finding(&self, finding: &VulnerableDriverFinding) -> Result<()>;

    pub fn get_vuln_findings(&self, include_resolved: bool) -> Result<Vec<VulnerableDriverFinding>>;

    /// After a full scan: mark unresolved findings with last_seen < scan_started as
    /// resolved. Returns how many were resolved. Only call after a complete scan.
    pub fn resolve_vuln_findings_not_seen_since(&self, scan_started: DateTime<Utc>) -> Result<u64>;

    /// Cached hashes for a file, if its size and last-write time are unchanged.
    pub fn get_cached_hashes(&self, path: &str, size: u64, modified: DateTime<Utc>)
        -> Result<Option<DriverHashes>>;

    pub fn put_cached_hashes(&self, path: &str, size: u64, modified: DateTime<Utc>,
        hashes: &DriverHashes) -> Result<()>;
}

/// Aggregate statistics for a driver.
pub struct DriverStats {
    pub total_events: u64,
    pub process_accesses: u64,
    pub file_accesses: u64,
    pub registry_accesses: u64,
    pub network_connections: u64,
    pub bytes_sent: u64,
    pub bytes_received: u64,
    pub unique_process_targets: u64,
    pub unique_file_targets: u64,
    pub unique_network_destinations: u64,
}

/// Network destination summary.
pub struct NetworkDestination {
    pub remote_addr: String,
    pub remote_port: u16,
    pub connection_count: u64,
    pub bytes_sent: u64,
    pub bytes_received: u64,
    pub first_seen: DateTime<Utc>,
    pub last_seen: DateTime<Utc>,
}
```

#### Session Build Plan

**Session 1 — Types crate:**
- Define all structs, enums, and IPC message types as shown above
- Ensure everything derives Serialize/Deserialize
- Unit tests for serialization roundtrips
- Goal: `cargo test -p sentinel-types` passes

**Session 2 — Store crate (write path):**
- Create database with full schema
- Implement `open`, `open_default`, `insert_events` (batched with transaction)
- Implement `upsert_driver_session`, `register_driver`
- Implement `rotate` and `vacuum`
- Goal: can create db, insert 10,000 events, rotate old ones

**Session 3 — Store crate (read path):**
- Implement `query_events` with full EventFilter support
- Implement all get/count/search methods
- Implement `get_driver_stats`, `get_top_process_targets`, `get_unexpected_file_access`
- FTS5 search
- Goal: insert events, query them back with filters, get accurate stats

---

## Phase 2 — ETW Consumer Service

**Sessions:** 4–7  
**Goal:** A Windows service that captures real-time kernel events from ETW and writes them to the database.

### 2.1 — sentinel-etw

The ETW consumption engine. This is the hardest technical component.

```toml
# crates/sentinel-etw/Cargo.toml
[package]
name = "sentinel-etw"
version = "0.1.0"
edition = "2021"

[dependencies]
sentinel-types = { path = "../sentinel-types" }
chrono = { version = "0.4", features = ["serde"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[target.'cfg(windows)'.dependencies]
windows = { version = "0.58", features = [
    "Win32_System_Diagnostics_Etw",
    "Win32_Foundation",
    "Win32_System_Threading",
    "Win32_System_ProcessStatus",
    "Win32_Security",
] }
# Alternative: use the `ferrisetw` crate which wraps ETW in safe Rust
# ferrisetw = "1"
```

#### ETW Provider GUIDs

```rust
/// Microsoft-Windows-Kernel-Process
/// Events: process create/terminate, image load (DLL/driver)
pub const KERNEL_PROCESS_GUID: &str = "22FB2CD6-0E7B-422B-A0C7-2FAD1FD0E716";

/// Microsoft-Windows-Kernel-File
/// Events: file create, read, write, delete, rename
pub const KERNEL_FILE_GUID: &str = "EDD08927-9CC4-4E65-B970-C2560FB5C289";

/// Microsoft-Windows-Kernel-Registry
/// Events: registry open, query, set, delete
pub const KERNEL_REGISTRY_GUID: &str = "70EB4F03-C1DE-4F73-A051-33D13D5413BD";

/// Microsoft-Windows-Kernel-Network
/// Events: TCP/UDP connect, accept, send, receive
pub const KERNEL_NETWORK_GUID: &str = "7DD42A49-5329-4832-8DFD-43D979153A88";

/// Microsoft-Windows-Kernel-Audit-API-Calls
/// Events: OpenProcess (ID 5: TargetProcessId, DesiredAccess, ReturnCode),
/// OpenThread (ID 6), TerminateProcess (ID 2), PsSetLoadImageNotifyRoutine (ID 1).
/// Does NOT log memory reads (NtReadVirtualMemory / MmCopyVirtualMemory). Do not
/// assume kernel-mode handle opens appear here; treat it as user-mode coverage.
/// Source of every EventType::ProcessAccess event in ETW-only mode.
pub const KERNEL_AUDIT_GUID: &str = "E02A841C-75A3-4FA7-AFC8-AE09CF9B7F23";

// Not usable: Microsoft-Windows-Threat-Intelligence (logs READVM/WRITEVM) only
// delivers events to Antimalware-PPL consumers, which requires an ELAM driver
// signed through Microsoft's antimalware program.

/// Microsoft-Windows-DNS-Client
/// Events: DNS resolution
pub const DNS_CLIENT_GUID: &str = "1C95126E-7EEA-49A9-A3FE-A378B03DDB4D";
```

#### ETW Architecture

```
┌─────────────────────────────────────────────────┐
│                Windows Kernel                    │
│                                                  │
│  ETW Providers (always running in the kernel):   │
│    Kernel-Process  Kernel-File  Kernel-Registry  │
│    Kernel-Network  Kernel-Audit  DNS-Client      │
└──────────┬──────────┬───────────┬───────────────┘
           │          │           │
           ▼          ▼           ▼
┌─────────────────────────────────────────────────┐
│        Sentinel ETW Trace Session                │
│        (real-time consumer, admin required)       │
│                                                   │
│  ┌─────────────┐   ┌──────────────┐              │
│  │ Raw ETW     │──▶│ Parser       │──▶ SentinelEvent
│  │ Event Buffer│   │ (decode TDH) │              │
│  └─────────────┘   └──────────────┘              │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│             Enrichment Layer                      │
│                                                   │
│  PID → Process name/path (cached, refreshed)     │
│  Driver address → Driver name (from loaded list) │
│  Profile matching (is this a known anti-cheat?)  │
│  "Outside expected dir" tagging for file events  │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
                 Vec<SentinelEvent>
                 (batched, ready for store)
```

#### ETW Session Management

```rust
/// Configuration for the ETW session.
/// `#[serde(default)]` lets config.toml set only the fields it cares about.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct EtwConfig {
    /// Session name (must be unique on the system).
    pub session_name: String,
    /// Which providers to enable.
    pub providers: Vec<ProviderConfig>,
    /// Buffer size in KB.
    pub buffer_size_kb: u32,
    /// Minimum number of buffers.
    pub min_buffers: u32,
    /// Maximum number of buffers.
    pub max_buffers: u32,
    /// Flush timer in seconds.
    pub flush_timer_seconds: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProviderConfig {
    pub guid: String,
    pub level: TraceLevel,
    pub keywords: u64,      // Bitmask to select specific event categories
    /// Request kernel+user call stacks with each event (EVENT_ENABLE_PROPERTY_STACK_TRACE).
    /// This is how ETW-only mode attributes file/registry events to a driver.
    #[serde(default)]
    pub stack_trace: bool,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum TraceLevel {
    Critical = 1,
    Error = 2,
    Warning = 3,
    Info = 4,
    Verbose = 5,
}

impl Default for EtwConfig {
    fn default() -> Self {
        Self {
            session_name: "SentinelTrace".to_string(),
            providers: vec![
                ProviderConfig {
                    guid: KERNEL_PROCESS_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0x10 | 0x20 | 0x40,  // Process, Thread, Image
                    stack_trace: false,
                },
                ProviderConfig {
                    guid: KERNEL_FILE_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,  // All file operations
                    stack_trace: true,
                },
                ProviderConfig {
                    guid: KERNEL_REGISTRY_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
                    stack_trace: true,
                },
                ProviderConfig {
                    guid: KERNEL_NETWORK_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
                    stack_trace: false,
                },
                ProviderConfig {
                    guid: KERNEL_AUDIT_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
                    stack_trace: false,
                },
                ProviderConfig {
                    guid: DNS_CLIENT_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
                    stack_trace: false,
                },
            ],
            buffer_size_kb: 256,
            min_buffers: 4,
            max_buffers: 16,
            flush_timer_seconds: 1,
        }
    }
}

/// The main ETW consumer. Call `start()` to begin consuming, `stop()` to end.
/// Events are delivered via the callback provided to `start()`.
pub struct EtwConsumer {
    config: EtwConfig,
    session_handle: Option<TRACEHANDLE>,
    trace_handle: Option<TRACEHANDLE>,
}

impl EtwConsumer {
    pub fn new(config: EtwConfig) -> Self;

    /// Start the ETW trace session. The callback receives parsed events.
    /// This blocks the calling thread — run it in a dedicated thread.
    /// The callback should be fast; batch events and process them asynchronously.
    pub fn start<F>(&mut self, callback: F) -> Result<()>
    where
        F: Fn(SentinelEvent) + Send + Sync + 'static;

    /// Stop the trace session and clean up.
    pub fn stop(&mut self) -> Result<()>;

    /// Check if the session is currently active.
    pub fn is_running(&self) -> bool;
}
```

#### ETW Implementation Notes

**Option A — Direct Win32 API:**
Use `StartTrace`, `EnableTraceEx2`, `OpenTrace`, `ProcessTrace`, `StopTrace` from the `windows` crate. Parse events using TDH (Trace Data Helper) functions: `TdhGetEventInformation`, `TdhGetProperty`. This is verbose but gives full control.

**Option B — ferrisetw crate:**
The `ferrisetw` crate provides a safe Rust wrapper around ETW. Significantly less boilerplate. Recommended for initial implementation. Switch to raw API only if ferrisetw proves limiting.

```rust
// Example using ferrisetw 1.2 (preferred approach). Type-checked against
// ferrisetw 1.2.0 for x86_64-pc-windows-msvc.
use ferrisetw::native::ExtendedDataItem;
use ferrisetw::parser::Parser;
use ferrisetw::provider::{Provider, TraceFlags};
use ferrisetw::trace::{TraceError, TraceTrait, UserTrace};
use ferrisetw::{EventRecord, SchemaLocator};

// ferrisetw's TraceError does not implement std::error::Error, so wrap it in the
// crate's own error type (or return it directly) rather than Box<dyn Error>.
fn start() -> Result<(), TraceError> {
    let file_provider = Provider::by_guid(KERNEL_FILE_GUID)
        .any(0xFFFF) // keyword mask: see `logman query providers Microsoft-Windows-Kernel-File`
        .trace_flags(TraceFlags::EVENT_ENABLE_PROPERTY_STACK_TRACE) // per-event call stack
        .add_callback(|record: &EventRecord, locator: &SchemaLocator| {
            let Ok(schema) = locator.event_schema(record) else { return };
            let parser = Parser::create(record, &schema);
            let _file_object: Option<u64> = parser.try_parse("FileObject").ok();
            let _pid = record.process_id();
            for item in record.extended_data() {
                if let ExtendedDataItem::StackTrace64(stack) = item.to_extended_data_item() {
                    // Resolve each return address against the kernel module table
                    // (see Driver attribution below).
                    let _frames: &[u64] = stack.addresses();
                }
            }
            // Construct SentinelEvent, send to channel
        })
        .build(); // returns Provider, not Result

    let (trace, handle) = UserTrace::new()
        .named("SentinelTrace".to_string())
        .enable(file_provider)
        // .enable(process_provider).enable(registry_provider) ...
        .start()?;
    // ProcessTrace blocks: run it on a dedicated thread; call trace.stop() on shutdown.
    std::thread::spawn(move || UserTrace::process_from_handle(handle));
    // ...
    trace.stop()
}
```

**Kernel vs. User Trace Sessions**

The manifest-based Kernel-Process, Kernel-File, Kernel-Registry, Kernel-Network, and Kernel-Audit-API-Calls providers work in an ordinary real-time `UserTrace` session (as above). The legacy "classic" kernel events (`EVENT_TRACE_FLAG_*`) need a `KernelTrace`, which is useful for the image-load rundown that populates the driver table. Both require administrator privileges.

```rust
// Type-checked against ferrisetw 1.2.0 for x86_64-pc-windows-msvc.
use ferrisetw::provider::{kernel_providers, Provider};
use ferrisetw::trace::{KernelTrace, TraceError};

fn start_kernel() -> Result<(), TraceError> {
    let image_load = Provider::kernel(&kernel_providers::IMAGE_LOAD_PROVIDER)
        .add_callback(|record, locator| {
            // Image loads with ProcessId 0/4 are kernel drivers: record name, base, size.
        })
        .build();

    let (trace, _handle) = KernelTrace::new()
        .named("SentinelKernelTrace".to_string())
        .enable(image_load)
        .start()?;
    // ...
    trace.stop()
}
```

#### Enrichment Layer

```rust
/// Enriches raw parsed events with process metadata and driver identification.
pub struct Enricher {
    /// Cache of PID → ProcessInfo (refreshed periodically)
    process_cache: HashMap<Pid, ProcessInfo>,
    /// Cache of driver load addresses → DriverInfo
    driver_cache: HashMap<u64, DriverInfo>,
    /// Known anti-cheat profiles
    profiles: Vec<DriverProfile>,
}

impl Enricher {
    pub fn new(profiles: Vec<DriverProfile>) -> Self;

    /// Enrich a raw event with process name, path, publisher, and driver attribution.
    pub fn enrich(&mut self, event: &mut SentinelEvent);

    /// Refresh the process cache (call periodically, e.g., every 5 seconds).
    pub fn refresh_process_cache(&mut self);

    /// Update driver cache when a DriverLoad event is received.
    pub fn register_driver_load(&mut self, driver: &DriverInfo);

    /// Match a driver against known profiles.
    pub fn match_profile(&self, driver_name: &str) -> Option<&DriverProfile>;
}
```

**Process resolution:**
```rust
// Use windows crate:
// 1. OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, pid)
// 2. QueryFullProcessImageNameW → full exe path
// 3. GetProcessId → verify PID
// 4. WTSGetActiveConsoleSessionId → session attribution
//
// For publisher/digital signature:
// WinVerifyTrust or WTHelperGetProvSignerFromChain (expensive — cache aggressively)
//
// Cache strategy:
// - HashMap<Pid, ProcessInfo> with lazy population
// - On cache miss: resolve and cache
// - Periodic full refresh every 5 seconds to catch new processes
// - Remove entries for terminated PIDs
```

**Driver attribution:**
ETW file/registry events report the PID of the thread that issued the operation, not the driver. Kernel code running on a system worker thread shows up as PID 4 (System); kernel code running in another thread's context (e.g., inside a callback) shows up as *that* process's PID. To attribute operations to specific drivers:

1. Build a kernel module table (name, base, size) from kernel `ImageLoad` events plus a startup rundown (`EnumDeviceDrivers` / `NtQuerySystemInformation(SystemModuleInformation)`; both return real addresses only to elevated callers).
2. Enable stack traces on the file and registry providers (`ProviderConfig::stack_trace`). Each event then carries a `StackTrace64` extended item.
3. Walk the stack from the top. The first frame inside a module that isn't `ntoskrnl.exe`, `fltmgr.sys`, a file system, or another known I/O-stack driver identifies the originating driver.
4. If no frame resolves to a third-party module, store the event with `driver: None`. Do **not** guess from timing: a report that attributes events by correlation will be wrong, and wrong in a way readers will treat as evidence.

Limits that reports must state:

- **Memory reads are not observable.** A driver reading another process's memory (`MmCopyVirtualMemory`, `KeStackAttachProcess`, physical-memory mapping) emits no ETW event that a non-PPL consumer can receive. The minifilter cannot see it either.
- **Code outside a loaded image is unattributable.** Obfuscated drivers may run code from allocated pool memory; those frames resolve to no module.
- **Stack walking has a cost.** Budget for it against the <2% CPU target. Capture stacks only on the event IDs you attribute, and set `DisablePagingExecutive=1` so x64 kernel stacks walk reliably.

The minifilter driver (Phase 7) adds attribution for file I/O that ETW misses, using the same stack-walk technique. It is not definitive either, and it does not cover memory reads.

#### Session Build Plan

**Session 4 — ETW session setup:**
- Add `ferrisetw` (or raw `windows` crate ETW APIs) as dependency
- Create a trace session that subscribes to the kernel process provider
- Print raw events to stdout as proof of concept
- Goal: run the binary as admin, see process create/terminate events flowing

**Session 5 — Multi-provider + parser:**
- Enable all kernel providers (process, file, registry, network)
- Parse raw ETW events into `SentinelEvent` structs
- Handle each event type's specific fields
- Goal: structured SentinelEvent objects for all event categories

**Session 6 — Enrichment:**
- Build the process cache (PID → name, path, publisher)
- Build the driver catalog (track ImageLoad events for drivers)
- Profile matching (load TOML profiles, match against driver names)
- "Outside expected directory" tagging for file events
- Goal: enriched events with human-readable process/driver names

**Session 7 — Pipeline integration:**
- Wire up: ETW consumer → enrichment → batch → store insertion
- Channel-based architecture: ETW thread → mpsc channel → writer thread
- Batch events (100-500 per write) for database performance
- Driver session tracking (create/update sessions on load/unload)
- Goal: events flowing from kernel into SQLite continuously

### 2.2 — sentinel-service

The Windows service wrapper around the pipeline.

```rust
/// Service configuration loaded from %ProgramData%\Sentinel\config.toml.
/// (Not %APPDATA%: the service runs as SYSTEM, whose %APPDATA% is under
/// C:\Windows\System32\config\systemprofile and invisible to the user.)
/// Every section and field is optional; missing ones take the defaults below.
#[derive(Debug, Default, Serialize, Deserialize)]
#[serde(default)]
pub struct ServiceConfig {
    pub service: ServiceSection,
    pub etw: EtwConfig,
    pub storage: StorageConfig,
    pub pipeline: PipelineConfig,
    pub alerts: AlertConfig,
    pub ipc: IpcConfig,
    pub profiles: ProfilesConfig,
    pub vulndb: VulnDbConfig,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct ServiceSection {
    /// trace | debug | info | warn | error
    pub log_level: String,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct StorageConfig {
    /// Path to database. Default: %ProgramData%\Sentinel\data\sentinel.db
    pub db_path: Option<String>,
    /// Days to retain events before rotation.
    pub retention_days: u32,
    /// Run rotation at this hour (0-23, local time).
    pub rotation_hour: u32,
    /// Maximum database size in MB (0 = unlimited).
    pub max_db_size_mb: u64,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct PipelineConfig {
    /// Events per database write.
    pub batch_size: usize,
    /// Max time before flushing an incomplete batch.
    pub batch_timeout_ms: u64,
    pub process_cache_refresh_seconds: u64,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct AlertConfig {
    pub enabled: bool,
    pub rules_from_db: bool,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct IpcConfig {
    /// Named pipe name for CLI/dashboard communication.
    pub pipe_name: String,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct ProfilesConfig {
    /// Directory containing .toml driver profiles.
    pub directory: String,
    /// Auto-download community profiles from GitHub (future).
    pub auto_update: bool,
}

/// Known-vulnerable driver check (Phase 5).
#[derive(Debug, Serialize, Deserialize)]
#[serde(default)]
pub struct VulnDbConfig {
    pub enabled: bool,
    /// Where the lists live, relative to %ProgramData%\Sentinel.
    pub directory: String,
    pub scan_interval_hours: u32,
    /// Searched recursively for *.sys. Environment variables are expanded.
    pub scan_directories: Vec<String>,
    /// Also check every registered kernel / file-system driver service.
    pub include_services: bool,
    /// Warn when a list is older than this.
    pub stale_after_days: u32,
}

impl Default for ServiceSection {
    fn default() -> Self { Self { log_level: "info".to_string() } }
}

impl Default for StorageConfig {
    fn default() -> Self {
        Self { db_path: None, retention_days: 30, rotation_hour: 3, max_db_size_mb: 0 }
    }
}

impl Default for PipelineConfig {
    fn default() -> Self {
        Self { batch_size: 200, batch_timeout_ms: 1000, process_cache_refresh_seconds: 5 }
    }
}

impl Default for AlertConfig {
    fn default() -> Self { Self { enabled: true, rules_from_db: true } }
}

impl Default for IpcConfig {
    fn default() -> Self { Self { pipe_name: r"\\.\pipe\sentinel-service".to_string() } }
}

impl Default for ProfilesConfig {
    fn default() -> Self { Self { directory: "profiles".to_string(), auto_update: false } }
}

impl Default for VulnDbConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            directory: "vulndb".to_string(),
            scan_interval_hours: 24,
            // The directories LOLDrivers recommends scanning.
            scan_directories: vec![
                r"%SystemRoot%\System32\drivers".to_string(),
                r"%SystemRoot%\System32\DriverStore\FileRepository".to_string(),
                r"%SystemRoot%\inf".to_string(),
            ],
            include_services: true,
            stale_after_days: 30,
        }
    }
}
```

#### Service Architecture

```
┌─────────────────────────────────────────┐
│           sentinel-service.exe           │
│                                          │
│  ┌──────────────┐   ┌────────────────┐  │
│  │ ETW Consumer │──▶│ Event Channel  │  │
│  │ (thread 1)   │   │ (mpsc, bounded)│  │
│  └──────────────┘   └───────┬────────┘  │
│                              │           │
│                     ┌────────▼────────┐  │
│                     │ Pipeline Thread │  │
│                     │                 │  │
│                     │ 1. Dequeue batch│  │
│                     │ 2. Enrich       │  │
│                     │ 3. Alert check  │  │
│                     │ 4. Store write  │  │
│                     │ 5. Broadcast to │  │
│                     │    subscribers  │  │
│                     └────────┬────────┘  │
│                              │           │
│  ┌──────────────┐   ┌───────▼────────┐  │
│  │ IPC Server   │◀──│ SQLite Store   │  │
│  │ (named pipe) │   │ (WAL mode)     │  │
│  │ CLI/dashboard│   └────────────────┘  │
│  │ connect here │                        │
│  └──────────────┘                        │
│                                          │
│  ┌──────────────┐                        │
│  │ Maintenance  │  Rotation, vacuum,     │
│  │ (timer)      │  process cache refresh │
│  └──────────────┘                        │
└──────────────────────────────────────────┘
```

#### Windows Service Registration

```rust
// Use the `windows-service` crate for SCM integration
// Or implement manually with windows crate:
//   RegisterServiceCtrlHandlerExW
//   SetServiceStatus
//   SERVICE_STATUS with SERVICE_RUNNING, SERVICE_STOPPED, etc.
//
// The service binary supports two modes:
//   sentinel-service.exe --service    # Called by SCM
//   sentinel-service.exe --console    # Run in foreground for debugging
```

#### Session Build Plan

**Session 8 — Service skeleton:**
- Windows Service wrapper (start, stop, status)
- Config file loading
- Named pipe IPC server (reuse pattern from sovereign-shell shared/ipc)
- Console mode for debugging
- Goal: `sentinel-service --console` runs and accepts pipe connections

**Session 9 — Full pipeline:**
- Wire ETW consumer → channel → enrichment → store → alert
- IPC handlers for Status, Query, ListDrivers
- Live event subscription (clients can subscribe to filtered event stream)
- Goal: service runs, collects events, queryable via pipe

**Session 10 — Service lifecycle:**
- Install/uninstall as Windows Service via SCM
- Auto-start on boot (delayed auto-start recommended)
- Graceful shutdown on service stop signal
- Log rotation on schedule
- install-service.ps1 / uninstall-service.ps1 scripts
- Goal: fully managed Windows Service that survives reboots

---

## Phase 3 — CLI Tool

**Sessions:** 2–3  
**Goal:** Complete command-line interface for querying, reporting, and managing Sentinel.

```toml
# bins/sentinel-cli/Cargo.toml
[package]
name = "sentinel-cli"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "sentinel"
path = "src/main.rs"

[dependencies]
sentinel-types = { path = "../../crates/sentinel-types" }
clap = { version = "4", features = ["derive"] }
serde_json = "1"
chrono = "0.4"
colored = "2"
tabled = "0.15"     # Pretty table output
indicatif = "0.17"  # Progress bars
```

#### CLI Command Structure

```
sentinel
├── install [--driver]              # Install service (and optionally the minifilter)
├── uninstall [--driver] [--purge]  # Uninstall service (--purge removes data)
├── start                           # Start the service
├── stop                            # Stop the service
├── status                          # Show service status + active drivers
│
├── watch                           # Live event stream to terminal
│   ├── --driver <name>             # Filter by driver
│   ├── --events <type,...>         # Filter by event type
│   ├── --process <name>            # Filter by source process
│   └── --target <pattern>          # Filter by target path/IP
│
├── query                           # Query historical events
│   ├── files                       # File access events
│   ├── processes                   # Process access events
│   ├── registry                    # Registry events
│   ├── network                     # Network events
│   └── drivers                     # Driver load/unload events
│   Each subcommand supports:
│   ├── --driver <name>
│   ├── --after <datetime>
│   ├── --before <datetime>
│   ├── --today
│   ├── --days <n>
│   ├── --exclude <pattern>
│   ├── --sort-by <field>
│   ├── --limit <n>
│   └── --format text|json|csv
│
├── report                          # Generate audit report
│   ├── --driver <name>             # Report for specific driver
│   ├── --category anticheat        # Report for all anti-cheat drivers
│   ├── --today | --days <n>
│   └── --format text|json|html
│
├── diff                            # Compare two time periods
│   ├── --driver <name>
│   ├── --before <date>
│   └── --after <date>
│
├── drivers                         # List all known drivers
│   ├── --active                    # Currently loaded only
│   ├── --anticheat                 # Known anti-cheat only
│   └── --vulnerable                # Known-vulnerable/malicious drivers (Phase 5)
│       ├── --rescan                # Run a full scan first
│       └── --all                   # Include resolved findings
│
├── vulndb                          # Vulnerable-driver lists (Phase 5)
│   ├── update [--source loldrivers|microsoft]   # Download, validate, reload (elevated)
│   └── status                      # List ages, entry counts, blocklist on/off
│
├── alerts                          # Manage alert rules
│   ├── list
│   ├── add <rule-file.toml>
│   ├── remove <rule-id>
│   └── test <rule-id>              # Test rule against historical data
│
└── dashboard                       # Launch the Tauri dashboard
```

#### Terminal Output Formatting

Sample output is illustrative. Names and values are invented to show formatting, not observations of any product.

```
$ sentinel status

  Sentinel Service          RUNNING (uptime: 4h 32m)
  Events collected today:   14,832
  Events total:             1,247,553
  Database size:            342 MB
  Known-vulnerable drivers: 1 (run `sentinel drivers --vulnerable`)

  Active Kernel Drivers:
  ┌──────────────────────┬────────────────┬───────────┬──────────┐
  │ Driver               │ Publisher      │ Profile   │ Events/h │
  ├──────────────────────┼────────────────┼───────────┼──────────┤
  │ vgk.sys              │ Riot Games     │ Vanguard  │ 1,247    │
  │ EasyAntiCheat.sys    │ Epic Games     │ EAC       │ 342      │
  │ nvlddmkm.sys         │ NVIDIA         │ —         │ 89       │
  └──────────────────────┴────────────────┴───────────┴──────────┘

$ sentinel watch --process vgc.exe --events process,network

  14:22:01.334  PROCESS_ACCESS  vgc.exe → chrome.exe (PID 4521)  OPEN VM_READ|QUERY_INFO
  14:22:01.412  PROCESS_ACCESS  vgc.exe → discord.exe (PID 8832) OPEN VM_READ|QUERY_INFO
  14:22:02.001  NETWORK_SEND    vgc.exe → 104.18.23.2:443        1.2 KB
  14:22:03.118  PROCESS_ACCESS  vgc.exe → explorer.exe (PID 2104) OPEN QUERY_LIMITED_INFO
  ^C (Ctrl+C to stop)

$ sentinel query files --driver vgk.sys --exclude "C:\Riot Games\*" --today

  Found 23 file accesses outside game directory:

  ┌──────────────┬────────────────────────────────────────────────────┬──────┐
  │ Time         │ Target                                             │ Op   │
  ├──────────────┼────────────────────────────────────────────────────┼──────┤
  │ 14:22:04.221 │ %APPDATA%\Discord\Local Storage\leveldb\*.log     │ READ │
  │ 14:22:04.445 │ %USERPROFILE%\Documents\taxes_2025.pdf            │ ENUM │
  │ 14:23:11.003 │ C:\Program Files\Google\Chrome\Application\*.dll  │ READ │
  │ ...                                                                      │
  └──────────────┴────────────────────────────────────────────────────┴──────┘
```

#### Session Build Plan

**Session 11 — CLI skeleton + status/watch:**
- Set up clap argument parser with all subcommands
- Implement `sentinel status` (connect to service pipe, display status)
- Implement `sentinel watch` (subscribe to live stream, format and print)
- Goal: running service + `sentinel status` shows real data

**Session 12 — Query + report:**
- Implement all `sentinel query` subcommands
- Implement `sentinel report` with text output
- Implement `sentinel diff`
- Implement `sentinel drivers`
- Goal: full historical query capability from the command line

**Session 13 — Install/uninstall + polish:**
- Implement `sentinel install` / `sentinel uninstall`
- JSON and CSV output formats
- Colored terminal output
- Error handling and user-friendly messages
- Goal: complete CLI tool

---

## Phase 4 — Alert Engine

**Sessions:** 1–2  
**Goal:** Rule-based real-time alerting on events.

### sentinel-alert

```rust
pub struct AlertEngine {
    rules: Vec<AlertRule>,
    /// Recent event window for threshold-based rules.
    window: HashMap<String, VecDeque<DateTime<Utc>>>,
}

impl AlertEngine {
    pub fn new(rules: Vec<AlertRule>) -> Self;

    /// Evaluate an event against all active rules. Returns triggered alerts.
    pub fn evaluate(&mut self, event: &SentinelEvent) -> Vec<TriggeredAlert>;

    /// Evaluate a vulnerable-driver finding (Phase 5) against VulnerableDriver rules.
    /// Called by the scanner when a finding is created or becomes loaded.
    pub fn evaluate_finding(
        &mut self,
        finding: &VulnerableDriverFinding,
        change: FindingChange,
    ) -> Vec<TriggeredAlert>;

    /// Update rules (called when user modifies rules via CLI/dashboard).
    pub fn update_rules(&mut self, rules: Vec<AlertRule>);
}

pub enum FindingChange {
    /// First time this (path, sha256) was seen.
    New,
    /// An existing finding's driver was loaded.
    NowLoaded,
}

pub struct TriggeredAlert {
    pub rule: AlertRule,
    /// What triggered it. Most rules fire on events; VulnerableDriver fires on findings.
    pub subject: AlertSubject,
    pub timestamp: DateTime<Utc>,
    pub message: String,
}

pub enum AlertSubject {
    Event(SentinelEvent),
    VulnerableDriver(VulnerableDriverFinding),
}
```

#### Rule Matching Logic

```rust
fn matches_condition(event: &SentinelEvent, condition: &AlertCondition) -> bool {
    match condition {
        AlertCondition::FileAccess { driver_pattern, path_pattern } => {
            event.event_type.category() == EventCategory::File
                && driver_matches(&event.driver, driver_pattern)
                && glob_match(&event.target, path_pattern)
        }
        AlertCondition::ProcessAccess { driver_pattern, process_pattern } => {
            event.event_type == EventType::ProcessAccess
                && driver_matches(&event.driver, driver_pattern)
                && matches_event_detail_target_name(event, process_pattern)
        }
        AlertCondition::NetworkThreshold { driver_pattern, bytes_per_minute } => {
            // Accumulate in sliding window, trigger when threshold exceeded
            // (stateful — uses self.window)
            false // handled separately in evaluate()
        }
        AlertCondition::PersistenceAfterExit { .. } => {
            // Requires cross-referencing driver sessions with game process lifecycle
            // Checked periodically, not per-event
            false
        }
        AlertCondition::NewDriverLoad => {
            event.event_type == EventType::DriverLoad
        }
        AlertCondition::VulnerableDriver { .. } => {
            // Driven by scanner findings, not events: see evaluate_finding()
            false
        }
    }
}
```

#### Notification Dispatch

```rust
pub fn dispatch(alert: &TriggeredAlert, action: &AlertAction) {
    match action {
        AlertAction::Log => {
            // Write to sentinel alert log
            log::warn!("ALERT: {}", alert.message);
        }
        AlertAction::Toast { title, body } => {
            // Windows toast notification via windows crate
            // PowerShell fallback: New-BurntToastNotification
            show_toast(title, &alert.message);
        }
        AlertAction::Webhook { url } => {
            // POST JSON to webhook URL
            // Fire-and-forget, don't block the pipeline
            post_webhook(url, alert);
        }
    }
}
```

**Session 14 — Alert engine:**
- Implement rule matching for all condition types
- Wire into the pipeline (after enrichment, before store write)
- Log and toast notification dispatch
- Goal: create a rule "alert when vgk.sys reads files in Documents", trigger it

**Session 15 — Alert management:**
- Persist rules to database
- CLI: `sentinel alerts list/add/remove/test`
- IPC handlers for GetAlertRules / SetAlertRules
- Goal: manage alert rules from CLI, persist across restarts

---

## Phase 5 — Vulnerable Driver Check

**Sessions:** 3  
**Goal:** Flag drivers on this machine (loaded, registered as a service, or on disk) that appear on public lists of vulnerable or malicious drivers.

A legitimately signed driver with an exploitable flaw lets anyone with admin rights run code in the kernel ("bring your own vulnerable driver"). Anti-cheat drivers have been used this way: in 2022 ransomware operators abused Genshin Impact's `mhyprot2.sys` to kill antivirus software; the game didn't need to be installed (Trend Micro). Unlike behavior monitoring, this check is exact (hash matches), cheap, and sees everything it claims to.

The core crate depends only on Phase 1 and builds and tests on any OS. It sits here because it plugs into the service (Phase 2), the CLI (Phase 3), and the alert engine (Phase 4).

### Lists

| List | Contents | How Sentinel gets it | Matched on |
|------|----------|----------------------|------------|
| [LOLDrivers](https://www.loldrivers.io/) | ~700 drivers, ~2,400 samples, vulnerable and malicious (2026). Apache-2.0. | Snapshot bundled with the installer, with LOLDrivers' license notice. `sentinel vulndb update` fetches `https://www.loldrivers.io/api/drivers.json`. | SHA-1, SHA-256, Authenticode SHA-1/SHA-256 |
| Microsoft Vulnerable Driver Blocklist | The App Control (WDAC) policy Windows uses to refuse known-bad drivers | Not bundled (redistribution terms unclear). `sentinel vulndb update` fetches `https://aka.ms/VulnerableDriverBlockList` (a zip; use the `.xml` whose root element is `SiPolicy`). | Authenticode SHA-1/SHA-256 hash rules only |

**Authenticode hashes** cover the PE image minus its checksum and signature, so they don't change when a driver is re-signed or its signature is stripped. Both lists are built around them. Flat file hashes only match the exact file.

**Coverage limit, stated in every report:** the Microsoft blocklist's signer + file-version rules (`<FileAttrib>` referenced from a `<Signer>`) are not evaluated in v1. Hash rules and LOLDrivers cover most listed drivers, but not every version range Microsoft blocks.

When Microsoft's blocklist is on, Windows refuses to load drivers on it. So a *loaded* driver matched by a Microsoft hash rule means the blocklist is off or the machine's copy predates that rule. Reports show the blocklist setting so the reader can tell which.

### What Gets Checked

| Where | How | When |
|-------|-----|------|
| Loaded drivers | `EnumDeviceDrivers` + `GetDeviceDriverFileNameW` | Every full scan, and on every `DriverLoad` event (Phase 2) |
| Registered driver services | `HKLM\SYSTEM\CurrentControlSet\Services\*` with `Type` 1 (kernel) or 2 (file system); `ImagePath`, default `System32\drivers\<name>.sys` | Every full scan |
| Driver directories | `*.sys` under `scan_directories`, by default the three LOLDrivers recommends: `System32\drivers`, `System32\DriverStore\FileRepository`, `inf` | Every full scan (daily by default), at background I/O priority |

A loaded driver whose image file can't be read (deleted after loading, access denied) is reported in `VulnScanSummary::unverifiable_loaded`. Sentinel can't check it, and deleting a driver's file after loading it is a known attacker move.

Not covered: a driver file elsewhere on disk that is neither registered nor loaded. Once it's registered or loaded, it's checked.

### 5.1 — sentinel-vulndb

```toml
# crates/sentinel-vulndb/Cargo.toml
[package]
name = "sentinel-vulndb"
version = "0.1.0"
edition = "2021"

[dependencies]
sentinel-types = { path = "../sentinel-types" }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["serde"] }
sha1 = "0.10"
sha2 = "0.10"
# authenticode 0.6 pins object 0.39; keep these in step so PeFile implements PeTrait.
authenticode = { version = "0.6", features = ["object"] }
object = { version = "0.39", default-features = false, features = ["read_core", "pe"] }
roxmltree = "0.20"

[target.'cfg(windows)'.dependencies]
windows = { version = "0.58", features = [
    "Win32_Foundation",
    "Win32_Storage_FileSystem",
    "Win32_System_ProcessStatus",
    "Win32_System_Registry",
    "Win32_System_Threading",
] }
```

The code blocks below for `lib.rs`, `hashing.rs`, `loldrivers.rs`, `ms_blocklist.rs`, and `paths.rs` compile as written and pass the Session 16 tests.

```rust
// crates/sentinel-vulndb/src/lib.rs
pub mod hashing;
pub mod loldrivers;
pub mod ms_blocklist;
pub mod paths;
#[cfg(windows)]
pub mod scanner;

use sentinel_types::{DriverHashes, HashKind, VulnMatch};
use std::collections::HashMap;

/// One hash from a list, with the entry it belongs to. `entry.matched_on` is the
/// kind of hash this is.
#[derive(Debug, Clone)]
pub struct ListedHash {
    pub hash: String,
    pub entry: VulnMatch,
}

/// In-memory index over all loaded lists.
#[derive(Default)]
pub struct VulnDb {
    index: HashMap<(HashKind, String), Vec<VulnMatch>>,
}

impl VulnDb {
    pub fn add(&mut self, listed: Vec<ListedHash>) {
        for ListedHash { hash, entry } in listed {
            self.index.entry((entry.matched_on, hash)).or_default().push(entry);
        }
    }

    /// Every list entry matching any of the image's hashes, one per (source, entry_id).
    /// Authenticode matches come first: they're the hash both lists are built around.
    pub fn lookup(&self, hashes: &DriverHashes) -> Vec<VulnMatch> {
        let candidates = [
            (HashKind::AuthentihashSha256, hashes.authentihash_sha256.as_deref()),
            (HashKind::AuthentihashSha1, hashes.authentihash_sha1.as_deref()),
            (HashKind::Sha256, Some(hashes.sha256.as_str())),
            (HashKind::Sha1, Some(hashes.sha1.as_str())),
        ];
        let mut found: Vec<VulnMatch> = Vec::new();
        for (kind, hash) in candidates {
            let Some(hash) = hash else { continue };
            for m in self.index.get(&(kind, hash.to_string())).into_iter().flatten() {
                if !found.iter().any(|f| f.source == m.source && f.entry_id == m.entry_id) {
                    found.push(m.clone());
                }
            }
        }
        found
    }
}
```

```rust
// crates/sentinel-vulndb/src/hashing.rs
use authenticode::PeTrait;
use object::read::pe::{PeFile32, PeFile64};
use sentinel_types::DriverHashes;
use sha1::Sha1;
use sha2::{Digest, Sha256};

/// Hash a driver image: flat file hashes plus Authenticode hashes.
pub fn hash_image(bytes: &[u8]) -> DriverHashes {
    let (authentihash_sha1, authentihash_sha256) = match authentihashes(bytes) {
        Some((sha1, sha256)) => (Some(sha1), Some(sha256)),
        None => (None, None), // not a parseable PE: flat hashes only
    };
    DriverHashes {
        sha1: hex(&Sha1::digest(bytes)),
        sha256: hex(&Sha256::digest(bytes)),
        authentihash_sha1,
        authentihash_sha256,
    }
}

/// Authenticode hashes via the `authenticode` crate (google/authenticode-rs), which
/// follows the PE/COFF spec: headers minus checksum and certificate-table entry,
/// sections in file order, trailing data minus the certificate table.
fn authentihashes(bytes: &[u8]) -> Option<(String, String)> {
    // Drivers are PE32+ (x64, ARM64) or PE32 (x86).
    let pe: Box<dyn PeTrait + '_> = match PeFile64::parse(bytes) {
        Ok(pe) => Box::new(pe),
        Err(_) => Box::new(PeFile32::parse(bytes).ok()?),
    };
    let mut sha1 = Sha1::new();
    authenticode::authenticode_digest(pe.as_ref(), &mut sha1).ok()?;
    let mut sha256 = Sha256::new();
    authenticode::authenticode_digest(pe.as_ref(), &mut sha256).ok()?;
    Some((hex(&sha1.finalize()), hex(&sha256.finalize())))
}

fn hex(bytes: &[u8]) -> String {
    bytes.iter().map(|b| format!("{b:02x}")).collect()
}

/// Normalize a hash from a list: lowercase hex of the expected length, or None for
/// placeholders ("", "-") and malformed values.
pub fn normalize_hash(s: &str, hex_len: usize) -> Option<String> {
    let s = s.trim();
    (s.len() == hex_len && s.bytes().all(|b| b.is_ascii_hexdigit())).then(|| s.to_ascii_lowercase())
}
```

```rust
// crates/sentinel-vulndb/src/loldrivers.rs
//
// Parses https://www.loldrivers.io/api/drivers.json: an array of driver entries whose
// field names follow the LOLDrivers YAML template (YML-Template.yml in the LOLDrivers
// repo). Every field is optional here; the list is community-edited and a malformed
// entry must not take the whole list down.
use crate::hashing::normalize_hash;
use crate::ListedHash;
use sentinel_types::{HashKind, VulnCategory, VulnMatch, VulnSource};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
#[serde(rename_all = "PascalCase")]
struct LolDriver {
    id: String,
    #[serde(default)]
    category: Option<String>, // "vulnerable driver" | "malicious"
    #[serde(default, rename = "CVE")]
    cve: Option<Vec<String>>,
    #[serde(default)]
    resources: Option<Vec<String>>,
    #[serde(default)]
    known_vulnerable_samples: Option<Vec<LolSample>>,
}

#[derive(Debug, Deserialize)]
struct LolSample {
    #[serde(default, rename = "Filename")]
    filename: Option<String>,
    #[serde(default, rename = "SHA1")]
    sha1: Option<String>,
    #[serde(default, rename = "SHA256")]
    sha256: Option<String>,
    #[serde(default, rename = "Authentihash")]
    authentihash: Option<LolAuthentihash>,
}

#[derive(Debug, Deserialize)]
struct LolAuthentihash {
    #[serde(default, rename = "SHA1")]
    sha1: Option<String>,
    #[serde(default, rename = "SHA256")]
    sha256: Option<String>,
}

/// Returns every listed hash, plus the number of entries that failed to parse.
pub fn parse(json: &str) -> serde_json::Result<(Vec<ListedHash>, u64)> {
    let raw: Vec<serde_json::Value> = serde_json::from_str(json)?;
    let (mut out, mut skipped) = (Vec::new(), 0);
    for value in raw {
        let Ok(driver) = serde_json::from_value::<LolDriver>(value) else {
            skipped += 1;
            continue;
        };
        let category = match driver.category.as_deref() {
            Some(c) if c.to_ascii_lowercase().contains("malicious") => VulnCategory::Malicious,
            _ => VulnCategory::Vulnerable,
        };
        let references: Vec<String> =
            driver.cve.into_iter().chain(driver.resources).flatten().collect();
        for sample in driver.known_vulnerable_samples.into_iter().flatten() {
            let name = sample.filename.clone().unwrap_or_else(|| driver.id.clone());
            let (auth_sha1, auth_sha256) = match &sample.authentihash {
                Some(a) => (a.sha1.as_deref(), a.sha256.as_deref()),
                None => (None, None),
            };
            for (kind, hash, len) in [
                (HashKind::Sha1, sample.sha1.as_deref(), 40),
                (HashKind::Sha256, sample.sha256.as_deref(), 64),
                (HashKind::AuthentihashSha1, auth_sha1, 40),
                (HashKind::AuthentihashSha256, auth_sha256, 64),
            ] {
                let Some(hash) = hash.and_then(|h| normalize_hash(h, len)) else { continue };
                out.push(ListedHash {
                    hash,
                    entry: VulnMatch {
                        source: VulnSource::LolDrivers,
                        entry_id: driver.id.clone(),
                        category,
                        matched_on: kind,
                        name: name.clone(),
                        references: references.clone(),
                    },
                });
            }
        }
    }
    Ok((out, skipped))
}
```

```rust
// crates/sentinel-vulndb/src/ms_blocklist.rs
//
// Microsoft's blocklist is an App Control (WDAC) policy in SiPolicy XML. v1 uses only
// its hash rules:
//   <Deny ID="ID_DENY_FOO_SHA256" FriendlyName="foo.sys Hash Sha256" Hash="ABC..."/>
// Hash values are Authenticode hashes (SHA-1 or SHA-256) or page hashes. Page hashes
// cover only the first page of the image and are skipped. Signer + file-version rules
// (<FileAttrib> referenced from a <Signer>) are NOT evaluated in v1 — report that.
use crate::hashing::normalize_hash;
use crate::ListedHash;
use sentinel_types::{HashKind, VulnCategory, VulnMatch, VulnSource};

const SIPOLICY_NS: &str = "urn:schemas-microsoft-com:sipolicy";

pub fn parse(xml: &str) -> Result<Vec<ListedHash>, roxmltree::Error> {
    let doc = roxmltree::Document::parse(xml)?;
    let mut out = Vec::new();
    for deny in doc.descendants().filter(|n| n.has_tag_name((SIPOLICY_NS, "Deny"))) {
        let (Some(id), Some(hash)) = (deny.attribute("ID"), deny.attribute("Hash")) else {
            continue; // file-name / attribute rules
        };
        let name = deny.attribute("FriendlyName").unwrap_or(id);
        if name.contains("Page") {
            continue;
        }
        let (kind, len) = match hash.len() {
            40 => (HashKind::AuthentihashSha1, 40),
            64 => (HashKind::AuthentihashSha256, 64),
            _ => continue,
        };
        let Some(hash) = normalize_hash(hash, len) else { continue };
        out.push(ListedHash {
            hash,
            entry: VulnMatch {
                source: VulnSource::MicrosoftBlocklist,
                entry_id: id.to_string(),
                category: VulnCategory::Unknown,
                matched_on: kind,
                name: name.to_string(),
                references: Vec::new(),
            },
        });
    }
    Ok(out)
}
```

```rust
// crates/sentinel-vulndb/src/paths.rs

/// Convert the path forms Windows reports for drivers into a Win32 path:
///   \SystemRoot\System32\drivers\x.sys    (GetDeviceDriverFileNameW, ImagePath)
///   \??\C:\Tools\x.sys                    (ImagePath, sc.exe create)
///   \Device\HarddiskVolume3\Windows\...   (ETW image-load events)
///   System32\drivers\x.sys                (relative ImagePath)
///   \Windows\System32\drivers\x.sys       (no drive)
/// `system_root` is e.g. `C:\Windows`. `volumes` maps NT device names to drive
/// letters, e.g. (`\Device\HarddiskVolume3`, `C:`), built from QueryDosDeviceW.
pub fn normalize_driver_path(raw: &str, system_root: &str, volumes: &[(String, String)]) -> String {
    let p = raw.trim().trim_matches('"');
    let p = strip_prefix_ci(p, r"\??\").or_else(|| strip_prefix_ci(p, r"\\?\")).unwrap_or(p);
    if let Some(rest) = strip_prefix_ci(p, r"\SystemRoot\").or_else(|| strip_prefix_ci(p, r"%SystemRoot%\")) {
        return format!(r"{system_root}\{rest}");
    }
    for (device, drive) in volumes {
        if let Some(rest) = strip_prefix_ci(p, device).filter(|r| r.starts_with('\\')) {
            return format!("{drive}{rest}");
        }
    }
    let has_drive = p.len() >= 2 && p.as_bytes()[1] == b':';
    match (has_drive, p.starts_with('\\')) {
        (true, _) => p.to_string(),
        (false, true) => format!("{}{p}", &system_root[..2]), // \Windows\... on the system drive
        (false, false) => format!(r"{system_root}\{p}"),      // relative to %SystemRoot%
    }
}

fn strip_prefix_ci<'a>(s: &'a str, prefix: &str) -> Option<&'a str> {
    let head = s.get(..prefix.len())?;
    head.eq_ignore_ascii_case(prefix).then(|| &s[prefix.len()..])
}
```

**Loading and scanning** (signatures; `scanner.rs` is Windows-only):

```rust
// crates/sentinel-vulndb/src/lib.rs (continued)
impl VulnDb {
    /// Load the lists in `dir` (%ProgramData%\Sentinel\vulndb): loldrivers.json,
    /// microsoft-blocklist.xml, and sources.json (URL, retrieval time, and file SHA-256
    /// per list). A missing list is skipped; a list that fails to parse is an error.
    pub fn load_dir(dir: &Path) -> Result<(VulnDb, Vec<VulnDbSourceInfo>)>;
}

// crates/sentinel-vulndb/src/scanner.rs
pub struct ScanConfig {
    /// Already environment-expanded.
    pub directories: Vec<PathBuf>,
    pub include_services: bool,
}

/// Lets scans reuse hashes. Implemented over the store's file_hash_cache table.
pub trait HashCache {
    fn get(&mut self, path: &str, size: u64, modified: DateTime<Utc>) -> Option<DriverHashes>;
    fn put(&mut self, path: &str, size: u64, modified: DateTime<Utc>, hashes: &DriverHashes);
}

pub struct ScanHit {
    pub path: String,
    pub hashes: DriverHashes,
    pub loaded: bool,
    pub service: Option<DriverService>,
    pub matches: Vec<VulnMatch>,
}

pub struct ScanOutput {
    pub hits: Vec<ScanHit>,
    pub files_checked: u64,
    pub unverifiable_loaded: Vec<String>,
}

/// Full scan of loaded drivers, registered driver services, and *.sys under the
/// configured directories. Paths are normalized, then deduplicated, then hashed once.
/// Call from a thread in background mode (SetThreadPriority with
/// THREAD_MODE_BACKGROUND_BEGIN) so hashing DriverStore doesn't compete with the user.
pub fn full_scan(config: &ScanConfig, db: &VulnDb, cache: &mut dyn HashCache) -> ScanOutput;

/// Check one image after a DriverLoad event. None if the file couldn't be read.
pub fn check_loaded_image(raw_path: &str, db: &VulnDb, cache: &mut dyn HashCache)
    -> Option<ScanHit>;

/// EnumDeviceDrivers + GetDeviceDriverFileNameW, normalized.
pub fn loaded_drivers() -> Vec<String>;

/// Services with Type 1 (kernel driver) or 2 (file system driver), with the
/// normalized ImagePath (default System32\drivers\<service>.sys when absent).
pub fn registered_driver_services() -> Vec<(String, DriverService)>;

/// HKLM\SYSTEM\CurrentControlSet\Control\CI\Config\VulnerableDriverBlocklistEnable:
/// Some(true) for 1, Some(false) for 0, None if absent or unreadable. When absent,
/// Windows' default applies (on by default since Windows 11 22H2); say so, don't guess.
pub fn blocklist_enabled() -> Option<bool>;

/// (\Device\HarddiskVolumeN, C:) pairs for normalize_driver_path, from
/// GetLogicalDriveStringsW + QueryDosDeviceW.
pub fn volume_map() -> Vec<(String, String)>;
```

### Updating the Lists

`sentinel vulndb update` runs in the CLI, never the service, which makes no network requests. It needs an elevated prompt, because `%ProgramData%\Sentinel` is writable only by SYSTEM and Administrators.

1. Download over HTTPS to a temporary file in `%ProgramData%\Sentinel\vulndb`.
2. Parse it with the same parser the service uses. Reject it if parsing fails or yields too few hashes (LOLDrivers: under 1,000; Microsoft: under 100 hash rules). This guards against truncated downloads and HTML error pages.
3. Atomically replace the old file, and record URL, retrieval time, and file SHA-256 in `sources.json`.
4. Send `ReloadVulnDb`. The service rebuilds its index and runs a full scan.

`sentinel vulndb status` and every report show each list's age. Past `stale_after_days`, they warn.

### Service Integration

- **Scanner thread.** Runs a full scan two minutes after service start (to stay out of the way of boot), then every `scan_interval_hours`, and on `VulnerableDrivers { rescan: true }`. After each full scan, it upserts every hit and calls `resolve_vuln_findings_not_seen_since(scan_started)`.
- **DriverLoad events.** The pipeline sends the image path to the scanner thread, never hashing on the ETW callback thread. The thread runs `check_loaded_image`, upserts any hit with `loaded = true`, and evaluates alerts.
- **Alerts.** For a new finding, or an existing one that became loaded, call `AlertEngine::evaluate_finding` with `FindingChange::New` or `NowLoaded`.
- **IPC.** `VulnerableDrivers` returns findings plus the last `VulnScanSummary`. `ReloadVulnDb` reloads from disk. `Status` includes the unresolved finding count.

### Alerting

On first run, the service inserts one default rule:

```rust
AlertRule {
    id: "known-vulnerable-driver".to_string(),
    name: "Known-vulnerable or malicious driver".to_string(),
    enabled: true,
    condition: AlertCondition::VulnerableDriver { loaded_only: false },
    action: AlertAction::Toast {
        title: "Known-vulnerable driver".to_string(),
        body: "Run `sentinel drivers --vulnerable` for details.".to_string(),
    },
}
```

It's the only default rule. Hash matches are exact, so the alert always describes a real file. Whether to act on it is the user's decision.

### CLI Output

Illustrative output; names and values are invented.

```
$ sentinel drivers --vulnerable

  Known-vulnerable drivers (last scan: 2026-03-16 03:00, 4,812 files checked)

  ┌──────────────┬──────────────────────────────────────────┬────────────────────────┬──────────────────────┐
  │ Driver       │ Path                                     │ State                  │ Listed by            │
  ├──────────────┼──────────────────────────────────────────┼────────────────────────┼──────────────────────┤
  │ mhyprot2.sys │ C:\Windows\System32\drivers\mhyprot2.sys │ service (demand start) │ LOLDrivers: vuln.    │
  └──────────────┴──────────────────────────────────────────┴────────────────────────┴──────────────────────┘

  Lists: LOLDrivers (2026-03-01) · Microsoft blocklist (2026-02-11; hash rules only)
  Microsoft Vulnerable Driver Blocklist on this machine: off

  Sentinel doesn't remove or block drivers. To act: update or uninstall the software
  that installed it, or turn on the blocklist (Windows Security → Device security →
  Core isolation).
```

### Reports

Every report begins with a **Known-vulnerable drivers** section, including when nothing was found: "none found (LOLDrivers as of 2026-03-01; Microsoft hash rules as of 2026-02-11; signer/version rules not evaluated)". For each finding, the report gives the path, its state (loaded, the service's start type, or on disk only), which list matched and on which hash, and the list's references.

### Session Build Plan

**Session 16 — Core crate (any OS):**
- `hashing.rs`, `loldrivers.rs`, `ms_blocklist.rs`, `paths.rs`, `VulnDb`, as above
- Authenticode tests: known-answer against an independent implementation of the PE/COFF algorithm on real PE files (x64, x86, ARM64). Then re-sign each one synthetically: append a `WIN_CERTIFICATE`, point the security directory at it, change the checksum. The Authenticode hash must stay the same and the flat hash must change.
- Parser tests on fixtures:
  - LOLDrivers: a malformed entry is skipped, not fatal; `""` and `"-"` placeholders are dropped
  - Microsoft: page-hash and file-name rules are skipped; uppercase hashes are normalized
- Lookup tests: a re-signed copy still matches both lists, once each. A flat-hash-only entry misses the re-signed copy.
- Path tests: every form in `normalize_driver_path`'s doc comment, plus `HarddiskVolume1` vs `HarddiskVolume12`
- Goal: `cargo test -p sentinel-vulndb` passes on Linux and Windows

**Session 17 — Scanner + service (Windows):**
- `scanner.rs`: loaded drivers, registered services, directory walk, volume map, blocklist setting
- Store: the findings and hash-cache tables, `HashCache` over the store
- Scanner thread, DriverLoad hook, finding resolution, IPC handlers, `Status` count
- Known-answer check on Windows: compare one embedded-signed driver's SHA-256 Authenticode hash with the digest `signtool verify /pa /v` prints for it
- Goal: on a test VM, with a test list holding the hashes of a benign inbox driver (e.g. `null.sys`), `sentinel drivers --vulnerable --rescan` reports it as loaded. Never download real vulnerable drivers to test.

**Session 18 — CLI, alerts, reports:**
- `sentinel vulndb update/status` (`ureq`, `zip`), `sentinel drivers --vulnerable`, status line
- `VulnerableDriver` condition, `evaluate_finding`, default rule
- Report section
- Bundle a LOLDrivers snapshot and its license notice in the installer
- Goal: `sentinel vulndb update && sentinel drivers --vulnerable` end to end; the default alert fires for the test-list driver

---

## Phase 6 — Dashboard

**Sessions:** 4–6  
**Goal:** Real-time Tauri UI for monitoring and reporting.

### Dashboard Layout

Mockup data is illustrative, not observations of any product.

```
┌─[Sentinel — Kernel Activity Monitor]──────────────────────────────────┐
│                                                                        │
│  ┌─[Status Bar]──────────────────────────────────────────────────────┐ │
│  │ ● Running  |  Events today: 14,832  |  DB: 342 MB  |  3 drivers  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌─[Tabs]────────────────────────────────────────────────────────────┐ │
│  │ [Drivers]  [Live Stream]  [Network]  [Reports]  [Alerts]         │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌─[Tab: Drivers]────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  vgk.sys (Vanguard)           ████████████████░░░░  1,247 evt/h   │ │
│  │  Riot Games | Boot-start | Active since 14:22                     │ │
│  │  [View Report]  [Watch]  [Set Alert]                              │ │
│  │                                                                    │ │
│  │  EasyAntiCheat.sys (EAC)      ███████░░░░░░░░░░░░  342 evt/h     │ │
│  │  Epic Games | On-demand | Active since 16:05                      │ │
│  │  [View Report]  [Watch]  [Set Alert]                              │ │
│  │                                                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌─[Tab: Live Stream]───────────────────────────────────────────────┐  │
│  │ Filter: [driver ▾] [event type ▾] [target pattern        ] [Go]  │  │
│  │                                                                    │ │
│  │ 14:22:01.334  PROC_ACCESS  vgc.exe → chrome.exe      VM_READ    │ │
│  │ 14:22:01.412  PROC_ACCESS  vgc.exe → discord.exe     VM_READ    │ │
│  │ 14:22:02.001  NET_SEND     vgk.sys → 104.18.23.2     1.2 KB     │ │
│  │ 14:22:03.118  FILE_READ    vgk.sys → Discord\*.log   OUTSIDE    │ │
│  │ ...                                                               │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌─[Tab: Network]───────────────────────────────────────────────────┐  │
│  │ ┌─ Traffic Graph (per driver) ─────────────────────┐              │  │
│  │ │  ▁▂▃▅▇█▇▅▃▂▁▁▂▃▅▇█▇▅▃▂▁  vgk.sys (blue)       │              │  │
│  │ │  ▁▁▂▃▃▂▁▁▁▂▃▃▂▁▁▁▂▃▃▂▁▁  EAC.sys (green)       │              │  │
│  │ └──────────────────────────────────────────────────┘              │  │
│  │                                                                    │ │
│  │  Destinations:                                                    │ │
│  │  104.18.23.2:443    Riot CDN       1.2 MB sent    890 KB recv    │ │
│  │  52.44.128.91:443   AWS us-east-1  340 KB sent    120 KB recv    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

#### Frontend-Backend Communication

The dashboard connects to the sentinel-service via named pipe (same as CLI) but also supports a streaming mode for live events:

```rust
// src-tauri/src/commands.rs

#[tauri::command]
async fn get_status() -> Result<ServiceResponse, String>;

#[tauri::command]
async fn get_drivers(active_only: bool) -> Result<Vec<DriverSession>, String>;

#[tauri::command]
async fn query_events(filter: EventFilter) -> Result<Vec<SentinelEvent>, String>;

#[tauri::command]
async fn generate_report(driver: Option<String>, days: u32) -> Result<String, String>;

#[tauri::command]
async fn get_network_destinations(driver: String) -> Result<Vec<NetworkDestination>, String>;

#[tauri::command]
async fn subscribe_live(filter: EventFilter) -> Result<(), String>;
// Live events are pushed to the frontend via Tauri's event system:
// app_handle.emit("sentinel-event", event)

#[tauri::command]
async fn get_alert_rules() -> Result<Vec<AlertRule>, String>;

#[tauri::command]
async fn set_alert_rules(rules: Vec<AlertRule>) -> Result<(), String>;
```

#### Session Build Plan

**Session 19 — Dashboard scaffold + status:**
- Tauri project setup (matches sovereign-shell theme)
- Connect to service via named pipe
- Status bar with live stats
- Drivers tab: list active drivers with event rate, with a VULNERABLE badge on Phase 5 findings
- Goal: launch dashboard, see active drivers

**Session 20 — Live event stream:**
- Subscribe to filtered event stream from service
- Real-time event table with scrolling
- Filters: driver, event type, target pattern
- Color coding by event category
- Pause/resume stream
- Goal: watch kernel events flowing in real time

**Session 21 — Network tab:**
- Traffic graph per driver (SVG/Canvas line chart)
- Destination table with byte counts
- DNS query log
- Goal: see where drivers are sending data

**Session 22 — Reports + alerts tabs:**
- Report viewer: generate and display text/HTML reports
- Report export (save to file)
- Alert rules editor: list, add, remove, toggle rules
- Alert history viewer
- Goal: full dashboard functionality

**Session 23 — Process heatmap + polish:**
- Process access heatmap: visual grid of which apps are being scanned
- System tray integration
- Auto-start with service option
- Minimize to tray
- Goal: polished, daily-driver monitoring dashboard

---

## Phase 7 — Minifilter Driver (Optional)

**Sessions:** 3–5 (requires Windows Driver Kit)  
**Goal:** Kernel-mode filter driver that attributes file I/O to the originating driver, including I/O that ETW stack traces miss.

### Scope

A minifilter (Filter Manager) only sees **file system** I/O. The other callbacks this driver could add are separate kernel mechanisms, each with its own registration:

| What | Mechanism | Sees |
|------|-----------|------|
| File I/O | Minifilter (`FltRegisterFilter`) | All file system operations, including kernel-mode ones |
| Registry | `CmRegisterCallbackEx` | All registry operations, including kernel-mode ones |
| Process/thread handles | `ObRegisterCallbacks` | Handle opens/duplicates. Drivers that reference processes by pointer (`PsLookupProcessByProcessId`) create no handle and never appear |
| Process memory reads | — | **Nothing.** No supported callback exists for `MmCopyVirtualMemory` / `KeStackAttachProcess` |

Attribution works the same way in all of them: walk the stack in the **originating thread's context** and find the first frame in a third-party module.

### Prerequisites

- Windows Driver Kit (WDK) installed
- Visual Studio with WDK integration
- A minifilter altitude assigned by Microsoft (allocated on request; see "Load order groups and altitudes for minifilter drivers" in the WDK docs). Use a test altitude only in development
- Test signing enabled for development (`bcdedit /set testsigning on`). **Riot Vanguard refuses to run with test signing on (VAN 128)**, and EasyAntiCheat and BattlEye reportedly do too. A test-signed build can be developed against ordinary drivers, but observing an anti-cheat requires the production (attestation-signed) build.
- EV certificate for production signing (see Driver Signing below)

### Driver Architecture

```c
// sentinel-filter.c — Minifilter registration

#include <fltKernel.h>
#include <ntddk.h>

PFLT_FILTER g_Filter = NULL;

// Communication port for user-mode service
PFLT_PORT g_ServerPort = NULL;
PFLT_PORT g_ClientPort = NULL;

// Operation callbacks — observe only, never modify.
// Defined before FilterRegistration, which references it.
const FLT_OPERATION_REGISTRATION Callbacks[] = {
    { IRP_MJ_CREATE,  0, PreCreateCallback,  PostCreateCallback },
    { IRP_MJ_READ,    0, PreReadCallback,    NULL },
    { IRP_MJ_WRITE,   0, PreWriteCallback,   NULL },
    { IRP_MJ_SET_INFORMATION, 0, PreSetInfoCallback, NULL },
    { IRP_MJ_OPERATION_END }
};

// Minifilter registration structure
const FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),
    FLT_REGISTRATION_VERSION,
    0,                          // Flags
    NULL,                       // Context registration
    Callbacks,                  // Operation callbacks
    FilterUnload,               // Unload routine
    InstanceSetup,              // Instance setup
    NULL,                       // Instance query teardown
    NULL,                       // Instance teardown start
    NULL,                       // Instance teardown complete
    NULL, NULL, NULL            // Unused
};
```

**Critical design constraint:** The driver is an observer. It must never interfere with I/O.

- Pre-operation callbacks return `FLT_PREOP_SUCCESS_WITH_CALLBACK` only where a post-op result is needed (create: did the open succeed?), otherwise `FLT_PREOP_SUCCESS_NO_CALLBACK`. Never `FLT_PREOP_COMPLETE` or `FLT_PREOP_PENDING`, never modify parameters or buffers.
- Post-operation callbacks return `FLT_POSTOP_FINISHED_PROCESSING`.
- **Capture attribution in the pre-op callback.** Pre-op runs in the issuing thread for synchronous I/O. Post-op runs in an arbitrary completion context whose stack says nothing about the originator.

### Driver ↔ Service Communication

```c
// Communication port message structure
typedef struct _SENTINEL_MESSAGE {
    ULONG EventType;           // Maps to EventType enum
    LARGE_INTEGER Timestamp;
    ULONG SourcePid;
    ULONG SourceTid;
    ULONG StackDepth;
    ULONG_PTR Stack[16];       // RtlWalkFrameChain/RtlCaptureStackBackTrace in pre-op.
                               // The top frames are fltmgr/ntoskrnl; user mode resolves
                               // the first third-party frame against the module table.
    WCHAR Target[512];         // File path, registry key, etc.
    ULONG DataSize;            // Bytes for read/write operations
} SENTINEL_MESSAGE, *PSENTINEL_MESSAGE;

// Callbacks must NOT call FltSendMessage:
//  - post-op callbacks can run at DISPATCH_LEVEL; FltSendMessage requires
//    IRQL <= APC_LEVEL, so calling it there can bugcheck the machine;
//  - even a 1 ms timeout adds up to 1 ms to every file operation system-wide.
// Instead, callbacks copy the event into a fixed-size ring buffer in nonpaged pool
// (drop and count on overflow; never wait), and a dedicated PASSIVE_LEVEL system
// thread drains it to the client port with FltSendMessage.

VOID QueueEvent(PSENTINEL_MESSAGE message);   // IRQL <= DISPATCH_LEVEL; never blocks
VOID DrainThread(PVOID context);              // PASSIVE_LEVEL; sends batches to g_ClientPort
```

### Driver Signing

```
Development:
  1. Enable test signing: bcdedit /set testsigning on
     (anti-cheat software will refuse to run while this is on — see Prerequisites)
  2. Create self-signed cert (makecert is deprecated):
     New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=Sentinel Dev" -CertStoreLocation Cert:\CurrentUser\My
  3. Sign driver: signtool sign /fd sha256 /n "Sentinel Dev" sentinel-filter.sys

Production:
  1. Request a minifilter altitude from Microsoft
  2. Purchase an EV code signing certificate (keys are issued on a hardware token/HSM)
  3. Register with the Microsoft Partner Center hardware program using the EV cert
  4. Submit the driver package for attestation signing
  5. Microsoft signs the driver; it loads on Windows 10/11 client without test signing
     (Windows Server requires full WHQL certification instead)
```

---

## Phase 8 — Anti-Cheat Profiles

Community-maintained TOML files describing known driver behaviors.

### Profile Schema

```toml
# profiles/vanguard.toml

[driver]
name = "Vanguard"
vendor = "Riot Games"
filenames = ["vgk.sys", "vgc.exe"]
publishers = ["Riot Games, Inc."]
load_behavior = "boot_start"

[games]
titles = ["VALORANT"]
executables = ["VALORANT-Win64-Shipping.exe"]

[expected_behavior]
install_directories = [
    "C:\\Riot Games\\",
    "C:\\Program Files\\Riot Vanguard\\"
]
network_destinations = [
    "*.riotgames.com",
    "*.riotcdn.net",
    "*.pvp.net"
]

# Community-reported behavior. Sentinel cannot verify most of this itself
# (memory reads are not observable), so every claim needs a citation in
# `sources`. Profiles with unsourced claims should not be merged: they are
# statements about a real vendor's product.
[known_activity]
sources = []  # e.g. "https://arxiv.org/abs/2408.00500"
scans_all_processes = true
scans_at_boot = true
persists_after_game_exit = true
reads_browser_memory = true
enumerates_installed_software = true
checks_hypervisor_presence = true
notes = """
Vanguard's vgk.sys loads at boot as a boot-start driver. It runs continuously
regardless of whether VALORANT is open. It scans all running processes for
known cheat signatures and checks for debugging/virtualization tools.
Community reports indicate it reads memory from browser processes and enumerates
installed software via the Uninstall registry key.
"""
```

---

## Configuration File

Default location: `%ProgramData%\Sentinel\config.toml` (all sections optional; see `ServiceConfig`)

```toml
[service]
log_level = "info"          # trace | debug | info | warn | error

[etw]
session_name = "SentinelTrace"
buffer_size_kb = 256
flush_timer_seconds = 1

[storage]
retention_days = 30
rotation_hour = 3           # 3 AM local time
max_db_size_mb = 0          # 0 = unlimited

[pipeline]
batch_size = 200            # Events per database write
batch_timeout_ms = 1000     # Max time before flushing an incomplete batch
process_cache_refresh_seconds = 5

[ipc]
pipe_name = "\\\\.\\pipe\\sentinel-service"

[profiles]
# Directory containing .toml driver profiles
directory = "profiles"
# Auto-download community profiles from GitHub (future)
auto_update = false

[vulndb]
enabled = true
directory = "vulndb"        # Lists live in %ProgramData%\Sentinel\vulndb
scan_interval_hours = 24
scan_directories = [
    "%SystemRoot%\\System32\\drivers",
    "%SystemRoot%\\System32\\DriverStore\\FileRepository",
    "%SystemRoot%\\inf",
]
include_services = true
stale_after_days = 30       # Warn when a list is older than this
```

---

## Rust Crate Preferences

- **ETW:** `ferrisetw` (safe wrapper) — fall back to raw `windows` crate if needed
- **Windows APIs:** `windows` crate (official Microsoft bindings)
- **SQLite:** `rusqlite` with `bundled` feature
- **CLI:** `clap` v4 with derive macros
- **Serialization:** `serde` + `serde_json` + `toml`
- **Time:** `chrono` (with serde feature)
- **Terminal output:** `colored` + `tabled`
- **Logging:** `tracing` + `tracing-subscriber`
- **Hashing:** `sha1` + `sha2` (0.10) + `authenticode` (google/authenticode-rs, with `object`)
- **XML:** `roxmltree` (read-only; Microsoft blocklist)
- **HTTP (CLI only):** `ureq` + `zip` for `sentinel vulndb update`. The service never makes requests.
- **Async (if needed):** `tokio` — but prefer threads + channels for simplicity
- **Desktop UI:** Tauri v2

---

## Testing Strategy

### Unit Tests (every crate)
- Serialization roundtrips for all types
- Database schema creation and CRUD operations
- Event filter matching logic
- Alert rule evaluation
- Report generation from known data
- Vulnerable-driver hashing, list parsing, path normalization, and lookup (Phase 5)

### Integration Tests
- ETW session start/stop without crashing
- Event flow: ETW → parse → enrich → store → query
- Named pipe IPC: service ↔ CLI communication
- Database rotation (insert old events, rotate, verify deletion)

### Manual Testing Checklist
- [ ] Service starts and collects events from kernel
- [ ] `sentinel status` shows accurate counts
- [ ] `sentinel watch` streams events in real time
- [ ] `sentinel query files --driver vgk.sys` returns results
- [ ] `sentinel report` generates accurate audit report
- [ ] Dashboard connects and shows live data
- [ ] Alert triggers toast notification when rule matches
- [ ] `sentinel drivers --vulnerable --rescan` flags a driver from a test list, and the default vulnerable-driver alert fires
- [ ] Service survives 24h continuous operation without memory leak
- [ ] Database rotation runs on schedule
- [ ] Log file rotation works
- [ ] Service restarts cleanly after crash

---

## Performance Targets

- ETW event consumption: handle 10,000+ events/second without dropping
- Database write throughput: 5,000+ events/second (batched)
- Query response time: <200ms for filtered queries over 1M events
- Memory usage: <100 MB steady state (configurable)
- CPU usage: <2% average during normal monitoring
- Dashboard refresh: <100ms for status updates

---

## Security Considerations

- Service runs as SYSTEM (required for kernel ETW sessions)
- Database file permissions: readable only by SYSTEM and Administrators
- Named pipe DACL: accessible by Administrators group only
- No outbound network connections from the service itself (except webhook alerts). Vulnerable-driver lists are downloaded by the CLI, on explicit request, and validated before they replace the old ones
- Vulnerable-driver lists only produce findings and alerts. Sentinel never blocks, unloads, or deletes a driver because of them, so a tampered list can cause false alerts or missed detections, nothing worse
- Driver code is minimal — reduce kernel attack surface
- All user inputs (CLI arguments, IPC messages) validated before use

---

*"Observe everything. Modify nothing. Report what you see."*
