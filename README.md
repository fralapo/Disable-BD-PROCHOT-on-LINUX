<a id="readme-top"></a>

# Disable BD PROCHOT on Linux

Stop BD PROCHOT from pinning your Intel CPU at ~800 MHz — at boot and after every resume from suspend.

[![License: MIT](https://img.shields.io/github/license/fralapo/Disable-BD-PROCHOT-on-LINUX?style=flat-square)](./LICENSE)
[![Issues](https://img.shields.io/github/issues/fralapo/Disable-BD-PROCHOT-on-LINUX?style=flat-square)](https://github.com/fralapo/Disable-BD-PROCHOT-on-LINUX/issues)
[![Stars](https://img.shields.io/github/stars/fralapo/Disable-BD-PROCHOT-on-LINUX?style=flat-square)](https://github.com/fralapo/Disable-BD-PROCHOT-on-LINUX/stargazers)

BD PROCHOT (Bi-Directional PROCHOT) is an Intel feature that lets external chips (VRMs, chipset, thermal sensors) signal the CPU to throttle hard, even when the CPU itself is cool. On some laptops a dying battery or a misreporting sensor can pin the cores at 800 MHz indefinitely. This installer clears bit 0 of `MSR 0x1FC` so the CPU ignores those external signals, and it keeps clearing it after every sleep state.

## Table of Contents

- [Who this is for](#who-this-is-for)
- [What's new](#whats-new-april-2026)
- [Prerequisites](#prerequisites)
- [Install](#install)
- [Verify it worked](#verify-it-worked)
- [How it works](#how-it-works)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)
- [Cautions](#cautions)
- [License](#license)

## Who this is for

Intel laptops (Sandy Bridge and newer) stuck at a low frequency because of a broken sensor or an overzealous EC, on:

- Arch Linux
- Ubuntu, Debian and derivatives
- Fedora, CentOS, RHEL
- Bazzite, Silverblue, Kinoite and other rpm-ostree immutable distros

AMD CPUs don't expose `MSR 0x1FC` the same way, so this project is Intel-only.

## What's new (April 2026)

- **Works on immutable distros.** All unit files now live under `/etc/systemd/system/` (writable everywhere) instead of `/usr/lib/systemd/system-sleep/` (read-only on Bazzite / Silverblue / Kinoite). Fixes [#1](https://github.com/fralapo/Disable-BD-PROCHOT-on-LINUX/issues/1).
- **Every sleep state is covered.** Resume runs after `suspend`, `hibernate`, `hybrid-sleep`, and `suspend-then-hibernate`.
- **Correct MSR bit clearing.** The worker reads `MSR 0x1FC`, clears bit 0 with a bitmask, and writes the result back. No hardcoded hex that could be wrong on some CPUs.
- **`msr` module is persisted** via `/etc/modules-load.d/msr.conf`, so the service has everything it needs at early boot.
- **Kernel lockdown detection.** The installer checks `/sys/kernel/security/lockdown` up front and offers an interactive `mokutil --disable-validation` path when Secure Boot is blocking MSR writes. The worker logs the exact failure to the journal so kernel lockdown stops being a silent failure. Addresses [#4](https://github.com/fralapo/Disable-BD-PROCHOT-on-LINUX/issues/4).

## Prerequisites

`msr-tools`. The installer can fetch it for you; pick the right option when prompted.

## Install

```bash
curl -LO https://raw.githubusercontent.com/fralapo/Disable-BD-PROCHOT-on-LINUX/main/Disable_BD_PROCHOT
sudo bash Disable_BD_PROCHOT
```

The installer asks which package manager to use, then writes:

| Path | Purpose |
|---|---|
| `/usr/local/bin/disable_bd_prochot.sh` | worker that clears bit 0 of `MSR 0x1FC` |
| `/etc/systemd/system/disable_bd_prochot.service` | runs the worker at boot |
| `/etc/systemd/system/disable_bd_prochot-resume.service` | runs the worker on every resume |
| `/etc/modules-load.d/msr.conf` | loads the `msr` kernel module at boot |

On Bazzite and other rpm-ostree systems, `msr-tools` is layered into the next deployment; a reboot is required before the services can run. The installer warns you when this is the case.

<details>
<summary>Manual install (without running the installer)</summary>

1. Install `msr-tools` with your package manager.
2. Persist the `msr` module: `echo msr | sudo tee /etc/modules-load.d/msr.conf`.
3. Create `/usr/local/bin/disable_bd_prochot.sh` (chmod 0755):

    ```bash
    #!/bin/bash
    modprobe msr 2>/dev/null || true
    shopt -s nullglob
    cpus=(/dev/cpu/[0-9]*)
    [ ${#cpus[@]} -eq 0 ] && { logger -t disable_bd_prochot "no /dev/cpu/*/msr"; exit 1; }
    fail=0
    for cpu in "${cpus[@]}"; do
        cpu_id="${cpu##*/cpu/}"; cpu_id="${cpu_id%%/*}"
        cur=$(rdmsr -p "$cpu_id" 0x1FC 2>/dev/null) || { logger -t disable_bd_prochot "rdmsr failed on cpu $cpu_id"; fail=1; continue; }
        new=$(( 16#$cur & ~1 ))
        wrmsr -p "$cpu_id" 0x1FC "$(printf '0x%x' "$new")" 2>/dev/null || { logger -t disable_bd_prochot "wrmsr failed on cpu $cpu_id (kernel lockdown?)"; fail=1; }
    done
    exit $fail
    ```

4. Create `/etc/systemd/system/disable_bd_prochot.service`:

    ```ini
    [Unit]
    Description=Disable BD PROCHOT at boot
    After=multi-user.target
    ConditionPathExists=/usr/local/bin/disable_bd_prochot.sh

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/disable_bd_prochot.sh
    RemainAfterExit=yes

    [Install]
    WantedBy=multi-user.target
    ```

5. Create `/etc/systemd/system/disable_bd_prochot-resume.service`:

    ```ini
    [Unit]
    Description=Disable BD PROCHOT on resume from suspend/hibernate
    After=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
    ConditionPathExists=/usr/local/bin/disable_bd_prochot.sh

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/disable_bd_prochot.sh

    [Install]
    WantedBy=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
    ```

6. Enable both:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now disable_bd_prochot.service
    sudo systemctl enable disable_bd_prochot-resume.service
    ```

</details>

<p align="right"><a href="#readme-top">back to top</a></p>

## Verify it worked

```bash
sudo rdmsr -a 0x1FC
```

Look at the last hex digit of every line:

- even (`0 2 4 6 8 a c e`) → BD PROCHOT is disabled
- odd (`1 3 5 7 9 b d f`) → still active

Quick example:

```text
2c005c   # last digit 'c', even → disabled
2c005d   # last digit 'd', odd  → active
```

Check the cores aren't pinned low:

```bash
watch -n 1 'grep MHz /proc/cpuinfo'
```

You should see frequencies moving freely up to the turbo ceiling, not stuck near 800 MHz.

## How it works

Two systemd oneshots share one worker script.

At boot, `disable_bd_prochot.service` runs after `multi-user.target`. After every sleep state, `disable_bd_prochot-resume.service` runs the same worker. The resume unit is wired to `suspend.target`, `hibernate.target`, `hybrid-sleep.target`, and `suspend-then-hibernate.target` with both `After=` and `WantedBy=`. That combination is what the [Arch wiki Power management page](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate) documents as the reliable pattern for "run on resume".

The worker itself:

```bash
for cpu in /dev/cpu/[0-9]*; do
    cpu_id="${cpu##*/cpu/}"
    cpu_id="${cpu_id%%/*}"
    current=$(rdmsr -p "$cpu_id" 0x1FC)
    new=$(( 16#$current & ~1 ))          # clear bit 0
    wrmsr -p "$cpu_id" 0x1FC "$(printf '0x%x' "$new")"
done
```

`MSR 0x1FC` is `MSR_POWER_CTL` on Intel. Bit 0 is the BD PROCHOT enable bit: `1` means the CPU reacts to external PROCHOT assertions, `0` means it ignores them. The worker reads the current value, clears bit 0 with a bitmask, and writes the result back, so it stays correct regardless of what the other bits happen to be on your CPU.

<p align="right"><a href="#readme-top">back to top</a></p>

## Troubleshooting

### CPU still stuck at ~800 MHz after resume

Check both services are enabled:

```bash
systemctl is-enabled disable_bd_prochot.service
systemctl is-enabled disable_bd_prochot-resume.service
```

The resume unit shows `inactive (dead)` between resumes. That is expected for a oneshot. What matters is `is-enabled` returning `enabled`.

Check the last run:

```bash
journalctl -u disable_bd_prochot-resume.service -b -1
```

### `rdmsr: unknown command`

`msr-tools` is not installed. Re-run the installer or install it manually for your distro.

### `modprobe: FATAL: Module msr not found`

Some stripped kernels omit the MSR module. Confirm with `grep CONFIG_X86_MSR /boot/config-$(uname -r)` — you want `=y` or `=m`. If it is missing, you need a different kernel.

### `wrmsr: pwrite: Operation not permitted`

Kernel lockdown is blocking the MSR write. On distro kernels with Secure Boot enabled (Ubuntu, Linux Mint, Debian, Fedora) the lockdown LSM auto-engages in `integrity` mode and refuses `wrmsr` even as root.

Diagnose:

```bash
cat /sys/kernel/security/lockdown
# [none] integrity confidentiality       → not blocked
# none [integrity] confidentiality       → blocked, wrmsr will fail
```

Pick one fix:

**A — Disable Secure Boot (cleanest, recommended).**
Reboot, enter UEFI firmware, set Secure Boot to *Disabled*, save and reboot. Re-run the installer. `lockdown` will be `[none]`. The installer's pre-check detects this automatically.

**B — `mokutil --disable-validation` (no firmware access required).**
The installer offers this interactively when it detects active lockdown. Manual equivalent:

```bash
sudo apt install mokutil          # or dnf / pacman
sudo mokutil --disable-validation
# pick an 8–16 char one-time password, reboot,
# at the blue MOK Management screen choose:
#   Change Secure Boot state → enter the same password → reboot
```

Shim will then boot in *insecure mode*; the kernel sees Secure Boot as disabled and never engages lockdown. Reversible with `sudo mokutil --enable-validation`. Security impact is equivalent to disabling Secure Boot at the firmware level.

**C — Remove `lockdown` from the LSM cmdline (keeps firmware Secure Boot on).**
Works on some Debian / Ubuntu builds, not all — newer distro kernels re-trigger lockdown from the Secure Boot path even when `lockdown` is missing from `lsm=`. Try it; if `/sys/kernel/security/lockdown` still shows `[integrity]` after reboot, fall back to A or B.

```bash
cat /sys/kernel/security/lsm
# e.g. capability,lockdown,landlock,yama,...
```

Edit `/etc/default/grub`, append the LSM list **without** `lockdown` to `GRUB_CMDLINE_LINUX_DEFAULT`:

```
GRUB_CMDLINE_LINUX_DEFAULT="... lsm=capability,landlock,yama,..."
```

Then:

```bash
sudo update-grub        # Debian/Ubuntu/Mint
# or: sudo grub2-mkconfig -o /boot/grub2/grub.cfg   # Fedora
sudo reboot
cat /sys/kernel/security/lockdown   # should show [none]
```

> `lockdown=none` on its own is **not** enough when Secure Boot is enabled: the kernel ignores it by design.

After any of A/B/C, the installed services start working from the next boot or resume. Tail the journal to confirm:

```bash
journalctl -t disable_bd_prochot -b
```

### Legacy installation leftover

Older versions of the installer wrote a sleep hook to `/usr/lib/systemd/system-sleep/disable_bd_prochot`. The current installer cleans that up automatically. If anything odd persists, run `Uninstall_BD_PROCHOT` and reinstall.

<p align="right"><a href="#readme-top">back to top</a></p>

## Uninstall

```bash
curl -LO https://raw.githubusercontent.com/fralapo/Disable-BD-PROCHOT-on-LINUX/main/Uninstall_BD_PROCHOT
sudo bash Uninstall_BD_PROCHOT
```

This removes every file listed in the install table, plus the legacy sleep hook if it still exists. BD PROCHOT is re-enabled at the next reboot.

## Cautions

Disabling BD PROCHOT removes one layer of thermal defense. If the external signal was real (actual overheating rather than a bad sensor), you now have no hardware-level brake. Keep an eye on temperatures:

```bash
watch -n 1 sensors
```

If the fans can't keep up, reverse the change by running `Uninstall_BD_PROCHOT`. This project targets the specific case where the throttle trigger is a faulty external signal, not a CPU that is actually overheating.

## License

MIT. See [LICENSE](./LICENSE).

## Acknowledgments

- [ThrottleStop](https://www.techpowerup.com/download/techpowerup-throttlestop/) for the Windows-side prior art.
- Intel MSR documentation for `MSR_POWER_CTL` (MSR 0x1FC).
- Contributors who reported the post-suspend issue on immutable distros in [#1](https://github.com/fralapo/Disable-BD-PROCHOT-on-LINUX/issues/1).

<p align="right"><a href="#readme-top">back to top</a></p>
