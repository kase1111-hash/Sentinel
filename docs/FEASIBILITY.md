# Feasibility Review

**Date:** 2026-10-21  
**Scope:** Does the design in `SPEC.md` work, and is there a real use case? Reviewed before any code exists.

---

## Verdict

**The use case is valid, and nothing else serves it.** No end-user tool audits what anti-cheat software does on a player's machine. The closest prior art is academic reverse engineering, and generic tools like Sysmon log raw events without knowing which ones belong to an anti-cheat.

**ETW-only Sentinel is buildable** with documented, supported Windows APIs. It needs no kernel driver and doesn't touch the game, so the risk of a ban is low.

**The headline claim is not achievable.** "Which processes did Vanguard read memory from" can't be observed by any supported means (details below). The README and spec now say so. Everything else the README asks is answerable, fully or mostly.

**The minifilter (Phase 6) is worth less than the spec implies.** It sees file I/O only, can't see memory reads, and needs a production-signed build to observe anti-cheats, which refuse to run in test-signing mode. Defer it until ETW-only results show a gap worth a signed driver.

---

## Can Sentinel answer the README's questions?

| Question | Answerable? | How | Caveats |
|---|---|---|---|
| Is the anti-cheat still running after I closed the game? | **Yes** | Kernel image-load events + module list, correlated with the game process exiting (profile `games.executables`) | "Loaded" is not "active." A loaded driver may be idle. |
| How much data did EasyAntiCheat send today? | **Yes** (volume, destinations) | Kernel-Network per-process send/receive, DNS-Client for hostnames | Content is encrypted. Telemetry is usually sent by the user-mode service, which is attributable. Telemetry sent by the game process itself can't be separated from game traffic. |
| Did BattlEye access files outside the game directory? | **Mostly** | Kernel-File events with ETW call stacks, first third-party frame → driver | Code running outside a loaded image is unattributable. Needs a baseline: reading system DLLs is normal. |
| What registry keys did it read or modify? | **Mostly** | Kernel-Registry with call stacks | Same as files. |
| What processes did Vanguard inspect? | **Partly** | Kernel-Audit-API-Calls OpenProcess (event 5: target PID, requested access) from the anti-cheat's user-mode service | **Kernel-mode memory reads by `vgk.sys` are not observable.** |

### Why kernel memory reads are out of reach

- ETW has no memory-read event for ordinary consumers. Microsoft-Windows-Threat-Intelligence logs READVM/WRITEVM, but only delivers events to Antimalware-PPL processes, which needs an ELAM driver signed through Microsoft's antimalware program.
- Even that provider instruments the syscall path (`NtReadVirtualMemory`). A driver that looks up the process by pointer and calls `MmCopyVirtualMemory` or `KeStackAttachProcess` likely never hits it.
- `ObRegisterCallbacks` only sees handle opens. Pointer-based access creates no handle.
- Hypervisor-based introspection could see it, but anti-cheats detect and refuse third-party hypervisors and VMs.

The honest product statement is the one the README now makes: Sentinel reports handle opens that *request* memory-read access, and says plainly that it cannot see direct kernel reads.

---

## Use case

**Who it's for:** privacy-minded players, security researchers and journalists, and people who administer shared or family machines.

**What it does best, from most to least solid:**

1. **Lifecycle and persistence.** Which anti-cheat drivers are loaded, since when, and whether they stayed after the game exited or was uninstalled. This is fully observable and cheap, and it's the question players ask most.
2. **Known-abused drivers.** Ransomware abused `mhyprot2.sys` (Genshin Impact) in 2022 to kill antivirus software, and the game didn't need to be installed (Trend Micro, Aug 2022). Flagging loaded or present drivers against the [LOLDrivers](https://www.loldrivers.io/) list is a concrete security win. **Not in the spec yet. Recommend adding it.**
3. **Network accounting** per anti-cheat product: destinations, timing, volume.
4. **Handle-open auditing:** which of your processes the anti-cheat's service asks for memory-read access to.
5. **File and registry attribution** via call stacks. Valuable but carries the most uncertainty; the Phase 2 spike below decides it.

**Prior art:**

- Sysmon (generic logging, covers much of 3–5)
- Process Monitor (ad hoc, no history)
- "If It Looks Like a Rootkit and Deceives Like a Rootkit: A Critical Examination of Kernel-Level Anti-Cheat Systems" (ARES 2024, arXiv:2408.00500), which reverse-engineers Vanguard, EAC, BattlEye, and FACEIT

Nothing found that turns this into an end-user audit trail. Sentinel's niche is Sysmon-class collection plus anti-cheat knowledge plus reports a non-expert can read. The README's FAQ now says this.

**Risks:**

- **Claims about real vendors.** Profiles and reports name real companies. A wrong claim is a credibility problem and potentially a legal one. Profiles now need `sources`, sample output is labeled illustrative, and reports must distinguish "observed 0" from "not observable."
- **Bans.** ETW-only mode loads no driver and doesn't interact with the game. Risk is low but unknown, since vendors don't publish their detection lists. A Sentinel driver raises it.
- **The database is sensitive itself.** It records every file path, process, and DNS lookup on the machine. The 30-day default retention and the SYSTEM/Administrators-only ACLs in the spec matter; keep them.

---

## Recommended scope changes

1. **Build ETW-only first (Phases 1–5).** Treat Phase 6 as conditional.
2. **Start Phase 2 with a spike on real hardware.** Session 4 should run on a Windows machine with Vanguard or EAC installed and answer:
   - Do stack-attributed file and registry events from the anti-cheat driver appear at all?
   - What are the event rate and CPU cost with stacks on? The target is under 2% CPU.
   - Do OpenProcess events (Kernel-Audit-API-Calls, event 5) fire for kernel-mode `ZwOpenProcess` callers? This is undocumented; the event maps to `NtOpenProcess`.
   - Does the anti-cheat react to the trace session?

   Go/no-go on feature 5 above depends on the first two answers.
3. **Attribute by product, not just by `.sys`.** Most observable anti-cheat activity comes from the user-mode service (`vgc.exe` for Vanguard), not the kernel driver. The profile already lists both. Reports and `--driver` filters should roll up by profile.
4. **Add a known-vulnerable-driver check** (LOLDrivers, and Microsoft's vulnerable driver blocklist).
5. **Make coverage explicit in every report:** observed, not observed, and not observable.

---

## Spec bugs found and fixed

Every item marked "verified" was reproduced against the code exactly as written in `SPEC.md`, then re-checked after the fix.

| # | Problem | How verified | Fix |
|---|---|---|---|
| 1 | FTS5 index desyncs on any `UPDATE` to `events` (no update trigger) | SQLite 3.45: `integrity-check` fails with "database disk image is malformed" | Added `events_fts_update` trigger. Integrity holds after update and delete. |
| 2 | Profile TOML (Phase 7) doesn't load into `DriverProfile` (Phase 1): nested tables vs. flat struct; `"boot_start"` rejected by `LoadBehavior` | `toml` crate: "missing field `name`"; serde: "unknown variant `boot_start`" | Struct mirrors the TOML, with `snake_case` enum names. Gained `games.executables` (needed for the persistence alert) and `sources`. |
| 3 | `ServiceConfig` doesn't compile: `EtwConfig` lacks `Debug`/`Serialize`/`Deserialize` | rustc: 5 errors | Added derives. |
| 4 | Sample `config.toml` doesn't parse, and has `[service]`, `[pipeline]`, `[profiles]` sections the struct lacks | `toml` crate: "missing field `providers`" | Every config struct gets `#[serde(default)]` and a `Default`. Added the three missing sections. Sample and empty config both parse. |
| 5 | Kernel-Audit-API-Calls declared but never enabled, so `ProcessAccess` had no source. Its comment claimed it logs `NtReadVirtualMemory`. | Event map from [API-To-ETW](https://github.com/jdu2600/API-To-ETW): IDs 1–8, no memory reads | Added to default providers; corrected the comment; noted why Threat-Intelligence is unusable. |
| 6 | `ferrisetw` snippets don't compile against `ferrisetw` 1.2.0 | `cargo check --target x86_64-pc-windows-msvc`: 9 errors | Rewritten with stack traces enabled. Both snippets type-check as they appear in `SPEC.md`. |
| 7 | `%APPDATA%` used for config and database, but the service runs as SYSTEM, so that resolves under `C:\Windows\System32\config\systemprofile` | Windows service behavior | `%ProgramData%\Sentinel\`. |
| 8 | Minifilter C: `Callbacks` used before it's declared | C declaration rules | Reordered; declared `g_Filter`. |
| 9 | Minifilter rule "every callback returns `NO_CALLBACK`" means `PostCreateCallback` never runs | Filter Manager semantics | Pre-ops return `SUCCESS_WITH_CALLBACK` only where a post-op is needed. |
| 10 | `FltSendMessage` called from callbacks. It requires IRQL ≤ APC_LEVEL; post-ops can run at DISPATCH_LEVEL (possible bugcheck). The 1 ms timeout can add 1 ms to every file operation. | Microsoft docs for `FltSendMessage` and `PFLT_POST_OPERATION_CALLBACK` | Ring buffer in nonpaged pool, drained by a PASSIVE_LEVEL thread. |
| 11 | A single "return address" in a minifilter callback points into `fltmgr.sys`, not the originating driver | Call-stack structure | Capture a 16-frame stack in the pre-op; resolve the first third-party frame. |
| 12 | Minifilter described as covering registry and process operations | Minifilters are file-system-only | Scope table: `CmRegisterCallbackEx`, `ObRegisterCallbacks`, and memory reads (nothing). |
| 13 | Dev path uses test signing, but Vanguard refuses to run with it on (VAN 128); EAC and BattlEye reportedly too | Riot support guidance | Documented in SPEC and README. |
| 14 | Missing altitude step; `makecert` is deprecated | WDK docs | Added both to signing steps. |
| 15 | Ricochet driver listed as `atvi-reven.sys` | Community reports name `atvi-randgrid.sys` | Updated. Confirm on a real install. |
| 16 | EAC listed only as `EasyAntiCheat.sys` | Fortnite uses EAC's EOS build | Added `EasyAntiCheat_EOS.sys`. |
| 17 | Sample output showed `vgk.sys` doing `READ_MEMORY` | Not observable (see above) | Replaced with observable events; samples labeled illustrative. |

Items 8–14 are design corrections. The C wasn't compiled here; that needs the WDK.

---

## Not verified (needs Windows)

Nothing in this review ran on Windows.

- The ETW snippets type-check but have not run.
- Whether Kernel-Audit-API-Calls logs kernel-mode `ZwOpenProcess` callers.
- Stack-trace overhead at real event rates.
- How anti-cheats react to a stack-tracing ETW session.
- EasyAntiCheat and BattlEye refusing test-signing mode (Vanguard is confirmed).
- `atvi-randgrid.sys` as Ricochet's current driver name.

These are the Phase 2 spike's job.
