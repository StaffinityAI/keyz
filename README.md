# keyz

A key remapping daemon for macOS, written in Zig — a reimplementation of
[keyd](https://github.com/rvaiya/keyd) that reads the **same configuration
format**.

> **Status:** early development. Nothing below works yet; this document
> describes the target behaviour. Progress is tracked as git-native issues
> (`git issue ls`).

## Why

keyd offers a small, readable config language with powerful primitives —
layers, oneshots, overloads (tap/hold), chords, macros — but it is Linux-only
(evdev/uinput). macOS options are either heavyweight GUI tools or use their own
config formats. keyz aims to let you take an existing keyd config to a Mac and
have it just work.

## Goals

- **Config compatibility with keyd v2.** Existing `.conf` files should parse
  and behave the same. Anything that cannot be supported on macOS is reported
  clearly by `keyz check`, never silently ignored.
- **System-wide, low-level remapping.** Works in every app, the login window
  and secure input fields, not only in apps that accept synthetic events.
- **Per-device configs** via `[ids]` (`vendor:product`), so an external
  keyboard and the built-in one can have different mappings.
- **Single static binary**, no runtime dependencies beyond what macOS ships.
- **Familiar CLI**, mirroring keyd's subcommands.

## Non-goals (for now)

- A GUI.
- Linux support — use keyd.
- Per-application mapping (`keyd-application-mapper`); may come later.

## Configuration

Config files use keyd's format. See the
[keyd man page](https://github.com/rvaiya/keyd/blob/master/docs/keyd.scdoc) for
the full reference. A short example:

```ini
[ids]
*

[main]
# Caps Lock: Escape when tapped, Control when held
capslock = overload(control, esc)

# Hold space for the nav layer
space = overload(nav, space)

[nav]
h = left
j = down
k = up
l = right
```

Config files are `*.conf` files that each start with an `[ids]` section, as in
keyd. keyz looks for them in the standard macOS folders, resolved with
[known-folders](https://github.com/ziglibs/known-folders):

| Scope | Directory |
| --- | --- |
| System-wide (daemon) | `/Library/Preferences/keyz/` |
| Per user | `~/Library/Application Support/keyz/` |

`include` paths resolve relative to the config directory. Existing keyd files can
be copied there unchanged.

### Supported features (target)

| Feature | Notes |
| --- | --- |
| `[ids]` with `*`, explicit ids, `-` exclusions, `k:`/`m:` prefixes | ids are USB/Bluetooth `vendor:product` |
| `[main]`, custom layers, modifier layers (`[name:C-S]`) | `M` maps to Command |
| Composite layers (`[a+b]`), layouts (`[name:layout]`) | |
| `[aliases]`, `include` | |
| `[global]` options (`chord_timeout`, `oneshot_timeout`, …) | |
| Chords (`j+k = esc`) | |
| Actions: `layer`, `oneshot`, `toggle`, `swap`, `setlayout`, `clear`, `overload`, `overloadt`, `overloadt2`, `overloadi`, `lettermod`, `timeout`, `repeat`, `noop`, `*m` variants | |
| Macros: `macro()`, `macro2()`, `C-a` style chords | |
| `command()` | runs as the daemon's user |
| Unicode output | keyd uses XCompose; keyz will need a macOS-specific mechanism |
| `layer_indicator` | Caps Lock LED, where hardware allows |

### Modifier mapping

| keyd | macOS |
| --- | --- |
| `control` / `C` | Control |
| `shift` / `S` | Shift |
| `alt` / `A` | Option |
| `meta` / `M` | Command |
| `altgr` / `G` | Right Option |

## CLI (planned)

```
keyz                     run the daemon
keyz check [file...]     validate config files
keyz reload              reload configuration
keyz monitor [-t]        print key events and device ids
keyz listen              print layer changes
keyz bind <binding>...   apply bindings at runtime (or `reset`)
keyz list-keys           list valid key names
keyz input <text>...     type text
keyz do <macro>          run a macro expression
```

## How it works (planned)

1. **Input:** seize matching keyboards with IOKit HID (`IOHIDManager`) so their
   events do not reach the system directly.
2. **Processing:** a platform-independent engine (config parser, layer stack,
   timers, actions) — the same model as keyd, written in Zig and unit tested
   without any hardware.
3. **Output:** emit the resulting events through a virtual HID keyboard so the
   OS sees a real device.

The daemon runs as root under launchd and needs the macOS **Input Monitoring**
permission.

### Virtual keyboard driver

For v1, keyz sends its output through
[Karabiner-DriverKit-VirtualHIDDevice](https://github.com/pqrs-org/Karabiner-DriverKit-VirtualHIDDevice),
the standalone driver package behind Karabiner-Elements. It is **not**
Karabiner-Elements: there is no UI, no updater and no remapping engine, just the
driver and a small helper daemon. [kanata](https://github.com/jtroo/kanata) uses
it the same way.

1. Install `Karabiner-DriverKit-VirtualHIDDevice-<version>.pkg` from its
   releases page.
2. Activate the driver and approve the system extension when macOS asks:

   ```sh
   /Applications/.Karabiner-VirtualHIDDevice-Manager.app/Contents/MacOS/Karabiner-VirtualHIDDevice-Manager activate
   ```

3. The `Karabiner-VirtualHIDDevice-Daemon` must be running as root. keyz will
   ship a launchd plist for it.

keyz talks to that daemon over its root-only Unix socket. The daemon rejects
clients built for a different protocol version, so keyz documents which driver
version it supports. A driver of keyz's own may replace this later. It would need
a DriverKit entitlement from Apple.

**Panic sequence:** as in keyd, pressing `backspace+escape+enter` together
terminates keyz, in case a config locks you out.

## Building

Requires Zig 0.17.1 or newer.

```sh
zig build            # binary at zig-out/bin/keyz
zig build test       # run tests
zig build run -- check my.conf
```

## Contributing

Work is tracked with git-native issues stored in the repository
(`refs/issues/*`). See [AGENTS.md](AGENTS.md) for the workflow.

```sh
git issue sync origin   # fetch and push issues
git issue ls
```

## Acknowledgements

keyz is a reimplementation of [keyd](https://github.com/rvaiya/keyd) by
Raheman Vaiya. The configuration format and behaviour come from that project.

## License

MIT — see [LICENSE](LICENSE).
