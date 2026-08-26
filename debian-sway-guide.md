# Debian Stable + Sway on a ThinkPad T14 Gen 3

Everything worth knowing to build this and then not think about it for three years.

**For the step-by-step install, see [`INSTALL.md`](INSTALL.md).** This document
is the reference — why each choice was made, and what breaks when it's made
differently. INSTALL.md links back into the sections here.

---

## 0. The design bet

The goal is to *work* on this machine, not maintain it. That constrains every
decision below:

- **Base system: boring and frozen.** Debian stable. Two-year release cycle,
  five-year support, no surprise package majors.
- **Freshness: layered on top, and only where you need it.** Nix or Flatpak for
  the browser and current dev tooling. Never by pulling from testing/sid.
- **Minimality: by package selection, not dependency suppression.** You install
  `sway`, not a desktop task. Recommends stay enabled — Debian Policy treats
  them as strong dependencies, and turning them off globally buys a smaller
  package count at the price of packages that are installed but subtly
  incomplete. See §2.

The failure mode this avoids is the Arch one: attention paid continuously in
small amounts. The failure mode it introduces is *staleness*, which is real but
bounded and fixable in exactly two places (browser, dev tools). That's the
trade.

---

## 1. Installation

### Which image

Grab the **netinst** image for Trixie. Since Bookworm, the official images
include non-free firmware by default, so there's no separate "unofficial
firmware" image to hunt for anymore. Verify the checksum — Debian signs
`SHA256SUMS` with the CD signing key.

### Partitioning

For a laptop that leaves the house, full-disk encryption is not optional.

```
/boot/efi   512M    FAT32, unencrypted (required)
/boot       1G      ext4, unencrypted  (optional — see below)
/           rest    LUKS2 → ext4 or btrfs
```

**Points to get right:**

- **LUKS2 with argon2id** is the default in Trixie's installer. Fine as-is.
- **`/boot` inside or outside LUKS?** GRUB can unlock LUKS2, but it's slow
  (argon2 in GRUB is painful) and you end up typing the passphrase twice.
  Keeping `/boot` unencrypted is the pragmatic choice; the threat it exposes
  you to is an evil-maid bootloader tamper, which Secure Boot partially
  mitigates.
- **Swap sizing.** You have 32 GB RAM. If you want hibernation, swap must be
  ≥ RAM, so a 32–36 GB swap partition. If you don't care about hibernate
  (see §6), an 8 GB swapfile or zram is plenty. Decide now — resizing later
  under LUKS is annoying.
- **ext4 vs btrfs.** ext4 if you want zero thought. btrfs if you want
  snapshots before upgrades, which genuinely helps at release-upgrade time.
  btrfs on a single NVMe with no RAID is stable and safe in 2026. If you pick
  btrfs, use subvolumes `@` and `@home` so snapshots don't drag your data
  along.

### tasksel — the important screen

**Deselect every desktop environment. Leave "standard system utilities"
checked.**

The common minimalist advice is to deselect standard as well, and I'd argue
against it for the same reason we leave recommends enabled (§2): it's
minimalism by suppression rather than by selection.

What "standard" actually contains is about a hundred packages, nearly all of
which you want on a working machine — `cron`, `logrotate`, `less`, `man-db`,
`wget`, `procps`, `file`, `bash-completion`, `xz-utils`. Deselecting it means
re-adding most of them by hand, and forgetting one shows up weeks later as
"why aren't my logs rotating."

The usual argument for deselecting is that you end up with an explicit,
auditable package list. But `apt-mark showmanual` gives you that regardless,
so you're paying a real cost for something you already have.

`bootstrap-sway` still lists a few of these explicitly (`less`, `man-db`,
`bash-completion`, `unzip`). That's harmless redundancy — apt skips what's
already present — and it means the script still produces a working system if
you *did* deselect standard, or if you run it on a container image.

### The root password trick

If you **leave the root password blank** in the installer, Debian disables the
root account and adds your user to `sudo` automatically — the Ubuntu model
you're used to. If you set a root password, your user is *not* in `sudo`, and
you'll be confused when the bootstrap script fails on its first `sudo` call.
Fix from a root shell: `usermod -aG sudo yourname`, then log out fully.

### Secure Boot

Works out of the box — Debian ships a Microsoft-signed shim. Leave it enabled.
Two consequences to know:

- Kernel lockdown mode is active, which **blocks hibernation** on some setups
  and blocks unsigned kernel modules (DKMS needs MOK enrolment).
- If you ever want to build custom modules, you'll need `mokutil` and to enrol
  a Machine Owner Key at reboot.

---

## 2. apt configuration

### Recommends: leave them on

Debian Policy §7.2 defines Recommends as a strong dependency — packages found
together with the depending package **in all but unusual installations**. That
wording is the whole argument. Disabling recommends globally is a declaration
that your system is permanently unusual, and the maintainers' testing does not
cover it.

You will see the global-off setting recommended widely:

```
# /etc/apt/apt.conf.d/99norecommends   ← NOT recommended for a desktop
APT::Install-Recommends "false";
APT::Install-Suggests "false";
```

**Where that is legitimate:** Dockerfiles, container base images,
`debootstrap`, appliance and server builds — contexts where the package set is
fixed, audited once, and never grows interactively. That's the context it was
popularised in, and it's a reasonable fit there.

**Why it's the wrong default for this laptop:** you get packages that are
installed but functionally incomplete, and the failures are quiet and
delayed — you meet them weeks later, having forgotten the cause.

| Package | Skipped recommend | Symptom |
|---|---|---|
| `thunar` | `gvfs`, `tumbler` | No trash, no removable media, no thumbnails |
| `firefox-esr` | `libavcodec-extra` | Some video won't play |
| `cups` | `cups-filters`, `printer-driver-*` | Printer detected, nothing prints |
| `network-manager` | `wpasupplicant` | Wi-Fi silently unavailable |
| `zathura` | backend plugin | Opens, renders nothing |

### The option you actually want already exists

apt prints the complete resolved package list and waits for confirmation on
every install. That *is* the per-package decision point — it's built in, and
it's why a global setting isn't needed to stay in control.

To see the same thing with no commitment at all:

```bash
apt install -s <pkg>                      # simulate, full resolved list
apt-cache depends --recommends <pkg>      # what the recommends actually are
```

So the workflow is: install normally, read the list, and if a specific package
looks disproportionate, `Ctrl-C` and re-run that one with
`--no-install-recommends`. A deliberate, documented exception beats a blanket
policy.

`bootstrap-sway` follows this model. Run it with `--compare` to see the
exact cost of recommends for this package set before installing anything.
`extras` follows it too — both take `--lean` if you ever want the opposite,
but the default in both is Debian's tested configuration.

### On the GNOME fear specifically

A minimal Sway install *will* pull in packages named `gnome-*` and
`libgnome-*` — gsettings schemas, `gnome-keyring`, GTK support libraries.
**This is not GNOME.** The desktop environment only arrives via the
metapackages `gnome`, `gnome-core`, or `task-gnome-desktop`, and nothing in a
Sway package set recommends any of them.

Verify rather than trust:

```bash
apt install -s sway | awk '/^Inst /{print $2}' | grep -xE 'gnome|gnome-core|task-gnome-desktop'
```

Empty output means no desktop environment is coming. The `--compare` mode of
the bootstrap script runs exactly this check.

### The one setting that does need thought

```
APT::AutoRemove::RecommendsImportant "false";
```

By default `apt autoremove` will not remove an auto-installed package that is
recommended by something you installed manually. Setting this false lets
autoremove reclaim them. It's more dangerous than the install-time setting
because it acts retroactively on a working system. Leave it alone unless
you're deliberately doing a pruning pass, and take a snapshot first.

### sources.list format

Trixie uses deb822 format at `/etc/apt/sources.list.d/debian.sources`. It
should look like this — note `non-free-firmware`, without which your Wi-Fi and
audio firmware won't install:

```
Types: deb
URIs: https://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main contrib non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://security.debian.org/debian-security
Suites: trixie-security
Components: main contrib non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

`contrib` is there for a few firmware helper packages. Add `non-free` only if
you specifically need it — on an all-Intel machine you likely don't.

---

## 3. Sway session plumbing — where most setups break

This is the section that costs people entire evenings. Four separate mechanisms
have to agree with each other.

### 3.1 The environment variable trap

`~/.config/environment.d/*.conf` is read by the **systemd user manager**. It
applies to systemd user *units*. It does **not** apply to processes sway spawns
when you launch sway from a login shell — those inherit the shell's environment,
which never saw `environment.d`.

This bites everyone. Symptom: Firefox runs under XWayland despite
`MOZ_ENABLE_WAYLAND=1` being "set", and screen sharing doesn't work.

**The fix** (what the bootstrap script now does): keep a plain sourceable file,
and launch through a wrapper.

```bash
# ~/.config/sway-env.sh   — sourced by the wrapper
export XDG_CURRENT_DESKTOP=sway
export MOZ_ENABLE_WAYLAND=1
export QT_QPA_PLATFORM='wayland;xcb'   # NOTE the quotes — see below
# ...

# ~/.local/bin/start-sway
#!/usr/bin/env bash
. "$HOME/.config/sway-env.sh"
exec sway "$@"
```

Always start with `start-sway`, never bare `sway`.

**Quote your values.** `QT_QPA_PLATFORM=wayland;xcb` is the correct value, but
unquoted in a shell file the `;` is a command separator: you silently get
`QT_QPA_PLATFORM=wayland` (no X11 fallback) plus a `xcb: command not found`
error at every login. systemd's `environment.d` format does *not* want the
quotes; a sourced shell file does. That difference is why `bootstrap-sway`
generates the two files from one list rather than copying one to the other.

### 3.2 Pushing the environment into D-Bus

Even with the wrapper, `xdg-desktop-portal` is started by D-Bus activation and
gets D-Bus's idea of the environment, not sway's. It has to be told explicitly.
Put this in your sway config:

```
exec systemctl --user import-environment DISPLAY WAYLAND_DISPLAY SWAYSOCK XDG_CURRENT_DESKTOP
exec dbus-update-activation-environment --systemd DISPLAY WAYLAND_DISPLAY SWAYSOCK XDG_CURRENT_DESKTOP
```

Skip this and screen sharing fails **silently** — the picker appears, you
select a window, and the other end sees black. No error anywhere.

### 3.3 Portal backend selection

You have two portal backends installed and they divide the work:

- `xdg-desktop-portal-wlr` — screencast, screenshot
- `xdg-desktop-portal-gtk` — file chooser, settings, appearance

If the wrong one claims an interface you get a file picker that never opens.
Pin it explicitly:

```
# ~/.config/xdg-desktop-portal/sway-portals.conf
[preferred]
default=gtk
org.freedesktop.impl.portal.Screenshot=wlr
org.freedesktop.impl.portal.ScreenCast=wlr
```

Debugging command when something is off:

```bash
systemctl --user restart xdg-desktop-portal xdg-desktop-portal-wlr xdg-desktop-portal-gtk
journalctl --user -u xdg-desktop-portal -f
```

### 3.4 Group membership

`brightnessctl` writes to `/sys/class/backlight/*`. Debian ships udev rules
granting the `video` group access — but **you must be in that group and have
logged out and back in**. Group changes do not apply to an existing session.

Symptom: brightness keys do nothing, `brightnessctl s 50%` says permission
denied. Fix: `sudo usermod -aG video,input $USER`, then log out *fully*
(not just restart sway).

---

## 4. Making a bare WM feel finished

A tiling WM gives you no settings daemon, so several things that "just work"
elsewhere need explicit setup.

### GTK apps look wrong / can't change theme

There's no gsettings daemon. Install `gsettings-desktop-schemas` and set
values directly:

```bash
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
gsettings set org.gnome.desktop.interface gtk-theme 'Adwaita-dark'
gsettings set org.gnome.desktop.interface cursor-theme 'Adwaita'
gsettings set org.gnome.desktop.interface font-name 'Inter 11'
```

Note that GTK4 apps read `color-scheme`, GTK3 reads `gtk-theme`. You need both.

### Qt apps look wrong

```bash
sudo apt install qt6ct qt5ct
# then in sway-env.sh:
export QT_QPA_PLATFORMTHEME=qt6ct
```

### Invisible or giant cursor

Classic Wayland issue when crossing between native and XWayland surfaces. Set
in sway config:

```
seat seat0 xcursor_theme Adwaita 24
```

and `XCURSOR_SIZE=24` / `XCURSOR_THEME=Adwaita` in the env file. Both are
needed; they cover different clients.

### Electron apps blurry or on XWayland

```bash
export ELECTRON_OZONE_PLATFORM_HINT=auto
```

Some apps ignore it; those need `~/.config/<app>/<app>-flags.conf` containing
`--ozone-platform-hint=auto`.

### Java apps show a grey box

`_JAVA_AWT_WM_NONREPARENTING=1`. Already in the env file.

### No clipboard history

`wl-clipboard` gives you copy/paste but nothing persistent, and copied content
**dies with the source application** on Wayland. `cliphist` is installed for
this; it needs two watchers and a picker binding in your sway config:

```
exec wl-paste --type text --watch cliphist store
exec wl-paste --type image --watch cliphist store

bindsym $mod+grave exec cliphist list | fuzzel --dmenu --with-nth 2 | cliphist decode | wl-copy
```

Screenshots are the same shape — `grim` and `slurp` are the primitives,
`grimshot` is the wrapper worth binding:

```
bindsym Print          exec grimshot --notify savecopy output
bindsym $mod+Print     exec grimshot --notify savecopy window
bindsym Control+Print  exec grimshot --notify savecopy area
```


### Volume and media keys do nothing

Nothing binds them for you. `wpctl` comes with wireplumber, which is already
installed, so this needs no extra package:

```
bindsym --locked XF86AudioRaiseVolume exec wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ 5%+
bindsym --locked XF86AudioLowerVolume exec wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
bindsym --locked XF86AudioMute        exec wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
bindsym --locked XF86AudioMicMute     exec wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle

bindsym --locked XF86MonBrightnessUp   exec brightnessctl set 5%+
bindsym --locked XF86MonBrightnessDown exec brightnessctl set 5%-

bindsym XF86AudioPlay exec playerctl play-pause
bindsym XF86AudioNext exec playerctl next
bindsym XF86AudioPrev exec playerctl previous
```

`--locked` is what makes them work while the screen is locked, which is the
whole point of media keys.

`-l 1.0` caps the volume at 100%. Without it, repeated presses happily push
past unity into distortion — `pactl` has no equivalent, so this is one place
the native tool is genuinely better.

**`pactl` is the other option and it works**, since pipewire-pulse provides
the interface — but the binary lives in `pulseaudio-utils`, which
`pipewire-pulse` only *Suggests*. Suggests are never installed, not even
with recommends enabled, so `pactl` bindings on a fresh machine fail
silently. If you prefer them, install that package deliberately.

### No blue light filter

Two tools, both in Debian, both tiny. Pick one — wlroots hands
`wlr-gamma-control` to a single client at a time, so running both means the
second one silently gets nothing.

**gammastep is the one in `extras desktop`**, for two capabilities wlsunset
does not have at all — not for the tray indicator, which on sway costs you
a tray (§10) and is no advantage here:

| | gammastep | wlsunset |
|---|---|---|
| Automatic shift by sunset | yes | yes |
| Fixed times instead of coordinates | `dawn-time`/`dusk-time` | `-S`/`-s` |
| **Set a temperature on demand** | **`-O TEMP`** | no — daemon only |
| **Reduce brightness at night** | **`-b DAY:NIGHT`** | no |
| Config file | `-c FILE` | no, flags only |

`-O` is what makes a keybinding possible: wlsunset has no manual mode, so
"warm the screen now" is not something it can do at all. `-b` dims in
software below the backlight's minimum, which on a laptop panel in a dark
room is the difference that actually matters.

The usual arguments against gammastep — geoclue, the missing systemd unit —
are arguments against its *defaults*, not against the program. Set
`location-provider=manual` or fixed times and start it from your sway
config, and neither applies.

wlsunset remains the better answer if all you want is the automatic shift:
one line, no config file, nothing to maintain.

```bash
# gammastep — config file below, plus keybindings for on-demand shifts
exec gammastep

# wlsunset — the whole setup, if you want nothing else
exec wlsunset -l 52.5 -L 13.4 -t 3800 -T 6500
```

`-l` is latitude and `-L` is longitude; transposing them is the usual
mistake, and a longitude in the latitude slot gives you sunset times for
somewhere you are not. wlsunset also takes fixed times instead, which
removes location handling entirely:

```bash
exec wlsunset -S 07:00 -s 21:00 -t 3800 -T 6500
```

**Three Debian-specific traps:**

- **`redshift` is not an option here.** It is the X11 original that
  gammastep forked from. Its `randr` method finds nothing under sway and its
  `drm` method fights the running compositor. It installs cleanly and does
  nothing, which is the worst possible failure.
- **Debian's gammastep package ships no systemd user unit**, so the
  `systemctl --user enable --now gammastep` you will find in every online
  guide fails with "Unit gammastep.service not found". Start it from your
  sway config instead.
- **Do not use gammastep's `geoclue2` location provider.** It needs the
  geoclue daemon *and* an allowlist entry in `/etc/geoclue/geoclue.conf`,
  and without that it gets no location and does nothing. `manual` with
  coordinates is one line and needs no daemon.

`~/.config/gammastep/config.ini`, with fixed times so there is no location
to configure and nothing to correct if you travel. `dawn-time` and
`dusk-time` must both be set or neither:

```ini
[general]
temp-day=6500
temp-night=3800
brightness-day=1.0
brightness-night=0.85
fade=1
adjustment-method=wayland
dawn-time=6:30-7:45
dusk-time=20:15-21:30
```

For real sunset times instead, drop the two time lines and add a location.
Manual coordinates, never geoclue:

```ini
location-provider=manual

[manual]
lat=52.5
lon=13.4
```

`brightness-night` is the setting people miss. Valid range is 0.1 to 1.0;
below roughly 0.7 the screen starts to look murky rather than dim.

For on-demand shifts, either tool's one-shot mode makes a good keybinding
and exits immediately, so it does not conflict with a running daemon:

```
bindsym $mod+Shift+n exec gammastep -O 3500
bindsym $mod+Shift+m exec gammastep -x
```

**When it does nothing at all**, check that the binary exists before
anything else. `exec` failures in a sway config go to sway's stderr, which
disappears the moment sway takes over the tty — so an `exec` line for a
package you have not installed fails completely silently, and a config
carried over from another machine is the usual way that happens.

```bash
command -v wlsunset gammastep
pgrep -a 'wlsunset|gammastep'     # is something else already holding gamma?
swaymsg -t get_config | grep -n 'wlsunset\|gammastep'
```

### Screen never locks / locks during video

`swayidle` has no idea what you're doing. Standard config:

```
exec swayidle -w \
  timeout 300 'swaylock -f' \
  timeout 600 'swaymsg "output * power off"' \
  resume 'swaymsg "output * power on"' \
  before-sleep 'swaylock -f'
```

Do this in your sway config rather than as a systemd user unit: `sleep.target`
does not exist in the systemd *user* manager, and a unit cannot know
`WAYLAND_DISPLAY` ahead of time. `swayidle`'s `before-sleep` runs inside your
session, where both are known.

For video, use the idle-inhibit protocol — mpv supports it natively; browsers
do it for fullscreen video only.

---

## 5. ThinkPad T14 Gen 3 specifics

### Firmware you must have

- **`firmware-sof-signed`** — the Sound Open Firmware DSP blob. Without it you
  have **no audio at all**, and nothing obviously errors. This is the single
  most common "my new ThinkPad has no sound on Linux" cause.
- **`firmware-iwlwifi`** — the AX211 Wi-Fi card.
- **`intel-microcode`** — CPU errata fixes, applied at boot.

### Firmware updates

Lenovo publishes to LVFS properly, and there have been real fixes for this
generation (thermal behaviour, USB-C/Thunderbolt, battery reporting).

```bash
sudo fwupdmgr refresh
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

Do this early and then roughly quarterly. Keep the machine plugged in.

### Suspend: the Modern Standby problem

Check what your kernel offers:

```bash
cat /sys/power/mem_sleep
```

Intel T14 Gen 3 supports **s2idle only** — there is no S3 deep sleep. This is a
firmware decision, not a Linux limitation. Practical consequences:

- Suspend drains battery faster than on older ThinkPads. Losing 1–2% per hour
  is normal and healthy; 5–10% per hour means something is keeping the system
  awake.
- Diagnose with `sudo dmesg | grep -i "PM: suspend"` and check the s0ix
  residency: `sudo turbostat --show Pkg%pc/PkgWatt --quiet sleep 10`.
- The usual culprits are a USB device holding a wakeup source, or Thunderbolt.
  `cat /proc/acpi/wakeup` shows what can wake the machine.

**If suspend drain bothers you**, hibernation is the answer — but it requires
swap ≥ 32 GB, a `resume=UUID=...` kernel parameter, and Secure Boot lockdown
may block it. Consider this a project, not a checkbox.

### Power management

Install TLP, and then **leave it alone**. The default configuration is good.

Critical: TLP and `power-profiles-daemon` conflict. Only one may be active.
The bootstrap script disables PPD. Verify:

```bash
sudo tlp-stat -s
```

Also worth setting once — battery longevity thresholds, which the ThinkPad
firmware supports natively:

```
# /etc/tlp.d/01-battery.conf
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
```

If the laptop mostly lives on a desk, this materially extends battery lifespan.

### Trackpoint and touchpad

Sway needs explicit input config; there's no settings panel:

```
input type:touchpad {
    tap enabled
    natural_scroll enabled
    dwt enabled            # disable while typing
    scroll_method two_finger
}
input type:pointer {
    accel_profile flat
}
```

Find device identifiers with `swaymsg -t get_inputs`.

### Docking

`kanshi` is the piece that makes this painless. It watches for output changes
and applies a matching profile:

```
# ~/.config/kanshi/config
profile laptop {
    output eDP-1 enable scale 1 position 0,0
}
profile docked {
    output eDP-1 enable position 0,1080
    output "Dell Inc. DELL U2720Q ABC123" enable mode 3840x2160 scale 1.5 position 0,0
}
```

Get exact output names from `swaymsg -t get_outputs`.

**USB-C dock pitfall:** some docks need `thunderbolt` device authorisation.
`boltctl list` shows pending devices; `boltctl enroll <uuid>` authorises
permanently.

### Fingerprint reader

Depends on the exact sensor. `lsusb | grep -i synaptics` then check whether
`libfprint` supports it. Even when it works, integrating it with `swaylock`
requires PAM configuration and is a common source of lockouts. My advice:
skip it. The security benefit over a good passphrase is marginal.

---

## 6. Staying fresh without touching the base

This is the half of the strategy that makes "boring base" viable. Three
mechanisms, in order of how much I'd reach for them.

### Flatpak — for GUI applications

Best for the browser, chat clients, anything with a big dependency tree you
don't want in your base system.

```bash
flatpak install flathub org.mozilla.firefox
```

**Pitfalls:**
- Flatpak apps run sandboxed and don't see your themes by default. Install a
  matching GTK theme runtime, or use `flatseal` to grant filesystem access.
- They don't inherit your environment file, so Wayland flags must be set via
  `flatpak override --socket=wayland`.
- First install pulls a large runtime (~1 GB). Subsequent apps share it.

### Nix — for CLI tooling

The better answer for `neovim`, language servers, compilers, anything you want
current and per-project.

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

Then `nix profile install nixpkgs#neovim`, or better, use `nix develop` /
`direnv` for per-project toolchains so nothing is globally installed at all.

**Pitfalls:**
- The Nix store lives in `/nix` and grows. `nix-collect-garbage -d`
  periodically, or it will quietly eat 20 GB.
- Nix-installed binaries linked against Nix's glibc won't load system
  libraries. This is by design, but it surprises people mixing the two.
- Nix's systemd units survive Debian upgrades fine, but the daemon install
  writes to `/etc/bash.bashrc` — check that after a release upgrade.

### Backports — use sparingly

For when you need one newer thing from Debian itself, e.g. a newer kernel for
hardware support.

```bash
sudo apt install -t trixie-backports linux-image-amd64
```

**Critical rule:** backports are opt-in only. Never set a global pin that pulls
everything from backports. Install specific packages with `-t`.

### The thing not to do: Frankendebian

Do **not** add testing or sid to your sources and pin. It appears to work,
then, months later, a libc transition partially applies and you have an
unbootable system with no clean recovery. This is the single most common way
people destroy a Debian install. If you find yourself wanting many packages
from sid, you wanted a different distro — which brings us back to the earlier
conversation.

---

## 7. Maintenance

The whole point is that this is short.

### Routine

```bash
sudo apt update && sudo apt upgrade
```

Every week or two. In stable this is almost entirely security patches; it will
not break anything and will not ask you questions. Reading changelogs is
optional here in a way it never was on Arch.

Worth installing:

- **`unattended-upgrades`** — applies security updates automatically. Configure
  it for `${distro_id}:${distro_codename}-security` only, not general updates.
- **`needrestart`** — tells you which services (and whether the kernel) need a
  restart after an upgrade. On by default in Debian; make sure it's not
  silenced.
- **`apt-listchanges`** — shows important NEWS entries before applying.

### Knowing what you actually installed

With a minimal system this is genuinely useful — it's your build manifest:

```bash
apt-mark showmanual > ~/dotfiles/packages-manual.txt
```

Commit that alongside your dotfiles. Combined with the bootstrap script, it
means rebuilding this laptop from scratch is a 30-minute operation.

Cleaning up:

```bash
sudo apt autoremove --purge      # orphaned auto-installed deps
sudo apt clean                   # cached .debs
```

### Security tracking

```bash
sudo apt install debsecan
debsecan --suite trixie --format detail
```

Overkill for a laptop, but worth running once so you understand what
"Debian stable is old therefore insecure" actually means: it doesn't. Security
fixes are backported to the frozen versions.

### Point releases

Every couple of months Debian issues a point release (13.1, 13.2…). Nothing to
do — `apt upgrade` handles it. No reinstall, no reboot beyond the kernel.

### Release upgrade — roughly every 2 years

This is the one event that requires attention. When Forky (14) becomes stable:

1. **Snapshot or back up first.** If you chose btrfs, take a snapshot. If not,
   a `borg` or `restic` backup of `/etc` and `/home` at minimum.
2. Fully upgrade the current release first: `sudo apt update && sudo apt full-upgrade`
3. Remove non-Debian sources temporarily (backports, third-party repos).
4. Edit `/etc/apt/sources.list.d/debian.sources`, replace `trixie` → `forky`.
5. Two-phase upgrade — this ordering matters:
   ```bash
   sudo apt update
   sudo apt upgrade --without-new-pkgs
   sudo apt full-upgrade
   ```
6. Reboot. Then `sudo apt autoremove --purge`.
7. Read the release notes for that release. Genuinely — this is the one time
   it pays off, they document every breaking change.

Budget an afternoon. Do it a month or two after release, not on day one.

---

## 8. Editors

### Check the packaged Emacs first

Debian's `emacs-pgtk` is built with native compilation, tree-sitter, and pure
GTK. That last one matters here: the `emacs-gtk` package uses the X11 frontend
and runs under XWayland, which looks soft on any scaled output and doesn't do
Wayland input properly. `emacs-pgtk` is the Wayland-native build.

```bash
apt policy emacs-pgtk
```

If the version is acceptable, that's the end of it. Note that `emacs-common`
does **not** include the manuals — those are `emacs-common-non-dfsg` and live
in `non-free`, because the GFDL isn't DFSG-compatible.

### Building from source

Use `build-emacs`. The things it exists to get right:

- **`deb-src` must be enabled** or `apt build-dep emacs` silently does nothing.
  Trixie's deb822 format needs `Types: deb deb-src`.
- **`libgccjit` must match your system GCC major version.** Mismatch produces a
  runtime load failure with a confusing message, not a build error. The script
  reads `gcc -dumpversion` and installs the matching `-dev` package.
- **pgtk is GTK3**, so the dependency is `libgtk-3-dev`. There is no GTK4
  Emacs frontend.
- **`--with-native-compilation=aot`** costs 20–40 minutes on this CPU but does
  all the work up front. The default (`yes`) compiles lazily in the background
  during your first sessions — that's the "why is Emacs pegging a core"
  complaint people have after a source build.
- **Install into a Stow prefix, never bare `/usr/local`.** `make install`
  straight into `/usr/local` scatters files that no tool tracks, and removing
  them later is guesswork. With Stow, uninstall is `stow -D emacs-30.2` and
  multiple versions coexist.

```bash
./build-emacs 30.2
./build-emacs --list-installed
./build-emacs --uninstall 30.2
```

`/usr/local/bin` precedes `/usr/bin` in Debian's default PATH, so a source
build shadows the packaged one without conflicting. Keeping `emacs-pgtk`
installed as a fallback is sensible.

### Two source-build gotchas

**The systemd unit lands in the wrong place.** Upstream installs
`emacs.service` under your prefix, where the systemd user manager never looks.
The script writes a corrected one to `~/.config/systemd/user/` with an absolute
`ExecStart`.

**`eln-cache` is keyed to the build.** Every rebuild invalidates it, and the
old one is not cleaned up — it just accumulates in `~/.emacs.d/eln-cache`
(or `~/.config/emacs/eln-cache`). The script offers to clear it.

### Language servers

The shared set across Emacs and Neovim, and where each one comes from on
Debian — the split matters, because three different mechanisms are involved:

| Server | Source | Command |
|---|---|---|
| `clangd` | apt | in the `lsp` group |
| `lua-language-server` | apt if packaged, else upstream tarball | see below |
| `texlab` | apt if packaged, else cargo | `cargo install --locked texlab` |
| `ty` | uv | `uv tool install ty@latest` |
| `ruff` | uv | `uv tool install ruff@latest` |
| `rust-analyzer` | rustup | `rustup component add rust-analyzer` |

`./extras lsp` installs what apt has, then detects the gaps and offers the
fallback for each. Debian stable does not dependably package `texlab` or
`lua-language-server`, so don't assume apt covered them — check.

**Remove `pylsp` if you have it.** `ty` replaces `python-lsp-server`, and
running both leaves you with duplicated diagnostics and confusing completion
from two servers attached to the same buffer. The hook checks both dpkg and
pipx and offers to purge. Nothing in these scripts installs it, but an older
Emacs config might have.

**`ty` and `ruff` deliberately come from uv, not apt.** This is the same
"boring base, fresh where it matters" pattern from §6 — both move fast, both
are single static binaries, and `uv tool upgrade --all` keeps them current
without touching the system.

**Tree-sitter grammars are compiled on demand**, so Emacs needs `git` plus a
working C/C++ toolchain and linker at runtime, not just at build time. The
`lsp` group pulls in `build-essential` and `git` for exactly this reason — if
you install the `cpp` group you already have them, but the `lsp` group
shouldn't assume that.

**The failure you'll actually hit is PATH.** `uv` installs to `~/.local/bin`
and `rustup` to `~/.cargo/bin`. Emacs launched from a graphical session
inherits PATH from your login shell, so verify before debugging eglot:

```bash
command -v ty ruff rust-analyzer clangd texlab lua-language-server
```

Any blank line there is your problem, not your Emacs config. (§10 covers why
`~/.local/bin` may be missing from PATH after a fresh install.)

No Swift or SourceKit dependency exists on Linux — that part of a macOS
configuration has no equivalent here.

### Neovim

Don't compile it. Debian's version lags upstream, but the Nix layer from §6
closes that gap with no build step and keeps it closed:

```bash
nix profile install nixpkgs#neovim
```

This is precisely the case the "boring base, fresh where it matters" strategy
was designed for. Compiling Neovim to chase releases reintroduces exactly the
maintenance burden you left Arch to avoid.

---

## 9. Development: native or distrobox?

**Short answer: native for C/C++ and Python, distrobox for matching your
Ubuntu servers.**

### Why native wins for local development

Distrobox is excellent, but it solves a problem you mostly don't have here.
Debian Trixie's toolchain is current enough — GCC 14, Clang 19, CMake 3.31,
Python 3.13 — that containerising to get newer compilers buys little. Against
that, containers cost you:

- **Debugging friction.** `gdb` and `perf` need `ptrace`, and rootless podman
  restricts it by default. Workable with `--cap-add=SYS_PTRACE`, but it's one
  more thing to remember when you're already annoyed at a segfault.
- **clangd indexing.** Your editor runs on the host and the toolchain runs in
  the container, so `compile_commands.json` contains container paths your
  language server can't resolve. Solvable, tedious.
- **Shared `$HOME` by default.** This is distrobox's best feature and its worst
  footgun: `~/.cargo`, `~/.cache/pip`, and `~/.local/lib` are shared between
  host and container, so a build artifact compiled against container glibc can
  end up on the host's path.

### Where distrobox genuinely earns its place

**You run two Ubuntu servers.** That's the real use case: build and test
against the same libraries your deployment target has, without touching your
laptop's base system.

```bash
distrobox create --name ubuntu-srv --image ubuntu:24.04
distrobox enter ubuntu-srv
```

Also good for: pinning an old toolchain for one legacy project, testing a build
on a distro you don't run, and isolating anything that wants to scatter files
system-wide. If you use it seriously, isolate the home directory:

```bash
distrobox create --name build-el9 --image rockylinux:9 \
  --home ~/.local/share/distrobox/build-el9
```

### C/C++ toolkit

The `cpp` group in `extras`. Beyond the obvious, three worth knowing:

- **`ccache`** — cuts rebuild times substantially. Enable per project with
  `-DCMAKE_CXX_COMPILER_LAUNCHER=ccache`.
- **`bear`** — generates `compile_commands.json` for Makefile projects that
  don't produce one: `bear -- make`. This is what makes clangd work on
  autotools codebases. CMake emits it natively with
  `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`.
- **`clangd`** over ccls. Both Emacs (eglot/lsp-mode) and Neovim speak LSP to
  it directly.

### Python: the one rule

Debian's system Python is marked **externally-managed** (PEP 668). `pip
install` outside a virtualenv refuses, deliberately, because apt owns those
files.

```bash
python3 -m venv .venv && . .venv/bin/activate   # per project
pipx install <cli-tool>                          # isolated CLI tools
```

**Never use `--break-system-packages`.** It does exactly what it says, and the
damage surfaces later as unrelated apt failures. If you find yourself reaching
for it, you wanted a venv.

Strong recommendation: `pipx install uv`. It handles virtualenvs, Python
version installs, and dependency resolution, an order of magnitude faster than
pip, and it sidesteps the system Python entirely.

For C extension work — pybind11, cffi, Cython — you need `python3-dev` for the
headers. It's in the `python` group.

---

## 10. Applications

### The tray problem — read this first

**Sway has no system tray.** Nextcloud client, Cryptomator, and the Mullvad VPN
GUI are all tray-driven: close the window and they become unreachable. Waybar
implements StatusNotifierItem, so enable its tray module before installing any
of them:

```json
"modules-right": ["tray", "pulseaudio", "network", "battery", "clock"],
"tray": { "spacing": 10, "icon-size": 18 }
```

Symptom if you skip it: the app launches, shows nothing, and appears to have
failed silently.

### From Debian's archive

| App | Package | Note |
|---|---|---|
| LibreOffice | `libreoffice-writer` etc. | **Also install `libreoffice-gtk3`** |
| GIMP | `gimp` | Check `apt policy gimp` for the major version |
| Nextcloud | `nextcloud-desktop` | Needs the tray |

`libreoffice-gtk3` is the one people miss. Without it LibreOffice falls back to
its own toolkit rendering — no native decorations, wrong scrolling, poor
Wayland behaviour. Install the specific components rather than the
`libreoffice` metapackage, which drags in Base, Draw, Math and a JRE.

### Tailscale — official Debian repo

Native packages, keyed per release codename. `install-thirdparty tailscale`
detects your codename from `/etc/os-release` and verifies the repo exists
before writing anything.

```bash
sudo tailscale up --operator=$USER
```

Use `--operator`, or every `tailscale status` afterwards needs sudo.

---

#### The baseline: reaching your own machines

This is all you need for "get to my servers and the services on them," and it
is the default state after `tailscale up`. No exit node, no extra flags.

Any machine running Tailscale is reachable by its MagicDNS name the moment
it's online:

```bash
tailscale status                  # what's in the tailnet
ssh user@myserver                 # MagicDNS name — no IP, no VPN toggle
curl http://myserver:8080         # a service on that machine
tailscale ping myserver           # is it direct, or relayed via DERP?
```

`tailscale ping` is the diagnostic worth knowing: it tells you whether the
connection is a direct WireGuard path or falling back to Tailscale's DERP
relays. Relayed connections work but are slower; the usual cause is a
restrictive NAT or firewall at one end.

**Subnet routers — probably the piece you actually want.** If some services
live on devices that *don't* run Tailscale (a NAS, a printer, a router admin
UI, containers on a bridge network), one machine on that LAN can route for the
whole subnet:

```bash
# On the server that lives on that LAN:
sudo tailscale up --advertise-routes=192.168.1.0/24
```

Approve the route in the admin console (Machines → server → Edit route
settings), then on the laptop:

```bash
sudo tailscale up --accept-routes
```

**`--accept-routes` is not the default on Linux.** This is the single most
common "subnet routing doesn't work" cause: the route is advertised, approved,
visible in the admin console, and still does nothing, with no error message
anywhere. Mobile and desktop GUI clients accept routes by default; the Linux
CLI does not.

The server also needs IP forwarding for subnet routing, same as for an exit
node:

```bash
printf 'net.ipv4.ip_forward = 1\nnet.ipv6.conf.all.forwarding = 1\n' \
  | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

**Watch for subnet collisions.** If your home LAN and a café network both use
`192.168.1.0/24`, routing that subnet will conflict while you're on the café
wifi. Renumbering your home LAN to something unusual (`192.168.73.0/24`) avoids
a class of confusing breakage. This is worth doing before you rely on it.

**Optional but useful:** `tailscale serve` puts a local service behind an HTTPS
name with a real certificate, inside the tailnet only:

```bash
tailscale serve --bg 8080         # https://myserver.tailnet-name.ts.net
```

Useful for anything that complains about running without TLS. `tailscale
funnel` is the same thing exposed to the public internet — deliberately not
what you want here.

---

### Optional: exit nodes

**You don't need any of this for the baseline above.** Skip it unless you
specifically want to route *all* internet traffic through another machine.
Recorded here for when the question comes up.

#### Exit nodes: what they actually are

By default **Tailscale is not a VPN in the "hide my traffic" sense.** Only
tailnet traffic (the `100.64.0.0/10` range) goes through WireGuard. Everything
else — your browsing, your updates — leaves through your normal interface,
exactly as if Tailscale weren't running. This surprises people who install it
expecting café-wifi protection and don't get it.

An **exit node** changes that. A device advertises willingness to route
`0.0.0.0/0` and `::/0`. When you select it, *all* your internet traffic goes
encrypted to that node, which NATs it out to the internet. Your apparent source
address becomes the exit node's.

Two rules that shape every decision below:

- **One exit node at a time.** You cannot use your own server and a Mullvad
  node simultaneously. It's a single global setting per device.
- **It's all-or-nothing per device.** Exit nodes don't do per-app or per-domain
  routing. (Tailscale's *app connectors* do that, but they're a separate
  feature aimed at routing specific SaaS domains through a known egress IP.)

---

#### Strategy A: your own server as exit node

You have two Ubuntu servers, so this costs nothing.

**On the server:**

```bash
sudo tailscale up --advertise-exit-node
```

Advertising alone is not enough — the kernel must forward packets, or the node
will announce the route and then silently drop everything:

```bash
printf 'net.ipv4.ip_forward = 1\nnet.ipv6.conf.all.forwarding = 1\n' \
  | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Then **approve it in the admin console** (Machines → the server → Edit route
settings → Use as exit node). Exit nodes require explicit approval; this is a
deliberate safety step, not an oversight. To skip it for future nodes, set
`autoApprovers` in your tailnet policy file.

**Throughput matters here.** An exit node terminates and re-encrypts everything
you do, so its CPU becomes your internet speed. On Linux, enable UDP GRO
forwarding on the server's physical interface — without it you can lose a large
fraction of achievable throughput:

```bash
NIC=$(ip route get 1.1.1.1 | awk '{print $5; exit}')
sudo ethtool -K "$NIC" rx-udp-gro-forwarding on rx-gro-list off
```

That doesn't persist across reboots; put it in a `networkd-dispatcher` hook or
a small systemd unit.

**On the laptop:**

```bash
tailscale exit-node list
tailscale set --exit-node=<server-name> --exit-node-allow-lan-access
tailscale set --exit-node=                      # turn it off
```

`--exit-node-allow-lan-access` is the flag everyone forgets. Without it, while
the exit node is active you lose your local printer, NAS, and router web UI,
because your entire local subnet is being routed through the tunnel.

**What this buys you:** your traffic is encrypted across the untrusted local
network, you egress from a stable known address, and you can reach anything
local to that server.

**What it does not buy you:** anonymity. You're egressing from *your own*
server, which is registered to you. Against a network operator or a snooping
café, it's excellent. Against the sites you visit, it identifies you *more*
consistently than a normal ISP address would. And if the server is a cloud VPS,
you inherit that provider's reputation — datacenter IP ranges collect CAPTCHAs
and outright blocks from Cloudflare, streaming services, and some banks.

---

#### Strategy B: Mullvad exit nodes through Tailscale

Bought through Tailscale, not Mullvad. The add-on costs $5/month per unit covering up to 5 devices, and it is available on every plan — including the Free plan. In the admin console: General settings → Mullvad VPN → Configure, then assign device access.

**How it works, technically.** This is the detail that makes the integration
interesting: your node registers its existing Tailscale-generated WireGuard key pair with Mullvad's infrastructure, traffic is terminated at Mullvad's network edge, and end-to-end encrypted all the way to your device. There is no second tunnel, no second daemon, no second routing table. Mullvad's servers in 40+ countries simply appear as nodes in your tailnet.

That is precisely why this doesn't conflict with anything, whereas running the
Mullvad *app* alongside Tailscale does.

**Using it:**

```bash
tailscale exit-node list                    # Mullvad nodes appear here
tailscale set --exit-node=de-ber-wg-001.mullvad.ts.net --exit-node-allow-lan-access
tailscale exit-node suggest                 # let Tailscale pick one
```

One quirk worth knowing: Tailscale picks a recommended standard exit node using latency and performance data, but because it has no performance information for Mullvad exit nodes, it recommends one based on location instead. So `suggest` is less clever for Mullvad nodes — check the result rather than trusting it.

**Assigning access.** You can manage which devices may use Mullvad through the
admin console UI *or* through the tailnet policy file, but not both — if you use the policy file, you must manage all Mullvad access there. The policy file is more flexible: it lets you assign Mullvad access to more devices than you have licenses for (the license caps concurrent use, not assignment).

---

#### The honest privacy comparison

This is where people get it wrong, so be explicit about it.

**Mullvad app, used directly:** you have an account number. No email, no name,
no identity provider. Mullvad sees your real IP and your traffic. Nobody else
is involved.

**Mullvad through Tailscale:** you log into Tailscale with an identity provider
(Google, GitHub, Microsoft…). Tailscale's control plane knows who you are, what
devices you own, and which exit node each one selected. Mullvad still sees your
traffic. You have added a party that knows both your real identity *and* your
exit-node choices.

**So: the Tailscale integration is strictly worse for anonymity than using
Mullvad directly.** It is not a privacy upgrade. What it is, is a *convenience
and architecture* upgrade — one WireGuard stack, one client, one bill, no
routing conflicts, and your tailnet stays reachable while you're "on VPN."

You also lose Mullvad's own client features, which are not part of the
integration:

- **DAITA** (their traffic-analysis defence)
- **Multihop** (entry and exit in different countries)
- **Mullvad's DNS-level ad and tracker blocking**
- **The kill switch / lockdown mode**

If your threat model is "my ISP and the sites I visit should not be able to
identify me," use the Mullvad app directly and don't run Tailscale at the same
time. If your threat model is "I don't want café wifi reading my traffic, I want
my servers reachable, and I'd sometimes like to appear to be in another
country," the Tailscale integration is the better engineering choice.

---

#### Choosing between them

| Need | Use |
|---|---|
| Reach your servers from anywhere | Tailscale alone (no exit node) |
| Protect traffic on hotel/café wifi | Own server as exit node |
| Stable, known egress IP | Own server as exit node |
| Appear to be in another country | Mullvad exit node |
| Egress not attributable to you | Mullvad exit node |
| Genuine anonymity | Mullvad app directly, without Tailscale |

**Start with no exit node at all.** Reaching your own machines needs none, and
adding one changes how every packet on the laptop is routed — which is a lot of
new failure surface to take on at the same time as a new OS install. Get the
baseline solid first.

Later, if you want café-wifi protection, advertising your own server takes
about five minutes. The $5 Mullvad add-on takes about one. Neither is a
decision you need to make now, and neither is hard to reverse.

---

#### The kill switch you don't have

Neither exit-node strategy fails closed. If `tailscaled` stops while an exit
node is set, traffic reverts to your normal route silently. The Mullvad app
blocks traffic in that situation; Tailscale does not.

For a laptop on hotel wifi, decide whether that matters. If it does, it has to
be firewall rules — sketch only, as an nftables policy permitting the tailnet
and the coordination path and nothing else:

```
table inet killswitch {
  chain output {
    type filter hook output priority 0; policy drop;
    oifname "tailscale0" accept
    oifname "lo" accept
    udp dport 41641 accept              # tailscale WireGuard
    ip daddr 100.64.0.0/10 accept       # tailnet CGNAT range
    meta skuid tailscaled accept        # daemon reaching the control plane
  }
}
```

Test this carefully before relying on it, and keep console access to the
machine while you do. A kill switch you haven't verified is worse than none,
because you will trust it. Note also that it will break captive portals — you
need a way to disable it before you can log into hotel wifi at all.

---

**MagicDNS needs a DNS manager.** With `systemd-resolved` active, tailscaled
uses the proper split-DNS API; without it, it rewrites `/etc/resolv.conf`
directly, which works but fights NetworkManager. A minimal Debian install often
has NetworkManager but *not* systemd-resolved — this is the most common cause
of "Tailscale is connected but names don't resolve." The script offers to
install it.

**For your two Ubuntu servers**, Tailscale SSH removes key management:

```bash
sudo tailscale up --operator=$USER --ssh
ssh user@server-name        # MagicDNS name, no keys, no config
```

Access is then governed by tailnet ACLs rather than `authorized_keys`.

**No official Linux GUI**, so no tray needed — the CLI is the interface. A
waybar custom module running `tailscale status --json` works if you want it
visible.

**One Trixie-specific bug:** there are reports of `tailscaled` failing to start
after `apt install` because `/etc/default/tailscaled` isn't created, making
`tailscale up` fail with a misleading "it doesn't appear to be running." The
script checks and creates the file if missing.

### SSH server — optional, and only if you pin it

Not installed by any script. Worth adding *only* bound to the tailnet:

```
# /etc/ssh/sshd_config.d/10-tailscale.conf
ListenAddress 100.x.y.z          # from `tailscale ip -4`
PasswordAuthentication no
PermitRootLogin no
```

The argument for it on a laptop is narrow but real: you have no display
manager and a WM you'll be iterating on, so a shell you can reach from your
phone when tty1 is showing a stack trace has genuine recovery value. Tailscale
gives the laptop a stable name from anywhere, which is what makes it practical
rather than a NAT puzzle.

With `ListenAddress` pinned, the daemon isn't listening on café wifi at all.
Use `ssh.socket` rather than `ssh.service` if you want it started on demand
only.

**Tailscale SSH** (`tailscale up --ssh`) is the alternative — no `sshd`, access
via tailnet ACLs instead of `authorized_keys`. It's the better fit on your
servers. On the laptop, the tradeoff is that if Tailscale is what's broken,
you've lost your recovery path, whereas a pinned `sshd` survives a `tailscaled`
restart.

What I'd argue against is a plain `apt install openssh-server` with no
`ListenAddress`: a permanent listening service on every hostile network you
join, for something you'd use a few times a year.

### Mullvad Browser — official Debian repo

Best case of the three: Mullvad self-hosts a signed apt repository covering
Debian, so this is a native package with normal apt updates. No tarball, no
Flatpak needed. `install-thirdparty` adds the repo and key.

One behaviour that looks like a bug and isn't: the browser resists
fingerprinting by reporting standardised window dimensions, which produces grey
letterboxing when tiled. Float it if that bothers you:

```
for_window [app_id="mullvad-browser"] floating enable
```

### Running AppImages on Debian

An AppImage is a self-contained binary, but it is **not** self-integrating:
nothing puts it on your PATH, in fuzzel, or in your icon theme. And there's one
Debian-specific trap.

**The libfuse2 problem.** Most AppImages in circulation are type-2, built
against libfuse2. Debian Trixie ships fuse3 by default and libfuse2 is a
separate package. Without it you get:

```
dlopen(): error loading libfuse.so.2
AppImages require FUSE to run.
```

which reads like a corrupt download. It isn't. Fix:

```bash
sudo apt install libfuse2t64      # 'libfuse2' on older releases
```

The name changed in the 64-bit `time_t` transition, which is why older guides
tell you to install a package that no longer exists.

If you'd rather not install a legacy library, every AppImage supports running
without FUSE at all:

```bash
./Foo.AppImage --appimage-extract-and-run
```

Slower to start, uses `/tmp`, always works.

**Integration.** Use `appimage-manage` from this bundle. Rather than
hand-writing a `.desktop` file, it extracts the one already inside the
AppImage — along with the real icon and `StartupWMClass` — and rewrites only
the `Exec` path:

```bash
./appimage-manage install ~/Downloads/Cryptomator-1.x.AppImage
./appimage-manage list
./appimage-manage update ~/Downloads/Cryptomator-1.y.AppImage cryptomator
./appimage-manage remove cryptomator
```

It installs to `~/.local/lib/appimages/`, symlinks into `~/.local/bin/`, and
drops the launcher into `~/.local/share/applications/`. fuzzel reads that
directory directly, so the app appears immediately.

Two notes:

- `~/.local/bin` is only added to PATH by Debian's `~/.profile` **if the
  directory existed at login**. On a fresh install you'll need to log out once.
- AppImages are almost always X11/XWayland, so sway window rules must match on
  `class`, not `app_id`. The script prints the correct rule for you.

**Alternatives I'd skip.** AppImageLauncher is the commonly recommended tool
but hasn't seen meaningful maintenance in years and isn't in Debian. Gearlever
(Flathub) is actively maintained and has a GUI if you'd prefer that. For two
AppImages, a 200-line script you can read beats either.



Not in Debian, and the official PPA is Ubuntu-only. Two options with a real
tradeoff:

- **AppImage** (upstream's recommendation) — full filesystem access, so vaults
  on external drives and other mount points work. No auto-updates.
- **Flatpak** (Flathub, community-maintained) — auto-updates and sandboxing,
  but users report removable drives and non-`$HOME` mount points being
  invisible, and the manifest declares X11 so it runs under XWayland.

If every vault lives under `$HOME` (e.g. inside your Nextcloud folder), Flatpak
is fine. If any vault is on an external disk, take the AppImage.

**The critical configuration point**, whichever you choose: point the Nextcloud
client at the **encrypted** vault directory, never at the unlocked mountpoint.
Syncing the decrypted view uploads your plaintext and defeats the entire
purpose. It's an easy mistake to make once and a bad one.

---

## 11. Diagnostic reference

Commands worth knowing before you need them:

```bash
# Session
echo $XDG_CURRENT_DESKTOP $XDG_SESSION_TYPE
loginctl show-session $(loginctl | grep $USER | awk '{print $1}') -p Type

# Sway
swaymsg -t get_outputs
swaymsg -t get_inputs
swaymsg -t get_tree | less

# Portals / screen sharing
journalctl --user -u xdg-desktop-portal -u xdg-desktop-portal-wlr -f

# Audio
wpctl status
pw-top

# Power
sudo tlp-stat -b          # battery, thresholds
sudo powertop             # what's consuming
cat /sys/power/mem_sleep

# Network — who owns the interface
nmcli device status
nmcli general status
pgrep -a wpa_supplicant
journalctl -u NetworkManager -b | tail -40

# Hardware
sudo lshw -short
sensors
sudo dmesg --level=err,warn

# Firmware
fwupdmgr get-devices

# apt forensics
apt-mark showmanual
apt-cache depends --recommends <pkg>
zcat /var/log/apt/history.log*.gz | less
```


### Wifi says `unmanaged`, or `unavailable`

The single most likely thing to bite you on this build, because nothing errors
when it goes wrong.

With every desktop task deselected, the Debian installer configures wifi
through **ifupdown**: the interface and the PSK go into
`/etc/network/interfaces`, and ifupdown starts its own `wpa_supplicant` for it.
That is what carries you through first boot. Then `bootstrap-sway` installs
NetworkManager — and Debian ships NM with

```
[ifupdown]
managed=false
```

so NM refuses to touch an interface ifupdown has claimed. Your wifi keeps
working, so you don't notice. You notice weeks later, when `nmtui` shows the
device as `unmanaged` and it reads like a driver or firmware fault.

Then the obvious fix has its own trap. Commenting the stanza out is *not*
enough: ifupdown's `wpa_supplicant` is still running and still holds the
netdev, so the device moves from `unmanaged` to `unavailable` — a different
symptom for the same unfinished handoff, and one that sends you off inspecting
rfkill and firmware for a fault that isn't there.

Do the whole handoff or none of it:

```bash
# 1. Which state are you actually in?
nmcli device status              # unmanaged | unavailable | disconnected
pgrep -a wpa_supplicant          # `-u` is NetworkManager's; `-i <iface>` is ifupdown's

# 2. Remove the WHOLE stanza — the auto/allow-hotplug line, the iface line,
#    and every indented option under it. A commented-out `iface` leaves its
#    wpa-ssid/wpa-psk lines orphaned, which is a malformed interfaces(5) file.
sudoedit /etc/network/interfaces
ls /etc/network/interfaces.d/    # and check here too, it is easy to miss

# 3. Release the device. Disabling the unit does not kill an orphan, because
#    ifupdown's hook scripts start the supplicant outside systemd.
sudo systemctl disable --now wpa_supplicant@wlp0s20f3.service
sudo pkill -f 'wpa_supplicant.*-i *wlp0s20f3'
sudo ifdown wlp0s20f3
sudo ip addr flush dev wlp0s20f3

# 4. Hand it over.
sudo systemctl restart NetworkManager
nmcli device status              # want: disconnected
nmtui                            # retype the password once
```

`bootstrap-sway` offers to do all of this at the end of its run, and it is
last in the script for a reason: it is the only step that can take the network
away, and everything before it needs the network.

Afterwards, `/etc/network/interfaces` should contain loopback and nothing else:

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback
```

Your credentials now live in `/etc/NetworkManager/system-connections/<SSID>.nmconnection`
(mode 600), written by `nmtui`. Nothing goes in `interfaces` by hand again.

If the device still says `unavailable` once NM owns it, the radio is down, not
the handoff:

```bash
rfkill list                # a hard block is Fn+F8 and must be cleared on the keyboard
nmcli radio wifi           # NM persists this in /var/lib/NetworkManager/NetworkManager.state,
                           # so an off written once stays off across reboots
dmesg | grep -i iwlwifi    # firmware, if it is genuinely the driver
```

---

## 12. What I'd deliberately skip

Advice is as much about omission. On this machine I would not bother with:

- **A display manager.** greetd/tuigreet is nice, but starting sway from tty1
  is one line in your shell profile and one fewer moving part. Append it to
  `~/.profile`, not a new `~/.bash_profile` — creating the latter stops bash
  reading the former, and `~/.profile` is what puts `~/.local/bin` on PATH.
  If you switch to zsh, that line has to move to `~/.zprofile`: zsh reads
  neither of the other two, and sway simply stops starting with nothing in
  any log to say why. `./extras shell` offers to mirror it for you.
- **The fingerprint reader.** Fiddly PAM configuration, real lockout risk,
  marginal gain.
- **Fractional scaling**, if you have the 1920×1200 panel. Scale 1 is correct
  and fractional scaling still causes blur in XWayland apps.
- **A full theming rabbit hole.** Set dark mode, set a font, move on. This is
  the classic way a "get out of my way" machine becomes a hobby.
- **OpenBSD on bare metal here.** As discussed — run it in the `virt` group's
  qemu, where it costs you nothing.

### GNOME as a fallback — deferred, not rejected

Worth recording the reasoning, since the question will come back.

The idea: install GNOME alongside sway, to log into when a tiling WM doesn't
give you something you need. Two reasons to not do it now:

- **Portal ambiguity.** With both `xdg-desktop-portal-gnome` and `-wlr`
  installed, backend selection becomes ambiguous and the failure mode is the
  silent one — screen sharing that produces a black frame with no error.
  `portals.conf` would have to be exactly right, permanently.
- **It reintroduces a display manager.** Making GNOME selectable means gdm or
  greetd owns your boot path, which is an architectural change rather than an
  addition.

And the non-technical cost: a fallback one login away means never pushing
through the friction of making sway good, so you maintain two half-configured
environments instead of one you know well.

**Separate the two things it might mean.** For *rescue* (broken config, no
usable desktop), a whole DE is overkill — you have a TTY, `sway -C` validates
a config without loading it, and a known-good `sway -c ~/.config/sway/rescue.conf`
covers it in ten lines. For *capability*, name the specific situations; most
resolve to one package rather than a desktop:

| "I need a DE for…" | Actually |
|---|---|
| Display arrangement | `wdisplays`, or `kanshi` for the automatic case |
| Bluetooth pairing | `blueman` |
| Printer setup | CUPS web UI at `localhost:631` |
| Audio routing | `pavucontrol` |
| Screen sharing | the wlr portal, already installed |

**Suggested approach:** keep a note for a month. Every time you think "a DE
would help here," write the specific thing down. Empty list after a month
answers the question. Three items, and they're probably three `apt install`
lines. Nothing in this setup blocks adding GNOME later — it's a reversible
decision, so there's no cost to deferring it.

---

## 13. Summary checklist

- [ ] Netinst image, checksum verified
- [ ] LUKS2 full disk encryption; swap sized for your hibernate decision
- [ ] tasksel: desktop environments deselected, "standard system utilities" KEPT
- [ ] Root password left blank (puts you in `sudo`)
- [ ] `non-free-firmware` present in sources
- [ ] Recommends left **enabled** (no `99norecommends` file present)
- [ ] `bootstrap-sway --compare` reviewed before installing
- [ ] Run `bootstrap-sway`
- [ ] **Log out and back in** for `video`/`input` groups
- [ ] Sway config includes `include ~/.config/sway/config.d/*`
- [ ] Launch via `start-sway`, not bare `sway`
- [ ] Verify: `echo $XDG_CURRENT_DESKTOP` → `sway`
- [ ] Verify: audio present (`wpctl status`), brightness keys work
- [ ] `fwupdmgr update`
- [ ] TLP active, power-profiles-daemon disabled
- [ ] Battery thresholds set if desk-bound
- [ ] Flatpak or Nix layer for browser / current tooling
- [ ] `apt-mark showmanual` committed to dotfiles
- [ ] `unattended-upgrades` configured for security only
- [ ] `./extras lsp`; then `command -v ty ruff rust-analyzer clangd texlab
      lua-language-server` returns a path for every one
- [ ] `pylsp` / `python-lsp-server` removed (ty replaces it)
- [ ] Waybar `tray` module enabled (before installing Nextcloud/Cryptomator)
- [ ] `libreoffice-gtk3` installed alongside LibreOffice
- [ ] Nextcloud pointed at the ENCRYPTED vault dir, not the unlocked mount
- [ ] Tailscale up with `--operator=$USER`; systemd-resolved active for MagicDNS
- [ ] `tailscale status` shows your servers; `ssh user@server` works by name
- [ ] If services live on non-Tailscale devices: subnet router advertised,
      route approved, and `--accept-routes` set on the laptop
- [ ] No exit node configured (not needed for reaching your own machines)
