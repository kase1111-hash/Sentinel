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
Phase 5: Dashboard (Tauri UI for real-time monitoring)
Phase 6: Minifilter driver (optional — adds driver-level attribution)
Phase 7: Anti-cheat profile database (community knowledge base)
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
│   └── sentinel-alert/             # Rule engine and notification
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── rules.rs            # Rule definition and matching
│           └── notify.rs           # Notification dispatch (toast, log, webhook)
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
│   │       │   └── diff.rs         # sentinel diff (compare sessions)
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
├── driver/                         # Kernel minifilter (Phase 6 — optional)
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
│   └── DRIVER-SIGNING.md
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
    ProcessAccess,       // Reading another process's memory or handle table
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

/// Anti-cheat driver profile loaded from TOML.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DriverProfile {
    pub name: String,
    pub vendor: String,
    pub driver_filenames: Vec<String>,
    pub known_publishers: Vec<String>,
    pub load_behavior: LoadBehavior,
    pub associated_games: Vec<String>,
    pub expected_directories: Vec<String>,
    pub known_network_destinations: Vec<String>,
    pub notes: String,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum LoadBehavior {
    /// Loads at boot, persists regardless of game state.
    BootStart,
    /// Loads when game starts, should unload when game exits.
    OnDemand,
    /// Loads when game starts, may persist after game exits.
    OnDemandPersistent,
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
    },
    Events { events: Vec<SentinelEvent>, total: u64 },
    Report { content: String, format: ReportFormat },
    Drivers { drivers: Vec<DriverSession> },
    AlertRules { rules: Vec<AlertRule> },
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

    /// Open at the default location: %APPDATA%\Sentinel\data\sentinel.db
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
/// Events: NtOpenProcess, NtReadVirtualMemory, etc.
pub const KERNEL_AUDIT_GUID: &str = "E02A841C-75A3-4FA7-AFC8-AE09CF9B7F23";

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

pub struct ProviderConfig {
    pub guid: String,
    pub level: TraceLevel,
    pub keywords: u64,      // Bitmask to select specific event categories
}

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
                },
                ProviderConfig {
                    guid: KERNEL_FILE_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,  // All file operations
                },
                ProviderConfig {
                    guid: KERNEL_REGISTRY_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
                },
                ProviderConfig {
                    guid: KERNEL_NETWORK_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
                },
                ProviderConfig {
                    guid: DNS_CLIENT_GUID.to_string(),
                    level: TraceLevel::Info,
                    keywords: 0xFFFF,
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
// Example using ferrisetw (preferred approach)
use ferrisetw::provider::Provider;
use ferrisetw::trace::UserTrace;
use ferrisetw::native::etw_types::EventRecord;

let process_provider = Provider::by_guid(KERNEL_PROCESS_GUID)
    .add_callback(|record: &EventRecord, schema| {
        // Parse event fields from schema
        // Construct SentinelEvent
        // Send to channel
    })
    .build()?;

let mut trace = UserTrace::new()
    .named("SentinelTrace".to_string())
    .enable(process_provider)
    .enable(file_provider)
    .enable(registry_provider)
    .enable(network_provider)
    .start()?;
```

**Critical: Kernel vs. User Trace Sessions**

Some kernel providers require a kernel trace session (`EVENT_TRACE_FLAG_*` flags on the system trace session) rather than a user-mode trace session. The Kernel-Process, Kernel-File, Kernel-Registry, and Kernel-Network providers work with `NT Kernel Logger` or with newer system-wide sessions. Requires administrator privileges.

If using `ferrisetw`, use `KernelTrace` for kernel providers:
```rust
use ferrisetw::trace::KernelTrace;
use ferrisetw::native::etw_types::*;

let trace = KernelTrace::new()
    .named("SentinelKernelTrace")
    .enable(EVENT_TRACE_FLAG_PROCESS)      // Process events
    .enable(EVENT_TRACE_FLAG_IMAGE_LOAD)   // Driver/DLL loads
    .enable(EVENT_TRACE_FLAG_DISK_FILE_IO) // File I/O
    .enable(EVENT_TRACE_FLAG_REGISTRY)     // Registry
    .enable(EVENT_TRACE_FLAG_NETWORK_TCPIP)// Network
    .start()?;
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
ETW file/registry events report the source PID, not the source driver. For kernel-mode operations, the PID is typically 4 (System process). To attribute operations to specific drivers:

1. Track `ImageLoad` events for kernel images (image loaded into PID 4 = kernel driver)
2. Maintain a map of driver load addresses to driver names
3. For file/registry events from PID 4, use the call stack (if available via ETW stack walking) to identify which driver initiated the operation
4. If stack attribution isn't available, use heuristics: correlate timing with driver load/unload events and known driver behavior patterns

The minifilter driver (Phase 6) provides definitive attribution by intercepting at the filter manager level.

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
/// Service configuration loaded from %APPDATA%\Sentinel\config.toml
#[derive(Debug, Serialize, Deserialize)]
pub struct ServiceConfig {
    pub etw: EtwConfig,
    pub storage: StorageConfig,
    pub alerts: AlertConfig,
    pub ipc: IpcConfig,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct StorageConfig {
    /// Path to database. Default: %APPDATA%\Sentinel\data\sentinel.db
    pub db_path: Option<String>,
    /// Days to retain events before rotation.
    pub retention_days: u32,
    /// Run rotation at this hour (0-23, local time).
    pub rotation_hour: u32,
    /// Maximum database size in MB (0 = unlimited).
    pub max_db_size_mb: u64,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct AlertConfig {
    pub enabled: bool,
    pub rules_from_db: bool,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct IpcConfig {
    /// Named pipe name for CLI/dashboard communication.
    pub pipe_name: String,
}

impl Default for ServiceConfig {
    fn default() -> Self {
        Self {
            etw: EtwConfig::default(),
            storage: StorageConfig {
                db_path: None,
                retention_days: 30,
                rotation_hour: 3,
                max_db_size_mb: 0,
            },
            alerts: AlertConfig {
                enabled: true,
                rules_from_db: true,
            },
            ipc: IpcConfig {
                pipe_name: r"\\.\pipe\sentinel-service".to_string(),
            },
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
│   └── --anticheat                 # Known anti-cheat only
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

```
$ sentinel status

  Sentinel Service          RUNNING (uptime: 4h 32m)
  Events collected today:   14,832
  Events total:             1,247,553
  Database size:            342 MB

  Active Kernel Drivers:
  ┌──────────────────────┬────────────────┬───────────┬──────────┐
  │ Driver               │ Publisher      │ Profile   │ Events/h │
  ├──────────────────────┼────────────────┼───────────┼──────────┤
  │ vgk.sys              │ Riot Games     │ Vanguard  │ 1,247    │
  │ EasyAntiCheat.sys    │ Epic Games     │ EAC       │ 342      │
  │ nvlddmkm.sys         │ NVIDIA         │ —         │ 89       │
  └──────────────────────┴────────────────┴───────────┴──────────┘

$ sentinel watch --driver vgk.sys --events process,network

  14:22:01.334  PROCESS_ACCESS  vgk.sys → chrome.exe (PID 4521)  READ_MEMORY
  14:22:01.412  PROCESS_ACCESS  vgk.sys → discord.exe (PID 8832) READ_MEMORY
  14:22:02.001  NETWORK_SEND    vgk.sys → 104.18.23.2:443        1.2 KB
  14:22:03.118  PROCESS_ACCESS  vgk.sys → explorer.exe (PID 2104) QUERY_INFO
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

    /// Update rules (called when user modifies rules via CLI/dashboard).
    pub fn update_rules(&mut self, rules: Vec<AlertRule>);
}

pub struct TriggeredAlert {
    pub rule: AlertRule,
    pub event: SentinelEvent,
    pub timestamp: DateTime<Utc>,
    pub message: String,
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

## Phase 5 — Dashboard

**Sessions:** 4–6  
**Goal:** Real-time Tauri UI for monitoring and reporting.

### Dashboard Layout

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
│  │ 14:22:01.334  PROC_ACCESS  vgk.sys → chrome.exe      READ_MEM   │ │
│  │ 14:22:01.412  PROC_ACCESS  vgk.sys → discord.exe     READ_MEM   │ │
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

**Session 16 — Dashboard scaffold + status:**
- Tauri project setup (matches sovereign-shell theme)
- Connect to service via named pipe
- Status bar with live stats
- Drivers tab: list active drivers with event rate
- Goal: launch dashboard, see active drivers

**Session 17 — Live event stream:**
- Subscribe to filtered event stream from service
- Real-time event table with scrolling
- Filters: driver, event type, target pattern
- Color coding by event category
- Pause/resume stream
- Goal: watch kernel events flowing in real time

**Session 18 — Network tab:**
- Traffic graph per driver (SVG/Canvas line chart)
- Destination table with byte counts
- DNS query log
- Goal: see where drivers are sending data

**Session 19 — Reports + alerts tabs:**
- Report viewer: generate and display text/HTML reports
- Report export (save to file)
- Alert rules editor: list, add, remove, toggle rules
- Alert history viewer
- Goal: full dashboard functionality

**Session 20 — Process heatmap + polish:**
- Process access heatmap: visual grid of which apps are being scanned
- System tray integration
- Auto-start with service option
- Minimize to tray
- Goal: polished, daily-driver monitoring dashboard

---

## Phase 6 — Minifilter Driver (Optional)

**Sessions:** 3–5 (requires Windows Driver Kit)  
**Goal:** Kernel-mode filter driver for definitive driver attribution.

### Prerequisites

- Windows Driver Kit (WDK) installed
- Visual Studio with WDK integration
- Test signing enabled for development (`bcdedit /set testsigning on`)
- EV certificate for production signing (Phase 7)

### Driver Architecture

```c
// sentinel-filter.c — Minifilter registration

#include <fltKernel.h>
#include <ntddk.h>

// Communication port for user-mode service
PFLT_PORT g_ServerPort = NULL;
PFLT_PORT g_ClientPort = NULL;

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

// Operation callbacks — observe only, never modify
const FLT_OPERATION_REGISTRATION Callbacks[] = {
    { IRP_MJ_CREATE,  0, PreCreateCallback,  PostCreateCallback },
    { IRP_MJ_READ,    0, NULL,               PostReadCallback },
    { IRP_MJ_WRITE,   0, NULL,               PostWriteCallback },
    { IRP_MJ_SET_INFORMATION, 0, NULL,        PostSetInfoCallback },
    { IRP_MJ_OPERATION_END }
};
```

**Critical design constraint:** Every callback returns `FLT_PREOP_SUCCESS_NO_CALLBACK` or `FLT_POSTOP_FINISHED_PROCESSING`. Never `FLT_PREOP_COMPLETE` (which would block the operation). The driver is an observer — it must never interfere with I/O.

### Driver ↔ Service Communication

```c
// Communication port message structure
typedef struct _SENTINEL_MESSAGE {
    ULONG EventType;           // Maps to EventType enum
    LARGE_INTEGER Timestamp;
    ULONG SourcePid;
    ULONG SourceTid;
    ULONG_PTR DriverAddress;   // Return address on the call stack (identifies driver)
    WCHAR Target[512];         // File path, registry key, etc.
    ULONG DataSize;            // Bytes for read/write operations
} SENTINEL_MESSAGE, *PSENTINEL_MESSAGE;

// Send message to user-mode service (non-blocking)
NTSTATUS SendEvent(PSENTINEL_MESSAGE message) {
    if (g_ClientPort == NULL) return STATUS_PORT_DISCONNECTED;

    LARGE_INTEGER timeout;
    timeout.QuadPart = -10000; // 1ms timeout — drop events rather than block I/O

    return FltSendMessage(
        g_Filter,
        &g_ClientPort,
        message,
        sizeof(SENTINEL_MESSAGE),
        NULL, NULL,
        &timeout
    );
}
```

### Driver Signing

```
Development:
  1. Enable test signing: bcdedit /set testsigning on
  2. Create self-signed cert: makecert -r -pe -n "CN=Sentinel Dev" -ss PrivateCertStore sentinel-dev.cer
  3. Sign driver: signtool sign /s PrivateCertStore /n "Sentinel Dev" sentinel-filter.sys

Production:
  1. Purchase EV code signing certificate (~$300/yr)
  2. Sign driver with EV cert
  3. Submit to Microsoft Hardware Developer Center for attestation signing
  4. Microsoft counter-signs the driver
  5. Driver loads on any Windows machine without test signing
```

---

## Phase 7 — Anti-Cheat Profiles

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

# What Vanguard is known to scan (from community research)
[known_activity]
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

Default location: `%APPDATA%\Sentinel\config.toml`

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
- No outbound network connections from the service itself (except webhook alerts)
- Driver code is minimal — reduce kernel attack surface
- All user inputs (CLI arguments, IPC messages) validated before use

---

*"Observe everything. Modify nothing. Report what you see."*
