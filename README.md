# rtw89-fix

A small Bash tool that stops the **Realtek RTL8852BE** WiFi card from dying on
Linux — and brings it back **without a reboot** when it already has.

## The card

| | |
|---|---|
| Chip | **Realtek RTL8852BE** |
| Type | Wi-Fi 6 (802.11ax) + Bluetooth combo card |
| PCI ID | `10ec:b85b` |
| Kernel driver | `rtw89` (`rtw89_8852be`, `rtw89_pci`, `rtw89_core`) |

## Is this your problem?

The card has to actually be an RTL8852BE:

```bash
lspci -nn | grep -i realtek
```

You are looking for a line containing **`[10ec:b85b]`**:

```
01:00.0 Network controller [0280]: Realtek Semiconductor Co., Ltd. RTL8852BE PCIe 802.11ax Wireless Network Controller [1T1R] [10ec:b85b]
```

Anything else means this tool is not for you. Other rtw89 cards (RTL8852AE
`10ec:8852`, RTL8852CE `10ec:c852`, …) fail in a similar way, but the script
matches one PCI ID and one module set; both are variables near the top of the
file if you want to adapt it.

If the ID matches and the Wi-Fi randomly disappears, this is what it looks like
from inside:

- The interface is gone from `ip link` (or shows `state DOWN` and never comes
  back), often only after suspend/resume or an idle period.
- Reloading the driver does **not** help: `modprobe -r rtw89_8852be &&
  modprobe rtw89_8852be` looks like it should, and does nothing.
- A reboot always fixes it, until the next time.
- `dmesg` is often surprisingly quiet about it — the failure is below the
  driver, so there may be nothing at all in the log. When there *is* something,
  it tends to be around the resume: PCIe AER or link errors from the root port
  (`pcieport 0000:00:02.2: AER: ...`), a firmware load or probe failure, or an
  `rtw89` timeout. Exact wording varies by kernel; paste whatever is there.

## What it does

**Prevention** (default action, `rtw89-fix` or `rtw89-fix --apply`):

- Writes `/etc/udev/rules.d/99-rtw89-d3cold.rules`, which sets
  `d3cold_allowed=0` on the card **and** on the device it hangs off (the
  immediate parent — a PCIe root port, or a bridge on some laptops) at every
  boot. The rule matches `ATTR{vendor}==0x10ec` + `ATTR{device}==0xb85b`, so it
  survives the bus being renumbered, and it reaches the parent through the
  card's own sysfs path rather than by hardcoding an address.
- Writes `/etc/modprobe.d/rtw89.conf` **in full mode only** (see below):
  - `disable_ps_mode=1` — keeps the firmware out of LPS low-power mode
  - `disable_aspm_l1ss=1`, `disable_clkreq=1` — keeps the PCIe link out of L1ss
  - `softdep … pre: btusb` — loads after Bluetooth firmware upload so the
    shared RF frontend doesn't race (your existing conf is backed up first)
- Applies the D3cold block immediately, so it's active before your next
  suspend rather than after your next reboot

**Recovery** (`rtw89-fix --heal`), when the card is already dead:

1. Blocks D3cold again (it is per-boot state, so this has to be re-applied)
2. Cycles the radio — enough if the card is only stuck in low-power mode
3. If that fails: removes the card from the PCI bus and rescans, forcing the
   kernel to re-probe it from a cold bus — the reboot-equivalent that doesn't
   cost you your session. This also works when the card has vanished from the
   bus entirely; if it does not come back, the script says so instead of
   quietly giving up.
4. Reconnects via NetworkManager when available

**Status** (`rtw89-fix --check`): no changes, no root needed. Prints the
distro, the card's PCI address and bound driver, the interface and link state,
every driver parameter this tool sets (`disable_ps_mode`, `disable_aspm_l1ss`,
`disable_clkreq`), `d3cold_allowed` for the card **and** its parent, the
parent's PCI address and class, the topology chain from the card upwards, and
which mode the installed files correspond to.

**Undo** (`rtw89-fix --revert`): removes every file the script created, restores
any backed-up modprobe conf, and puts `d3cold_allowed` back to 1 on the card
and the parent.

## Modes: `full` and `--minimal`

| | `--apply` (full, default) | `--apply --minimal` |
|---|---|---|
| `d3cold_allowed=0` on card + parent | yes | **yes** |
| `disable_ps_mode` | yes | no |
| `disable_aspm_l1ss`, `disable_clkreq` | yes | no |
| `softdep … pre: btusb` | yes | no |

```bash
rtw89-fix --minimal      # same as: rtw89-fix --apply --minimal
```

**Blocking D3cold is the core fix.** It is the one measure this tool is built
around: D3cold is the state that kills this card, and blocking it is what makes
the difference. The three driver-level settings — `disable_ps_mode`, the
L1ss/clkreq pair, and the `btusb` softdep — are extra measures that a lot of
workarounds use together with the D3cold block. They are **not individually
verified** here: nobody has isolated which of them, if any, does anything on
its own, and on the author's machine they were applied as a set. They also
carry the battery cost (below) and, like any driver parameter, can cause
problems on hardware where they are not needed. If `--apply` makes things
worse, or you only want the one measure you are confident in, use `--minimal`.

`--check` tells you which mode is installed: a modprobe conf carrying the
rtw89-fix marker means full mode, the udev rule without one means minimal.

### See it before you do it

```bash
rtw89-fix --apply --dry-run      # or --revert --dry-run
```

Prints every file that would be written or removed (with the full contents of
the two config files), every `sysfs` value that would change, and the module
reload, radio cycle and `udevadm` calls it would make — then changes nothing.
No root needed.

## Install & usage

```bash
git clone https://github.com/Solus-001/rtw89-fix.git
cd rtw89-fix
install -Dm755 rtw89-fix ~/.local/bin/rtw89-fix   # or /usr/local/bin

rtw89-fix            # apply prevention (asks for sudo; safe to re-run)
rtw89-fix --minimal  # apply prevention, D3cold block only
rtw89-fix --heal     # recover a dead card right now
rtw89-fix --check    # show status, no changes
rtw89-fix --revert   # remove everything it created
rtw89-fix --help
```

`--apply`, `--revert` and `--heal` need root (the script re-execs itself under
`sudo`). If you are connected over SSH, they say so first and ask before going
ahead, because reloading the driver can drop the Wi-Fi link carrying the
session — answer `y` to continue, or run it locally.

Requirements: `bash`, `kmod`, and `udev` — all present by default on the
supported distros.

## Tested on

| Machine | CPU | Distro | Kernel | Notes |
|---|---|---|---|---|
| HP 15-fc0000ni | AMD Ryzen 7 7730U | Arch Linux (CachyOS kernel) | 7.2.9-1-cachyos | author's machine; Wi-Fi over iwd/NetworkManager |

That is the whole list: this has been tested on one laptop. Everything else in
this README is written from that one machine and the kernel sources.

**Your machine is probably not in the table.** If it works there, please open
an issue using the [bug report template](.github/ISSUE_TEMPLATE/bug-report.md)
and say what you have — even "it works" is useful, and the template makes the
information to collect obvious.

One caveat about the udev rule: it was checked with `udevadm verify` and
`udevadm test` on the author's card (matches, the `d3cold_allowed` write and
the parent `RUN` command with its path expanded correctly), but `udevadm test`
never *runs* `RUN` commands — so the rule firing during an actual boot is the
one part still waiting on a real confirmation. `--check` shows the live value
of `d3cold_allowed` on the card and the parent after a reboot; if both are 0,
the rule fired.

## Supported distros

The distro is detected from `/etc/os-release` (`ID` + `ID_LIKE`, so derivatives
are covered):

- **Arch-based** — Arch, Manjaro, EndeavourOS, … (uses `pacman`)
- **Fedora-based** — Fedora, RHEL, CentOS, Rocky, Alma, … (uses `dnf`)
- **Ubuntu/Debian-based** — Ubuntu, Debian, Linux Mint, Pop!\_OS, … (uses `apt`)
- **openSUSE-based** — openSUSE Leap/Tumbleweed/MicroOS, SLES (uses `zypper`)

If a required tool (`modprobe`, `udevadm`) is missing, the script names the
exact package for your distro and offers to install it. `iw` and
NetworkManager are optional — on iwd or systemd-networkd setups the affected
steps are skipped with a note instead of failing.

## Battery life

This is a trade, stated plainly:

- **Blocking D3cold** means the card is never put into its deepest power state.
  A card that is powered down can suspend the radios and the whole PCIe link;
  a card that is blocked from D3cold stays powered, drawing more when idle.
- **Disabling L1ss** (`--apply`, not `--minimal`) means the PCIe link between
  the card and its parent does not enter its low-power substates either.

Both generally cost idle battery on a laptop; how much depends on the machine,
the kernel and how you use it. If it matters, measure before and after on your
own hardware rather than trusting a number from someone else — no figure here
is quoted because none has been measured in a controlled way.

If battery life is more important than a stable card, `--minimal` at least
gives back the L1ss part.

## Troubleshooting

### The card still dies after applying

Work through these in order; each one rules out a different cause:

1. **`rtw89-fix --check`.** Is `d3cold card: 0` and `d3cold parent: 0`?
   If either is `1`, the block is not in effect — reboot and check again,
   since the values are set at boot by the udev rule. If the parent line says
   `n/a`, the card was not on the bus when you checked.
2. **Check you really have the card.** `lspci -nn | grep -i realtek` must show
   `[10ec:b85b]`.
3. **Was a full or minimal apply installed?** `rtw89-fix --check` prints the
   mode. If a full apply is installed, try `--apply --minimal` to remove the
   driver parameters — they are the unverified part, and if one of them is
   fighting your firmware, minimal mode is the way to find out.
4. **Does `--heal` bring it back?** If yes, you have a recovery path and the
   card is dying, not permanently dead: `sudo rtw89-fix --heal` until you
   need a reboot. If it dies again, the prevention is not covering the case.
5. **Suspend or something else?** Note what precedes it. Some machines only
   die on suspend/resume, some only on long idle, some when a dock is attached.

### What to attach to a bug report

Use the [bug report template](.github/ISSUE_TEMPLATE/bug-report.md). The short
version:

```bash
rtw89-fix --check
lspci -nn | grep -i realtek
uname -r
sudo journalctl -k -b | grep -iE 'rtw89|8852|pci|d3cold|firmware|mac80211' | tail -50
```

Copy the journal lines **while the card is dead**, or from the boot where it
died — `dmesg` is gone after a reboot. The topology block in `--check` output
matters too: it says whether the card hangs off a PCIe root port or a bridge,
which changes what needs blocking.

## ⚠️ Use at your own risk

**This tool modifies system configuration and pokes hardware power
management directly.** It writes to `/etc/modprobe.d/` and
`/etc/udev/rules.d/`, unloads kernel modules, and — during `--heal` —
**removes a device from your PCI bus and rescans the whole bus**.

It is provided **as is, with no warranty of any kind**. By running it you
accept full responsibility for anything that happens: instability, data
loss, hardware damage, or a card that still dies. It has been tested on the
author's machine only. Read the script before running it (it's short and
heavily commented), use `--check` to look before you touch, `--dry-run` to see
exactly what a run would do, and `--revert` if you want out.

## License

[MIT](LICENSE) — which, fittingly, also says the software is provided
"as is", without warranty.