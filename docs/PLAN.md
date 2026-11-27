# keyz project plan

This document is the long-form plan. Day-to-day work is tracked as git-native
issues (`git issue tree`). Each milestone below has an epic issue; each bullet
is a child issue.

## 1. Scope

**keyz** is a macOS key remapping daemon written in Zig that reads
[keyd](https://github.com/rvaiya/keyd) v2 configuration files and behaves like
keyd.

In scope for v1.0:

- Everything in the keyd v2 config format, except features listed under
  "unsupported" in §7.
- System-wide remapping of keyboards matched by `[ids]`, including the built-in
  MacBook keyboard.
- keyd's CLI: `check`, `monitor`, `reload`, `bind`, `listen`, `list-keys`,
  `input`, `do`.
- Running as a launchd service.

Out of scope for v1.0: GUI, Linux/Windows, per-application mapping, an own
DriverKit driver.

### Why not just use kanata?

[kanata](https://github.com/jtroo/kanata) covers most of this feature set on
macOS with the same output driver. keyz exists because:

1. **keyd config compatibility.** One config format across Linux (keyd) and
   macOS (keyz), with no translation step.
2. **keyd semantics.** Tap/hold, oneshot and chord timing follow keyd exactly,
   checked against keyd's own tests.
3. **A small Zig codebase** owned by us, with a simpler setup on macOS.

If those reasons stop being important, the fallback is a keyd → kanata config
converter. It reuses the M1 parser, and M6 has an issue for it.

## 2. Architecture

```
          ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐
 HID ───▶ │ macOS input  │──▶│    engine    │──▶│   macOS output    │──▶ virtual
 devices  │ (IOHIDManager│   │ (pure Zig,   │   │ (report builder + │    HID kbd
          │  seize)      │   │  no I/O)     │   │  vhidd client)    │
          └──────────────┘   └──────▲───────┘   └───────────────────┘
                                    │ config model
                             ┌──────┴───────┐          ┌──────────┐
                             │ config parser│◀── files │ CLI / IPC│
                             └──────────────┘          └──────────┘
```

### Module layout

| Path | Contents | May import |
| --- | --- | --- |
| `src/config/` | lexer, parser, data model, key table, diagnostics | std |
| `src/engine/` | layer stack, actions, timers, chords, macros | config model |
| `src/platform/macos/` | IOKit HID input, vhidd client, report builder, permissions | engine interfaces, C (IOKit, CoreFoundation) |
| `src/daemon/` | run loop, IPC socket, reload | everything |
| `src/cli/` | subcommand dispatch and output | everything |
| `src/root.zig` | public `keyz` module: re-exports config and engine | — |

Rule: `config` and `engine` never import platform code. They build and test on
any OS, so CI can also run them on Linux.

### Key representation

- Internal key codes are an enum generated from keyd's key name table, so the
  names in configs and in `monitor` output match keyd.
- `src/config/keys.zig` maps each code to a HID usage (page + usage id) in both
  directions. Keys without a macOS HID equivalent are flagged. `keyz check`
  warns about them.
- Modifiers: `C` Control, `S` Shift, `A` Option, `M` Command, `G` Right Option.

### Engine contract

```zig
// Deterministic: same inputs + same timestamps => same outputs.
pub fn process(self: *Engine, ev: InputEvent, now: Instant, out: *Sink) void;
pub fn tick(self: *Engine, now: Instant, out: *Sink) void; // fire due timers
pub fn nextDeadline(self: *const Engine) ?Instant;          // for the run loop
pub fn reset(self: *Engine, out: *Sink) void;               // release everything
```

- No allocation on the event path. Buffers are sized when the config is loaded.
- `Sink` receives key down/up events, layer change notifications (for `listen`)
  and `command()` requests.
- One engine instance per config file. Devices matched by the same file share
  state, as in keyd.

### Daemon threading

Single thread on a `CFRunLoop`:

- IOHID value callbacks feed `engine.process`.
- One `CFRunLoopTimer` is re-armed to `engine.nextDeadline()`.
- The IPC Unix socket is a run loop source.
- Output writes to the vhidd socket synchronously. They are small and local.

This keeps the engine single-threaded and needs no locks.

### Output

- Version 1 uses
  [Karabiner-DriverKit-VirtualHIDDevice](https://github.com/pqrs-org/Karabiner-DriverKit-VirtualHIDDevice)
  through its root-only Unix socket, with a Zig reimplementation of the client
  protocol.
- A report builder turns engine key events into keyboard, consumer and Apple
  vendor/top-case reports and tracks which keys are pressed.
- An `Output` interface lets this backend be replaced by an own DriverKit
  driver or a `CGEventPost` fallback later.

### Configuration discovery

Resolved with [known-folders](https://github.com/ziglibs/known-folders):

| Scope | Directory |
| --- | --- |
| System-wide (daemon) | `/Library/Preferences/keyz/*.conf` |
| Per user | `~/Library/Application Support/keyz/*.conf` |

`include` resolves relative to the config directory. A share directory for
layouts may be added for packaging.

## 3. Milestones and releases

| Release | Milestones | What a user can do |
| --- | --- | --- |
| v0.1 | M0, M1 | `keyz check` validates keyd configs |
| v0.2 | M2, M3 | `keyz monitor` shows keys and device ids. The engine is complete and tested. |
| v0.3 | M4 + daemon loop from M5 | First real remapping, run manually with `sudo keyz` |
| v1.0 | M5 | launchd service, IPC commands, keyd parity |
| later | M6 | Unicode, LED indicator, Homebrew, research items |

### M0 Foundation

Goal: CI, a clean module layout and a CLI skeleton.
Exit: CI is green on macOS, `keyz --help` lists the subcommands, and the template
code is gone.

### M1 Config parser

Goal: parse every keyd v2 config into a typed model with precise diagnostics.
Exit: every config and layout shipped with keyd parses, and `keyz check` reports
errors as file:line:col.

Order: data model → key table → lexer → sections ([ids], layers, global/aliases)
→ actions/macros → include → discovery → check → corpus.

### M2 Engine

Goal: a state machine with keyd's behaviour, tested without hardware.
Exit: the ported keyd behaviour tests pass and the fuzzer runs for 1 hour with no
crash or stuck key.

The engine core depends only on the data model, so M2 can start once the model
issue is done and run in parallel with the rest of M1.

### M3 macOS input

Goal: seize keyboards matched by `[ids]` and turn HID values into engine
events. Exit: `keyz monitor` shows correct names and ids for built-in, USB and
Bluetooth keyboards, and grabbing them survives hotplug and sleep/wake.

Not blocked by M1 or M2, so it can run in parallel.

### M4 macOS output

Goal: emit through the Karabiner virtual keyboard. Exit: `sudo keyz` remaps a
real keyboard end to end, the panic sequence works, and no key is stuck after
the process is killed.

### M5 Daemon and IPC

Goal: run as a launchd service with keyd's runtime commands. Exit: install
script + launchd plist; `reload`, `bind`, `listen`, `input`, `do` work; the
manual test checklist passes.

### M6 Extras

Unicode output, `layer_indicator`, Homebrew, latency benchmark, keyd → kanata
converter, an own DriverKit driver and per-app mapping (research).

## 4. Testing strategy

| Layer | How |
| --- | --- |
| Config | Unit tests per construct, plus a corpus of keyd's example configs and layouts as fixtures |
| Engine | Unit tests with a fake clock, ported keyd behaviour tests (`t/` in keyd), fuzzing for crashes, leaks and stuck keys |
| macOS input/output | Thin layers. Report builder unit tests, vhidd protocol encoding tests against recorded bytes |
| End to end | Manual checklist (§6) per release, since CI cannot seize HID devices |

## 5. Decisions

| # | Decision | Why |
| --- | --- | --- |
| D1 | Config format = keyd v2, unchanged | Core reason for the project |
| D2 | Output through the Karabiner VirtualHIDDevice standalone package for v1 | Proven, signed, no Karabiner-Elements. Cost: separate install and protocol version pinning |
| D3 | Config dirs from known-folders (`/Library/Preferences/keyz`, `~/Library/Application Support/keyz`) | macOS conventions, no hard-coded paths |
| D4 | Single-threaded CFRunLoop daemon | No locks. Timers and IOHID integrate natively |
| D5 | Pure, deterministic engine with no I/O | Testable and fuzzable, portable |
| D6 | Git-native issues (`refs/issues/*`) | Issues live with the code. See AGENTS.md |
| D7 | Zig 0.17.1 minimum | Current stable. New `std.Io` APIs |

New decisions are added here and discussed in the corresponding issue.

## 6. Manual test checklist (per release)

- Built-in MacBook keyboard, a USB keyboard, a Bluetooth keyboard
- `[ids]` with `*`, an explicit id, and an exclusion
- Login window and FileVault unlock (when the daemon runs at boot)
- Secure input fields (password fields, Terminal with secure keyboard entry)
- Sleep/wake, unplug/replug, Bluetooth reconnect
- Panic sequence; `kill -9` leaves no stuck keys
- Caps Lock, fn/globe key, media keys, brightness keys
- Driver missing or wrong version: clear error message

## 7. Risks and unsupported features

| Risk | Mitigation |
| --- | --- |
| Karabiner driver protocol changes (as at v8) | Pin a supported version, detect mismatch at startup, keep the client small |
| Client header license (Unlicense vs Boost) unclear | Check before porting (M4 issue) |
| macOS permission UX (Input Monitoring, system extension, root) | `keyz doctor`-style checks with clear guidance |
| Built-in keyboard / fn key quirks on Apple Silicon | Dedicated M3 issue, test on hardware |
| Stuck keys if keyz dies | Release all keys on exit, reset the virtual keyboard on start |
| `command()` runs as root | Document it. Optionally drop to a configured user (M5) |
| keyd timing subtleties | Port keyd's tests, read `keyboard.c` while implementing |

Unsupported or different on macOS:

- XCompose-based Unicode output: needs a macOS mechanism (M6).
- `layer_indicator`: only where the Caps Lock LED can be set (M6).
- `keyd-application-mapper`: not in v1 (M6 research).
- USB gadget / SBC mode: not applicable.
