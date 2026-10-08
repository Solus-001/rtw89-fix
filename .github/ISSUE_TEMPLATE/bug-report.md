---
name: Bug report
about: The card still dies, or rtw89-fix did not work as documented
title: "[bug] <laptop model> - <kernel> - <one line about what happens>"
labels: bug
assignees: ''
---

<!--
Everything below can be pasted straight into a terminal. Nothing here can
hurt the machine: every command is read-only. Please do not delete the
sections you have nothing to say about - "none" is a useful answer, and an
empty box just hides whether you looked.
-->

## What happens

<!-- One or two sentences: what did you do, what did you expect, what did
you get instead? "Wi-Fi dies after suspend and does not come back" or
"--heal says the card is not on the bus". -->

## Machine

- **Laptop model:**
- **Distro and version:** <!-- e.g. openSUSE Leap 15.6, Fedora 42, Arch -->
- **Kernel:** <!-- output of: uname -r -->

## Does the machine match the card this tool is for?

```
lspci -nn | grep -i realtek
```

<!-- Expected: a line like
"01:00.0 Network controller [0280]: Realtek Semiconductor Co., Ltd. RTL8852BE
 PCIe 802.11ax Wireless Network Controller [1T1R] [10ec:b85b]".
If yours says something else (different device ID, or no match), that is
the answer: this tool only handles 10ec:b85b. -->

## What rtw89-fix sees

```
rtw89-fix --check
```

<!-- Paste the whole output. It includes the PCI addresses of the card and
its parent device, their PCI classes, and the topology chain between them,
which is exactly what is needed to work out whether the parent is a PCIe
root port or a bridge. -->

- **Have you run `sudo rtw89-fix --apply`?** yes / no
- **Did you use `--minimal`?** yes / no
- **Does the card still die after applying?** yes / no
- **If it died, did `sudo rtw89-fix --heal` bring it back?** yes / no / not tried

## Logs from the moment the card died

`dmesg` does not survive a reboot, so copy this *while the card is dead*,
or use the persistent journal:

```
sudo journalctl -k -b | grep -iE 'rtw89|8852|pci|d3cold|firmware|mac80211' | tail -50
```

<!-- or, on a live console:
dmesg | grep -iE 'rtw89|8852|pci|d3cold|firmware|mac80211' | tail -50
-->

Paste the relevant lines. Typical things worth seeing: `pci 0000:01:00.0:
device not responding`, firmware load failures, ASPM/L1 link errors, or a
hang at suspend/resume.

## Extra

<!-- Anything else: other rtw89 workarounds you already tried, whether
Bluetooth misbehaves too, whether the machine is on AC or battery when it
dies, kernel command line, etc. -->