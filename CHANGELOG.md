# Changelog

All notable changes to this project are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the tool uses
[semantic versioning](https://semver.org/).

## [0.2.0] - 2026-10-08

The first versioned release. Everything below was developed and tested on the
author's machine (HP 15-fc0000ni, Ryzen 7 7730U) with mutating runs exercised in
a namespaced fake-root sandbox rather than on the real system.

### Fixed

- **`--heal` on a card that had vanished from the bus.** `pci_power_cycle()`
  called `die()` as soon as the card was not found, so the worst case — the
  one the tool exists for — was unrecoverable. It now skips the remove step,
  rescans, waits for re-enumeration, and only gives up if the card is still
  absent.
- **`--apply` could no longer leave a half-applied state.** `write_udev_rule()`
  died when the card was absent, leaving the modprobe conf written and no udev
  rule. It now writes the rule regardless.
- **Interface lookup only matched `wlan*`.** Predictable names (`wlp1s0`) are
  the default on most distros, so `--check` reported "interface: none" and
  `--heal` could not reconnect. The interface is now resolved from the device's
  own `net/` directory, with the `wlan*` glob kept as a fallback.
- **`card_present()` tested `/sys/module/...`**, which is true for a loaded
  module even when the device is unbound or gone. The rebind wait in
  `--heal` could therefore report success while nothing was driving the card.
  It now tests the driver symlink on the device.
- **Pipes under `set -o pipefail`.** `iw ... | grep -q` could report a connected
  link as down (SIGPIPE), `iw ... | head -1` in the report had the same flaw,
  and `nmcli | awk` in the reconnect path could kill the script mid-recovery.
  All three capture their output first and test it without a pipeline.
- **`--revert` left `d3cold_allowed` at 0** for the rest of the session while
  `--check` kept reporting the fix as active. It now puts the value back to 1
  on the card and the parent, and says so plainly if it cannot.
- **`--revert` could overwrite a user-owned `rtw89.conf`** with a stale backup.

### Added

- **`--minimal`** — apply only the D3cold block: no modprobe options, no btusb
  softdep. Removes an already-installed full-mode conf and reloads the modules
  so the dropped options actually go away.
- **`--dry-run`** for `--apply` and `--revert`: lists every file that would be
  written or removed (with contents), every sysfs value that would change, and
  the module reload, radio cycle and udevadm calls. Changes nothing, needs no
  root.
- **Richer `--check`** (no changes, no root): `disable_aspm_l1ss` and
  `disable_clkreq`, `d3cold_allowed` for the card *and* its parent, the parent's
  PCI address and class, and the topology chain from the card upwards. It also
  distinguishes bound / unbound / absent, and which mode is installed.
- **SSH warning** before `--apply` and `--heal` — both reload the driver, which
  can drop the Wi-Fi link carrying the session. Warns always, asks before going
  ahead when there is a terminal to ask.
- **openSUSE support** — `zypper` in `pkg_manager()`, the SUSE family in
  `distro_family()`, and package names for `udevadm` and `rfkill`.
- **`--version`**, a `CHANGELOG.md`, a ShellCheck CI workflow and a bug report
  issue template.

### Changed

- **The udev rule matches `ATTR{vendor}`/`ATTR{device}` (`10ec:b85b`) instead
  of a hardcoded `KERNEL=="<bdf>"`.** Bus addresses move between boots; a
  hardcoded line then blocks D3cold on whatever landed at that address. The
  parent is now reached through the card's own sysfs path
  (`RUN+="/bin/sh -c 'echo 0 > /sys%p/../d3cold_allowed'"`), so no parent
  address is pinned either. Verified with `udevadm verify` and `udevadm test`.
- `root_port_bdf()` renamed to `parent_bdf()`: it returns the immediate sysfs
  parent, which is a root port on many machines and a bridge on others.
- `--check`, `--heal` and `--apply` are parsed by a flag loop, so options can
  be combined (`--apply --minimal`) and conflicting ones are rejected instead
  of silently ignored.

### Known limitations

- The udev rule's `RUN` command has never been observed firing during a real
  boot. `udevadm verify` passes and `udevadm test` shows the rule matching and
  the command queued with its path expanded, but `udevadm test` does not run
  `RUN`. After a reboot, `--check` showing `d3cold_allowed 0` on both the card
  and the parent is the confirmation.
- Tested on one machine only. See the "Tested on" table in the README.
- `disable_ps_mode`, the L1ss/clkreq pair and the btusb softdep are extra
  measures, applied as a set and never individually verified. D3cold blocking
  is the core fix; `--minimal` applies only that.

## [0.1.0]

Initial release. The script had no version string and no changelog; this entry
reconstructs it from the initial commit.

- Prevention: `/etc/modprobe.d/rtw89.conf` (`disable_ps_mode`,
  `disable_aspm_l1ss`, `disable_clkreq`, btusb softdeps, user file backed up)
  and a udev rule setting `d3cold_allowed=0` on the card and its root port,
  both applied immediately as well as at boot.
- Recovery: `--heal` radio cycle, then PCI remove + rescan, then reconnect.
- `--check` status report, `--revert` to undo, `--help`.
- Arch, Fedora and Ubuntu/Debian package-manager support.