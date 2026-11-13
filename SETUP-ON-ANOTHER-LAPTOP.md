# Setup on Another Laptop — AI-Executable Runbook

> **Who this is for:** an AI (or human) setting up a **second laptop** to match
> this machine: dual boot Windows + Fedora with the same boot visuals.
> Every step is a command or a file. Every config is copied verbatim from a
> working machine (receipts in `README.md` § Receipts).

**Order matters.** Do not skip the verification gates — if a gate fails, stop and
fix before continuing.

---

## Part 0 — Preflight (read before touching anything)

| Check | Why |
|---|---|
| Target laptop has one NVMe SSD ≥ 1 TB (or adjust partition sizes proportionally) | partition budget in Part 2 |
| **Back up the target disk first.** Partitioning is the only irreversible step here | recovery playbook in README §9 |
| UEFI boot mode (not legacy BIOS) | everything here is EFI-based |
| You have a Fedora 44 Live USB and a Windows USB | Part 1–2 |
| Know that **Secure Boot will stay disabled** | NVIDIA module + this boot stack |

## Part 1 — Windows side

1. Install Windows first (it writes its bootloader into the shared ESP without asking).
2. Partition budget on a 1 TB NVMe (matches this machine):

| # | Size | FS | Purpose |
|---|---|---|---|
| p1 | 100 MB | FAT32 | EFI System Partition (shared) |
| p2 | 16 MB | — | Microsoft reserved |
| p3 | ~476 GB | NTFS | Windows C: (installer partitions itself; budget ~113 GB used) |
| p4 | ~854 MB | NTFS | recovery (created by Windows installer) |

3. **Immediately after Windows install:** run `powercfg /hibernate off` in
   Windows (admin cmd). This keeps the NTFS volume clean for Linux-side writes.
   Verify from Linux later: `ls /Windows/hiberfil.sys` must **not** exist.

## Part 2 — Fedora 44 install

1. Boot the Fedora 44 (Server) live USB in UEFI mode. Install to the **free
   space**, custom partitioning:

| Mount | Size | FS | Note |
|---|---|---|---|
| (ESP) | reuse p1 | vfat | mount at `/boot/efi`, do NOT reformat |
| `/boot` | 2 GB | ext4 | becomes p5 |
| `/` | remainder (~474 GB) | ext4 | becomes p6 |

2. `/etc/fstab` should end up exactly:

```
UUID=...  /         ext4  defaults 1 1
UUID=...  /boot     ext4  defaults 1 2
UUID=...  /boot/efi vfat  umask=0077,shortname=winnt 0 2
```

(UUIDs are per-disk; the options columns are the part to match.)

3. After install, verify the Windows entry survived:
   `efibootmgr` must still list `Boot0000* Windows Boot Manager`.
4. Firmware boot order Fedora first: `sudo efibootmgr -t 1` (1 s firmware timeout).

## Part 3 — GRUB defaults (verbatim from this machine)

`/etc/default/grub`:

```
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved          ← remembers last choice at next boot
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=6
GRUB_CMDLINE_LINUX="rhgb quiet rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core efi_pstore.backend=efi"
```

Regenerate: `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`
(NVIDIA-only: the nouveau blacklist lines assume a proprietary driver, see Part 5.)

## Part 4 — SDDM + Qt6 dependencies (Fedora package names, verified on source machine)

```bash
sudo dnf install -y plasma-desktop sddm \
  qt6-qtsvg qt6-qtvirtualkeyboard qt6-qtmultimedia \
  powerline-fonts          # supplies "Droid Sans Mono for Powerline"
```

Versions verified working together: **qt6 6.11.2** (`qt6-qtsvg-6.11.2`,
`qt6-qtvirtualkeyboard-6.11.2`, `qt6-qtmultimedia-6.11.2`) on Fedora 44.
Verified via `rpm -qa` on the source machine.

Enable as display manager (replaces GDM if present):

```bash
sudo systemctl enable sddm && sudo systemctl disable gdm 2>/dev/null; true
```

## Part 5 — NVIDIA driver (this laptop's stack, skip on non-NVIDIA)

```bash
sudo dnf install akmod-nvidia
# wait for kmod build to finish (can take minutes after install/kernel update):
#   ls /lib/modules/$(uname -r)/weak-updates/... or just wait + reboot
```

Nouveau is blacklisted in GRUB cmdline (Part 3) — keep those flags only if you
install the proprietary driver.

**Kernel-update race:** if the GUI breaks after a kernel update, older kernels
remain in the GRUB menu — boot one and wait for `akmods` to finish.

## Part 6 — Login theme: Aerial Cinematic Qt6 (the active one)

Source: **https://github.com/RussH/Aerial-Cinematic-Qt6** (GPL, by RussH)
Video: `snowfall-in-forest.1920x1080.mp4` bundled with the repo.

1. Clone and install:

```bash
git clone https://github.com/RussH/Aerial-Cinematic-Qt6
sudo cp -r Aerial-Cinematic-Qt6 /usr/share/sddm/themes/aerial-cinematic-qt6
```
```

2. **Set as current theme** — write `/etc/sddm.conf.d/kde_settings.conf` verbatim:

```ini
[Autologin]
Relogin=false
Session=
User=

[General]
HaltCommand=
RebootCommand=

[Theme]
Current=aerial-cinematic-qt6
CursorTheme=breeze_cursors
Font=Noto Sans,10,-1,5,50,0,0,0,0,0

[Users]
MaximumUid=60000
MinimumUid=1000
```

3. Also write `/etc/sddm.conf.d/virtualkbd.conf`:

```ini
[General]
    InputMethod=qtvirtualkeyboard
 `/etc/sddm.conf` (the non-drop-in file) can stay at its default. Drop-ins in
 `/etc/sddm.conf.d/` **override** `/etc/sddm.conf` — this is the #1 thing people get wrong.
4. Theme config `theme.conf` (inside the theme folder) — key settings, verbatim:

```ini
[General]
backgroundVideo=snowfall-in-forest.1920x1080.mp4
showSessionSelect=true
fadeDuration=10000
inputPadding=20
displayFont="Droid Sans Mono for Powerline"
clockFontSize=84
inputFontSize=14
clockFontColor="white"
errorMsgFontColor="#ff4b2b"
accentColor="#00d2ff"
```

**Important:** the theme only reads `backgroundVideo` (`Main.qml` line 51 — the
only background source in the theme). A stray `type=image` in `theme.conf.user`
is ignored dead config; don't be confused by it on a working machine.

5. Virtual keyboard is forced off in theme.conf (`showVirtualKeyboard=false`)
   for ThinkPad touchscreen behavior — leave it.
6. **Test before reboot** (theme runs as root at boot — broken theme = black
   screen). Safest test: log out and look at the SDDM greeter. Keep
   `Ctrl+Alt+F3` in mind for TTY fallback; fix = set
   `Current=...` back to a known-good theme in `/etc/sddm.conf.d/kde_settings.conf`.

## Part 7 — Optional: the GIF login theme (sddm-astronaut-theme)

Source: **https://github.com/Keyitdev/sddm-astronaut-theme** (GPL-3.0+, v1.4, Qt6)
This is where the **animated GIF** login background lives: `Backgrounds/pixel_sakura.gif`.

1. Dependencies (Fedora): same as Part 4 (qt6-qtsvg, qt6-qtvirtualkeyboard,
   qt6-qtmultimedia).
2. Install:

```bash
git clone https://github.com/Keyitdev/sddm-astronaut-theme
sudo cp -r sddm-astronaut-theme /usr/share/sddm/themes/
```

3. Pick a variant by editing `Themes/<variant>.conf` — ten variants ship
   (`astronaut`, `pixel_sakura` ← the GIF, `black_hole`, `cyberpunk`, ...).
   The GIF variant is `pixel_sakura.conf`:
   `Background="Backgrounds/pixel_sakura.gif"` (verified verbatim on source machine).
4. To select it instead of Aerial: `/etc/sddm.conf.d/kde_settings.conf` →
   `Current=sddm-astronaut-theme` (drop-in wins over `/etc/sddm.conf`).
   NOTE: `/etc/sddm.conf` on the source machine still says
   `Current=sddm-astronaut-theme` — a leftover, overridden by the drop-in. Don't copy it.

## Part 8 — Plymouth boot splash

Verified state on source machine: **`text`** (plain text, no graphical splash).
Alternatives installed: `charge`, `details`, `spinner`, `tribar`.

```bash
plymouth-set-default-theme              # show current
sudo plymouth-set-default-theme spinner && sudo dracut -f   # graphical instead
```

## Part 9 — Windows↔Linux sharing rules (same as source machine)

- Windows partitions **NOT in fstab** — mount via udisksctl or a helper script.
- Fast Startup off (Part 1 step 3) or NTFS write safety breaks.
- NTFS gotchas: no Unix modes (git mode-ghosts), no symlinks, never a live
  Minecraft server on NTFS.

## Part 10 — Verification gates (run in order, all must pass)

| # | Command | Expected |
|---|---|---|
| 1 | `lsblk` | p1 ESP + NTFS + ext4 layout like README §1 |
| 2 | `efibootmgr` | Windows + Fedora entries, Fedora first, 1s timeout |
| 3 | `mokutil --sb-state` | SecureBoot disabled |
| 4 | `cat /etc/sddm.conf.d/kde_settings.conf` | Current=aerial-cinematic-qt6 |
| 5 | `plymouth-set-default-theme` | `text` |
| 6 | `ls /usr/share/sddm/themes/aerial-cinematic-qt6/theme.conf` | exists |
| 7 | `rpm -q qt6-qtmultimedia` | qt6 6.11.x package present |
| 8 | Reboot → GRUB menu 6s → login screen shows snowfall video | visually confirmed |

## Part 11 — What does NOT transfer (per-laptop facts)

- Partition UUIDs, hostname, users, ssh keys, `powerline-fonts` package name may differ
- Firmware logo is vendor-fixed
- Non-NVIDIA laptop: skip Part 5 and remove nouveau-blacklist flags (Part 3)
- Non-1TB disk: scale p3/p6 proportionally
- Source machine: i7-11800H, RTX 3070 Laptop, 16 GB RAM, 1 TB NVMe — visual results may differ on other GPUs
