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

The RTL8852BE is common in budget and mid-range laptops (HP, Lenovo, ASUS and
others). On many of them the card randomly drops dead — WiFi vanishes until a
reboot — because the kernel lets the device enter the PCI **D3cold** power
state. D3cold is *below* the driver: the card's crystal oscillator never
restarts, its firmware is gone for good, and reloading the `rtw89` modules
can't fix it. Only a full reboot — or a PCI bus remove/rescan — revives it.

> Other rtw89 cards (RTL8852AE `10ec:8852`, RTL8852CE `10ec:c852`, …) share
> the same D3cold behaviour. The script matches one PCI ID and one module
> set, but both are variables near the top of the file if you want to adapt
> it. Untested on anything but the RTL8852BE.

## What it does

**Prevention** (default action, `rtw89-fix` or `rtw89-fix --apply`):

- Writes `/etc/modprobe.d/rtw89.conf`:
  - `disable_ps_mode=1` — keeps the firmware out of LPS low-power mode
  - `disable_aspm_l1ss=1`, `disable_clkreq=1` — keeps the PCIe link out of L1ss
  - `softdep … pre: btusb` — loads after Bluetooth firmware upload so the
    shared RF frontend doesn't race (your existing conf is backed up first)
- Writes `/etc/udev/rules.d/99-rtw89-d3cold.rules`, which sets
  `d3cold_allowed=0` on the card **and** its PCIe root port at every boot —
  D3cold is negotiated port-wide, so blocking only the card isn't enough on
  some firmware
- Applies the D3cold block immediately, so it's active before your next
  suspend rather than after your next reboot

**Recovery** (`rtw89-fix --heal`), when the card is already dead:

1. Cycles the radio (enough if it's only stuck in low-power mode)
2. If that fails: removes the card from the PCI bus and rescans, forcing the
   kernel to re-probe it from a cold bus — the reboot-equivalent that doesn't
   cost you your session
3. Reconnects via NetworkManager when available

**Status** (`rtw89-fix --check`): prints the distro, PCI device, driver,
interface, link state, and whether both fixes are active. No changes, no root
needed.

**Undo** (`rtw89-fix --revert`): removes every file the script created and
restores any backed-up modprobe conf.

## Supported distros

Works on all three major families; the distro is detected from
`/etc/os-release` (`ID` + `ID_LIKE`, so derivatives are covered):

- **Arch-based** — Arch, Manjaro, EndeavourOS, … (uses `pacman`)
- **Fedora-based** — Fedora, RHEL, CentOS, Rocky, Alma, … (uses `dnf`)
- **Ubuntu/Debian-based** — Ubuntu, Debian, Linux Mint, Pop!\_OS, … (uses `apt`)

If a required tool (`modprobe`, `udevadm`) is missing, the script names the
exact package for your distro and offers to install it. `iw` and
NetworkManager are optional — on iwd or systemd-networkd setups the affected
steps are skipped with a note instead of failing.

## Install & usage

```bash
git clone https://github.com/Solus-001/rtw89-fix.git
cd rtw89-fix
install -Dm755 rtw89-fix ~/.local/bin/rtw89-fix   # or /usr/local/bin

rtw89-fix            # apply prevention (asks for sudo; safe to re-run)
rtw89-fix --heal     # recover a dead card right now
rtw89-fix --check    # show status, no changes
rtw89-fix --revert   # remove everything it created
rtw89-fix --help
```

Requirements: `bash`, `kmod`, and `udev` — all present by default on the
supported distros.

## ⚠️ Use at your own risk

**This tool modifies system configuration and pokes hardware power
management directly.** It writes to `/etc/modprobe.d/` and
`/etc/udev/rules.d/`, unloads kernel modules, and — during `--heal` —
**removes a device from your PCI bus and rescans the whole bus**.

It is provided **as is, with no warranty of any kind**. By running it you
accept full responsibility for anything that happens: instability, data
loss, hardware damage, or a card that still dies. It has been tested on the
author's machine only. Read the script before running it (it's short and
heavily commented), use `--check` to look before you touch, and `--revert`
if you want out.

## License

[MIT](LICENSE) — which, fittingly, also says the software is provided
"as is", without warranty.
