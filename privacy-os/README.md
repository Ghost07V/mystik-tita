# privacy-os (v0.1) — Daily Driver + Anonymous dual-mode live Debian

A custom Debian "live-build" project that produces ONE bootable ISO with
TWO boot menu choices:

  * **Daily Driver mode** — persistent storage, normal networking, GPU
    drivers, Steam/Lutris/Wine, but hardened (firewall, AppArmor,
    auto security updates, kernel sysctl hardening). This is your
    everyday/gaming mode.

  * **Anonymous mode** — same ISO, all traffic forced through Tor
    (transparent proxy), MAC address randomized, no persistence mounted.
    Behaves like Tails. Use this for anything where real anonymity matters.

You pick the mode at the boot menu, every time you boot the USB.

---

## Read this before you build anything

True Tor-grade anonymity and gaming do not mix on the same session:

- Tor adds heavy latency — most online games are unplayable over it.
- Many game servers / anti-cheat systems detect and block Tor exit nodes,
  VMs, and live-boot environments, sometimes resulting in bans.
- The second you log into Steam/Epic/your game account, you're identified
  regardless of what the network layer is doing — anonymizing the
  connection doesn't anonymize *you*.

So: Anonymous mode is for browsing, messaging, research — not for being
logged into your normal gaming accounts. Daily Driver mode is for
gaming/everyday use, with security hardening but normal (attributable)
networking. That split is intentional, not a limitation I forgot to fix.

---

## What you need to build it

- A real or virtual machine running **Debian or Ubuntu** (the build runs
  natively on the host — you are not building this on the live image
  itself). 25-30 GB free disk space, decent internet connection.
- `sudo apt install live-build`
- This project folder, copied onto that machine.

## Build steps

```sh
cd privacy-os
sudo lb clean        # safe even on a first run
sudo lb config        # reads auto/config — re-run if you edit it
sudo lb build         # downloads packages, builds the chroot, makes the ISO
```

This downloads several GB of packages from Debian's mirrors and will
take anywhere from 20 minutes to well over an hour depending on your
connection and CPU. When it finishes you'll have a file like
`live-image-amd64.hybrid.iso` in the project root.

Flash it to a USB stick (8GB minimum, 16GB+ recommended if you want
persistence + game saves):

```sh
sudo dd if=live-image-amd64.hybrid.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

(Replace `/dev/sdX` with your actual USB device — double-check with
`lsblk` first. Or just use Balena Etcher / Rufus if you prefer a GUI.)

---

## Boot menu setup

`config/bootloaders/isolinux/live.cfg.in.EXAMPLE` shows the two boot
entries you need (Daily Driver vs Anonymous). live-build's exact
template filenames/paths vary a bit by version, so:

1. Run `find /usr/share/live/build -iname '*isolinux*'` on your build
   machine to find the real template files.
2. Copy the matching file(s) into `config/bootloaders/isolinux/` under
   the same filename so live-build uses your version.
3. Edit the `LABEL` stanzas to match the example file.
4. Rebuild.

The two entries differ only in their kernel boot parameters:
`... persistence` (Daily Driver) vs `... anon` (Anonymous, no persistence).

---

## Setting up persistent storage (Daily Driver mode only)

After your first boot in Daily Driver mode, create a persistent partition
on the same USB stick (this is the standard Debian live-boot mechanism,
not something custom):

```sh
sudo cfdisk /dev/sdX               # create a new partition in free space
sudo mkfs.ext4 -L persistence /dev/sdX3   # label MUST be "persistence"
sudo mount /dev/sdX3 /mnt
echo "/ union" | sudo tee /mnt/persistence.conf
sudo umount /mnt
```

(Use `cryptsetup luksFormat` first if you want it encrypted — strongly
recommended since this partition will hold your home directory, Steam
library, browser profile, etc.) From the next boot in Daily Driver mode,
this partition mounts automatically and keeps your files, game saves,
and settings across reboots. Anonymous mode never touches it.

---

## What's already hardened, in both modes

- UFW firewall: deny incoming, allow outgoing (enabled on first real boot
  via `ufw-enable.service` — `ufw` can't fully initialize inside the
  build-time chroot since there's no live kernel netfilter stack then).
- AppArmor enabled.
- `unattended-upgrades` and `fail2ban` enabled.
- Kernel sysctl hardening (`ptrace_scope`, `dmesg_restrict`,
  ICMP redirect rejection, etc.) — see
  `config/hooks/live/9000-harden.hook.chroot`.

## What's specific to Anonymous mode

- `mode-init.service` reads `/proc/cmdline` at boot. If it sees `anon`,
  it randomizes your MAC, starts Tor, and forces all TCP/DNS through it
  via iptables, rejecting anything that isn't Tor traffic.
- See `config/includes.chroot/usr/local/sbin/mode-init`.

---

## Known rough edges (this is v0.1 — iterate from here)

- Package names (`task-xfce-desktop`, `steam-installer`, etc.) match
  Debian 12 (bookworm) at time of writing — double check against current
  Debian repos if you build later; package names occasionally get renamed.
- NVIDIA proprietary drivers need the `non-free` firmware repo (already
  enabled in `auto/config`) plus the matching `nvidia-driver` package
  added to `gaming.list.chroot` — left out by default since it varies by
  GPU generation.
- The isolinux boot-menu override step is the part most likely to need
  hand-fitting to your exact live-build version — see the note above.
- Consider adding: Firefox hardening via the arkenfox user.js project,
  a VPN-by-default toggle for Daily Driver mode, Secure Boot signing,
  and a graphical mode-indicator on the desktop panel.

---

## Building via GitHub Actions (no local Linux machine needed)

A ready-made workflow lives at `.github/workflows/build-iso.yml`. It builds
the ISO on GitHub's free Ubuntu runners and hands you back the finished
file as a downloadable artifact — your own machine never runs the build.

1. Create a new **public** repository on GitHub (public = unlimited free
   Actions minutes and free artifact storage; private repos have monthly
   minute/storage limits that this build can bump into).
2. Push or upload this entire `privacy-os/` folder into that repo,
   including the `.github/workflows/build-iso.yml` file.
   - If you don't want to use git, GitHub's web UI ("Add file →
     Upload files") accepts dragged folders in modern browsers. Either
     way, executable permissions on the scripts don't reliably survive
     the trip from Windows — that's already handled: the workflow
     re-applies `chmod +x` on the relevant files itself, so don't worry
     about it.
3. Go to the repo's **Actions** tab → select "Build privacy-os ISO" →
   **Run workflow**. It's manual-trigger only (`workflow_dispatch`),
   since this is a long build you don't want firing on every commit.
4. Wait — typically 30–90 minutes. Watch the live log if you want to see
   what it's doing.
5. When it finishes, open the completed run and scroll to **Artifacts**
   at the bottom. Download `privacy-os-iso.zip`, extract it — that's
   your `live-image-amd64.hybrid.iso`.
6. Flash it with Rufus (Windows) as described above, then boot it.

**If the build fails with "No space left on device":** the free
runner's disk, even after cleanup, is finite (~25-28GB usable). Trim
`config/package-lists/gaming.list.chroot` — `wine` in particular pulls
in a lot — and re-run. Tell me the exact error from the Actions log and
I'll help adjust the package lists.

---

## Project layout

```
privacy-os/
  auto/                      lb config/build/clean wrapper scripts
  config/
    package-lists/           *.list.chroot — what gets installed
    hooks/live/               *.hook.chroot — build-time setup scripts
    includes.chroot/          files copied verbatim into the live filesystem
      usr/local/sbin/mode-init        runtime mode-detection script
      etc/systemd/system/             mode-init.service, ufw-enable.service
    bootloaders/isolinux/     example dual boot-menu config
```
