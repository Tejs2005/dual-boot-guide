# Dual Boot System — Full Guide

**Machine:** laptop · Intel i7-11800H (8C/16T) · NVIDIA RTX 3070 Laptop GPU · 16 GB RAM · one 1 TB NVMe SSD
**OSes:** Windows (NTFS side) + **Fedora Linux 44 Server Edition** (kernel 7.2.12)
**Verified live on:** 2026-11-13

Every claim in this guide was taken from a live command run on this machine or
read verbatim from its configuration files. The receipts table at the bottom
maps each fact to its source.

---

## 1. The disk — one NVMe, two operating systems

```
nvme0n1  953.9 GB (whole SSD)
├─ p1  100 MB   vfat   EFI System Partition  ← shared by BOTH OSes, mounted at /boot/efi
├─ p2   16 MB   —      Microsoft reserved partition
├─ p3  476.5 GB NTFS   Windows C: ("Windows Basic data partition")
├─ p4  854 MB   NTFS   Windows recovery
├─ p5    2 GB   ext4   /boot   (Linux kernels + GRUB config)
└─ p6  474.5 GB ext4   /       (Linux root)
```

Current usage (2026-11-13):

| Partition | Size | Used | Free |
|---|---|---|---|
| Linux root (p6) | 466 GB | 354 GB (80%) | 90 GB |
| Linux /boot (p5) | 2 GB | 823 MB (45%) | 1 GB |
| EFI (p1) | 96 MB | 40 MB (41%) | 57 MB |
| Windows C: (p3) | 476.5 GB | ~113 GB (24%, 2026-10-10 measurement) | ~365 GB |

The Linux root was 90% full on 2026-10-10 and is at 80% now — both sides were
given ~500 GB on purpose (Rule 008): space can be freed on one side for the other.

## 2. The boot chain (what happens when you press power)

1. **UEFI firmware** — boot order is `Fedora` first, `Windows Boot Manager`
   second, firmware timeout **1 second** (barely visible).
2. **Fedora's entry** launches `\EFI\FEDORA\SHIMX64.EFI` on the shared ESP →
   shim → **GRUB2**.
3. **GRUB2 menu** — `GRUB_TIMEOUT=6`, style `menu`: a 6-second menu appears on
   every boot. Pick Windows here or it boots Fedora.
4. `GRUB_DEFAULT=saved` — GRUB remembers the **last OS you chose** and highlights
   it next time. To pin one permanently: `sudo grub2-set-default <number>`.

Boot entries (from `efibootmgr`):

```
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0000
Boot0000* Windows Boot Manager  \EFI\MICROSOFT\BOOT\BOOTMGFW.EFI
Boot0001* Fedora                \EFI\FEDORA\SHIMX64.EFI
```

**Secure Boot: disabled** (normal for this setup — the proprietary NVIDIA
driver's kernel module does not require it here, and `mokutil` reports
`SecureBoot disabled`).

To change the firmware boot order (rarely needed):

```bash
sudo efibootmgr -o 0000,0001   # make Windows first
sudo efibootmgr -o 0001,0000   # back to Fedora first
```

## 3. The Linux side (Fedora 44 Server)

| Fact | Value | Source |
|---|---|---|
| OS | Fedora Linux 44 (Server Edition) | `/etc/os-release` |
| Kernel | 7.2.12-200.fc44.x86_64 (three kernels installed: 7.2.5, 7.2.9, 7.2.12) | `uname -r`, `rpm -qa kernel` |
| SELinux | Enforcing | `getenforce` |
| Firewall | `firewalld` active on Wi-Fi (`wlo1`, public zone) — allows **only** dhcpv6-client + mdns. Everything else (SSH included) is blocked | `firewall-cmd --list-all` |
| Swap | zram only (7.7 GB in RAM). **No disk swap → Linux cannot hibernate.** | `/proc/swaps` |
| GPU driver | Proprietary NVIDIA 615.78.08, active (`nvidia-smi` works). Nouveau is blacklisted via kernel cmdline (`rd.driver.blacklist=nouveau,nova_core` in `/etc/default/grub`) | `nvidia-smi`, GRUB cmdline |

Fedora uses **BLS boot entries** — the kernel list lives in `/boot/loader/entries/`,
not inside `grub.cfg`. Don't hand-edit `grub.cfg`; regenerate with
`sudo grub2-mkconfig -o /boot/grub2/grub.cfg` only when GRUB defaults change.

### Kernel-update gotcha (seen here)

The proprietary NVIDIA driver is built per-kernel (`kmod-nvidia-<version>`).
Right after a kernel update, the new module may not be built yet — if the GUI
breaks on a fresh boot, wait a few minutes for `akmods` to finish building, or
reboot into the previous kernel from the GRUB menu (older kernels stay
installed — that's the three-kernels list above).

## 4. The Windows side

- Partitions p3 (C:) and p4 (recovery), NTFS.
- **Windows version: not verified from Linux this pass** — verifying it means
  mounting the NTFS partition, which was deliberately skipped in this session.
  To verify: mount (§5), then read `ProductName`/`CurrentBuildNumber` from
  `Windows/System32/config/SOFTWARE` with a registry reader, or simply boot
  Windows once and run `winver`.
- Nothing on the Windows side is auto-mounted in Linux. **This is deliberate**
  — see §6.

## 5. Mounting Windows from Linux (the safe, sanctioned way)

Windows partitions are **not in `/etc/fstab`** — on purpose. A bad fstab entry
can make the machine unbootable, and NTFS mounting is kept unprivileged and
reversible through `udisksctl` (works without sudo for the logged-in desktop
user):

```bash
udisksctl mount -b /dev/nvme0n1p3   # Windows C:  → /run/media/linux/Windows
udisksctl mount -b /dev/nvme0n1p4   # recovery    → /run/media/linux/<volume-uuid>
udisksctl unmount -b /dev/nvme0n1p3
```

Better — use the machine's own helper script:

```bash
~/bin/windows-storage status    # is it mounted, is it safe to write
~/bin/windows-storage mount
~/bin/windows-storage path      # print the storage folder, exit 1 if not mounted
~/bin/windows-storage unmount
```

An autostart entry (`~/.config/autostart/windows-storage.desktop`) mounts it
at login. The mountpoint is **ephemeral** (`/run/media/linux/...` changes) —
always ask `windows-storage path` instead of hardcoding it.

Key Windows-side paths once mounted:

- `/run/media/linux/Windows/VRTX-BRAIN/` — Windows copy of the brain
- `/run/media/linux/Windows/vrtx-brain-storage/` — repos, server data, projects
- `/run/media/linux/Windows/ai work hub/factory-campus/` — the campus project

## 6. Write safety — the one rule that protects both OSes

**Fast Startup / hibernation was ON in Windows** (detected 2026-10-10 by the
6.7 GB `hiberfil.sys`). With it on, every Windows "shutdown" leaves the NTFS
volume marked *dirty*. Linux can write to a dirty NTFS volume — but Windows may
then fail to resume from hibernation.

**Resolved 2026-10-10:** `powercfg /hibernate off` was run in Windows and
verified from Linux after reboot. Since then writes to the mounted Windows
volume are safe — but check first if hibernation is ever re-enabled:

```bash
ls /run/media/linux/Windows/hiberfil.sys 2>/dev/null && echo "FAST STARTUP IS BACK — do not write"
```

### NTFS gotchas learned on this machine

- NTFS loses Unix file modes → git sees phantom "mode changes" on NTFS-mounted paths.
- No working symlinks.
- **Never run a live Minecraft server off NTFS.**

## 7. Daily driving

**To switch OS:** reboot → GRUB menu shows for 6 seconds → pick with arrow keys.
Because `GRUB_DEFAULT=saved`, whichever you picked last is the default next time.

**To boot Windows when GRUB already timed out into Fedora:** reboot and choose
again, or `sudo systemctl reboot --boot-loader-entry=auto-windows` (Fedora's
one-liner to reboot straight into Windows).

**To change GRUB timeout / default:** edit `/etc/default/grub`, then
`sudo grub2-mkconfig -o /boot/grub2/grub.cfg`.

**To see GRUB menu contents as systemd sees them:**
`sudo bootctl list` won't work (that's systemd-boot); use
`grub2-editenv list` (shows the saved default) and `/boot/loader/entries/*.conf`
(needs root).

## 8. Boot visuals — what you see between power-on and desktop

> **Replicating this setup on another laptop?** Follow the step-by-step,
> AI-executable runbook: [SETUP-ON-ANOTHER-LAPTOP.md](SETUP-ON-ANOTHER-LAPTOP.md)

The boot chain has four visual stages. What each one shows on this machine
(verified 2026-11-13):

| Stage | What it is | What you see here | Config |
|---|---|---|---|
| 1. Firmware splash | Laptop vendor logo | Vendor logo — not user-changeable | in UEFI firmware |
| 2. **GRUB menu** | OS picker (6 s) | Fedora's default look — no custom theme verified (`/boot/grub2/` is root-only; see below) | `/boot/grub2/grub.cfg`, `/boot/grub2/themes/` |
| 3. **Boot splash (Plymouth)** | Fills screen while Linux loads | **Plain text mode** — no graphical animation. Theme is `text`; available alternatives: `charge`, `details`, `spinner`, `tribar` | `plymouth-set-default-theme` |
| 4. **Login screen (SDDM)** | Where you type your password | **Animated video background**: `snowfall-in-forest.1920x1080.mp4` from the `aerial-cinematic-qt6` theme | `/etc/sddm.conf.d/kde_settings.conf` |

### The GIF question, answered

- The active SDDM theme is **`aerial-cinematic-qt6`** (video background, not GIF).
  SDDM drop-in configs (`/etc/sddm.conf.d/`) **override** `/etc/sddm.conf` — and
  `/etc/sddm.conf` still says `Current=sddm-astronaut-theme`, which is why it can
  look contradictory. The drop-in wins.
- The animated GIF lives in the *inactive* theme:
  `~/sddm-astronaut-theme/Backgrounds/pixel_sakura.gif` (820 KB, plus static PNG
  fallback and two video options: `hyprland_kath.mp4`, `jake_the_dog.mp4`).
- To switch the login screen to that GIF theme, change
  `/etc/sddm.conf.d/kde_settings.conf` → `Current=sddm-astronaut-theme`.

### The full boot-visual stack (deep dive)

**Active login theme — Aerial Cinematic Qt6** (video background):

- Source: https://github.com/RussH/Aerial-Cinematic-Qt6 (GPL, by RussH), Qt6,
  installed at `/usr/share/sddm/themes/aerial-cinematic-qt6/` (verified present).
- Background = `snowfall-in-forest.1920x1080.mp4`. `Main.qml` line 51 is the
  theme's **only** background source (`config.backgroundVideo`) — the
  `type=image` line in the system copy's `theme.conf.user` is ignored dead
  config (Main.qml never reads a `type` key).
- Runtime deps (verified on this machine): `qt6-qtsvg-6.11.2`,
  `qt6-qtvirtualkeyboard-6.11.2`, `qt6-qtmultimedia-6.11.2`, SDDM with
  `InputMethod=qtvirtualkeyboard` (`/etc/sddm.conf.d/virtualkbd.conf`).
- Font: `Droid Sans Mono for Powerline`, clock 84 pt, accent `#00d2ff`,
  error `#ff4b2b` (from `theme.conf`).

**Inactive GIF theme — sddm-astronaut-theme** (Keyitdev, v1.4, Qt6,
GPL-3.0+, https://github.com/Keyitdev/sddm-astronaut-theme):

- Installed at `/usr/share/sddm/themes/sddm-astronaut-theme/` (verified present).
- Ten variants, one config each under `Themes/`. The animated-GIF variant is
  `pixel_sakura.conf` → `Background="Backgrounds/pixel_sakura.gif"`
  (verified verbatim). The rest of the Backgrounds folder: `black_hole`,
  `cyberpunk`, `hyprland_kath` (.mp4), `jake_the_dog` (.mp4), `japanese_aesthetic`,
  `pixel_sakura` (.gif), `post-apocalyptic_hacker`, `purple_leaves` — all PNG
  except the two MP4s and the GIF.
- `/etc/sddm.conf` still points at this theme (`Current=sddm-astronaut-theme`)
  — a leftover. The drop-in `kde_settings.conf` overrides it.

**Why the login screen looks the way it does** (the two-config rule):

```
/etc/sddm.conf               Current=sddm-astronaut-theme   ← stale, ignored
/etc/sddm.conf.d/kde_settings.conf   Current=aerial-cinematic-qt6   ← WINS
/etc/sddm.conf.d/virtualkbd.conf     InputMethod=qtvirtualkeyboard
```

### Changing the boot visuals

```bash
# Boot splash → graphical spinner instead of text
sudo plymouth-set-default-theme spinner && sudo dracut -f

# Login screen background → any file in the theme's folder, set in its theme.conf
# (aerial-cinematic-qt6: ~/Aerial-Cinematic-Qt6/theme.conf backgroundVideo=...)

# GRUB menu background → needs root; put a theme in /boot/grub2/themes/ and
# add GRUB_THEME= to /etc/default/grub, then grub2-mkconfig (see §7)
```

⚠️ Login-screen theming runs as root at boot — a broken SDDM theme can leave
you staring at a black screen. Test themes before committing; keep a TTY
fallback (`Ctrl+Alt+F3`) in mind.

## 9. Recovery playbook (only use in an emergency)

**Linux won't boot (GRUB broken):**
1. Boot a Fedora Live USB on this machine.
2. Mount: `sudo mount /dev/nvme0n1p6 /mnt && sudo mount /dev/nvme0n1p5 /mnt/boot && sudo mount /dev/nvme0n1p1 /mnt/boot/efi`
3. Chroot: `sudo chroot /mnt` (add `--bind` mounts for /dev /proc /sys, or use
   `sudo mount -t proc proc /mnt/proc` etc.)
4. Reinstall boot files: `sudo grub2-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Fedora`
   then `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`.
5. The Windows entry is never touched by this — it lives in the ESP independently.

**Windows Boot Manager missing from UEFI:** re-create with
`sudo efibootmgr -c -d /dev/nvme0n1 -p 1 -L "Windows Boot Manager" -l '\EFI\MICROSOFT\BOOT\BOOTMGFW.EFI'`.

**Worst case — nuke and pave:** the ESP is shared, so reinstalling either OS
normally leaves the other bootable. Always make the *installed* OS's bootloader
the one you repair, not the one you're reinstalling.

## 10. Hard-won rules from this machine's history

1. **Never `sed` into `/etc/pam.d/system-auth`** — it's managed by `authselect`
   and must stay a symlink. A one-character typo there on 2026-10-14 locked out
   every `sudo`/`su` password on this laptop for six days (the password was
   never wrong; the error message `Bad number supplied for deny argument` was
   the tell).
2. **No NTFS in fstab** — mount through `udisksctl`/the helper script instead.
3. **Check for `hiberfil.sys` before writing to Windows** — see §6.
4. **`/boot/efi` is mounted `umask=0077`** — a normal user gets "Permission
   denied" listing it. That's correct behavior, not breakage.
5. **After kernel updates, give `akmods` time** before assuming the GPU is broken (§3).

## 11. Known unknowns

- Exact Windows version/build — not verified this session (§4 has the how-to).
- GRUB menu background/theme — `/boot/grub2/` is root-only from the agent, so
  unverified. Check with: `sudo ls /boot/grub2/themes/ && grep GRUB_THEME /boot/grub2/grub.cfg`.
- Whether the Windows recovery partition (p4) still works — only testable from
  Windows.
- Firmware/BIOS model settings (passwords, boot lockdown) — not inspectable
  without a reboot into firmware setup.

---

## Receipts

| Claim | Command / file | Date verified |
|---|---|---|
| Partition map | `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,LABEL,PARTLABEL` | 2026-11-13 |
| Linux usage | `df -h / /boot /boot/efi` | 2026-11-13 |
| Windows usage | brain note `location/windows-storage.md` (measured 2026-10-10) | 2026-10-10 |
| Boot order + entries + 1s timeout | `efibootmgr` | 2026-11-13 |
| GRUB timeout 6s, saved default, nouveau blacklist | `/etc/default/grub` | 2026-11-13 |
| Fedora 44 Server | `/etc/os-release` | 2026-11-13 |
| Kernel list | `rpm -qa kernel`, `uname -r` | 2026-11-13 |
| Secure Boot disabled | `mokutil --sb-state` | 2026-11-13 |
| SELinux enforcing | `getenforce` | 2026-11-13 |
| Firewall services | `firewall-cmd --list-all` | 2026-11-13 |
| NVIDIA 615.78.08 active | `nvidia-smi`, `rpm -qa \| grep nvidia` | 2026-11-13 |
| zram-only swap | `/proc/swaps` | 2026-11-13 |
| fstab (no Windows entries, ESP umask) | `/etc/fstab` | 2026-11-13 |
| udisksctl workflow + Windows-side paths | brain note `ai-brains/zcode/memory/windows-vrtx-brain-access.md` | 2026-11-13 (paths), 2026-10-10 (origin) |
| Fast Startup resolved | brain note `location/windows-storage.md` §WRITE SAFETY | 2026-10-10 |
| PAM faillock incident | brain note `to-do/2026-10-14-hardening-the-laptop.md` + `repair/still-need-fixing/2026-10-20-pam-faillock-typo-locks-out-sudo.md` | 2026-10-20 |
| Login manager is SDDM | `systemctl status display-manager.service` | 2026-11-13 |
| Active SDDM theme = aerial-cinematic-qt6 (drop-in wins over sddm.conf) | `/etc/sddm.conf.d/kde_settings.conf` vs `/etc/sddm.conf` | 2026-11-13 |
| Login background = snowfall-in-forest video | `~/Aerial-Cinematic-Qt6/theme.conf` (`backgroundVideo=`) | 2026-11-13 |
| Plymouth theme = text (no graphical splash) | `plymouth-set-default-theme`; `/usr/share/plymouth/themes/` | 2026-11-13 |
| pixel_sakura.gif lives in the inactive astronaut theme | `ls ~/sddm-astronaut-theme/Backgrounds/` | 2026-11-13 |
| windows-storage script + autostart | `ls -la ~/bin/windows-storage ~/.config/autostart/windows-storage.desktop` | 2026-11-13 |

*Maintained by the brain (ZCode). Regenerate with: mount nothing, run the receipt commands, diff against this file.*
