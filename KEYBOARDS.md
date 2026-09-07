# Keyboard setup on Debian 13

The ThinkPad and external keyboards use the same final modifier layout through
different mechanisms:

- Kanata remaps only the ThinkPad's internal keyboard.
- QMK runs on the Keychron V3 Max ANSI and Q3 ANSI knob keyboards, so their
  layout also works with macOS, iPadOS, or another computer.

Both provide one-shot Caps/Control and Shift keys. QMK gives the Keychrons
this complete bottom-row order:

    Alt  Super  Control  Space  Control  Super  Right-Alt  Fn

Kanata leaves the ThinkPad's Fn key in its physical position and maps the
available modifier positions to the corresponding Alt, Super, and Control
order.

Right Alt remains ordinary because Sway uses `us(altgr-intl)`. Sway must not
also swap Alt and Super; the QMK and Kanata configurations already emit the
final modifier positions.

## Kanata for the ThinkPad keyboard

First deploy the Dotfiles repository as described in `INSTALL.md`. The Kanata
package supplies the configuration, systemd user unit, and udev rule source.

### Install the executable

Debian stable may not contain a sufficiently current Kanata. Download the
current ordinary Linux release from the [Kanata releases page](https://github.com/jtroo/kanata/releases),
choose the archive for `uname -m`, and verify its published SHA-256 checksum.
Do not use a `cmd_allowed` build.

For the x86-64 archive:

```sh
uname -m
unzip linux-binaries-*.zip
chmod 0755 ./kanata_linux_x64
sudo install -o root -g root -m 0755 ./kanata_linux_x64 /usr/local/bin/kanata
/usr/local/bin/kanata --version
```

Use the corresponding arm64 filename on an aarch64 machine.

### Grant access to input and uinput

```sh
getent group uinput || sudo groupadd --system uinput
sudo usermod -aG input,uinput "$USER"
sudo modprobe uinput
sudo install -o root -g root -m 0644 \
  ~/.config/kanata/linux/70-kanata-uinput.rules \
  /etc/udev/rules.d/70-kanata-uinput.rules
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=misc
```

Log out and back in before starting the user service; `usermod` cannot change
the groups of the existing session. Confirm that `id` lists both `input` and
`uinput`. Membership of `input` allows processes run by the account to read
keyboard events, so keep the account trusted.

### Validate the mapping and device filter

```sh
kanata --check --cfg "$HOME/.config/kanata/kanata.kbd"
sudo kanata --list
sudo kanata --cfg "$HOME/.config/kanata/kanata.kbd"
```

The Linux configuration excludes devices named exactly `Keychron V3 Max` and
`Keychron Q3`. Compare those strings with the names Kanata reports on this
machine and correct the configuration before enabling the service if they
differ. Hold physical left Control + Space + Escape to stop the foreground
test.

Test Caps, both Shift keys, the three modifiers on each side, Right Alt/AltGr,
ordinary key repeat, and the Sway shortcuts. Then enable the service:

```sh
systemctl --user daemon-reload
systemctl --user enable --now kanata.service
systemctl --user status --no-pager kanata.service
journalctl --user -u kanata.service -b --no-pager
```

The Linux executable does not support the macOS-only
`--release-grab-on-lock` option, and the supplied unit does not pass it. After
editing the configuration:

```sh
kanata --check --cfg "$HOME/.config/kanata/kanata.kbd" && \
  systemctl --user restart kanata.service
```

The Sway configuration intentionally retains `caps:ctrl_modifier` only as a
fallback for Caps when Kanata is stopped. It must not contain
`altwin:swap_lalt_lwin`. Reload Sway after changing that file.

## QMK for the Keychrons

QMK only has to be installed on one computer. Skip this Debian section if the
firmware is maintained from macOS.

Install QMK's CLI and toolchains through its official bootstrapper, reviewing
the downloaded script before running it:

```sh
sudo apt update
sudo apt install --no-install-recommends ca-certificates curl git ripgrep
curl -fsSL https://install.qmk.fm -o /tmp/install-qmk.sh
less /tmp/install-qmk.sh
sh /tmp/install-qmk.sh
qmk --version
```

Clone the personal source and the Keychron firmware tree, restore the tracked
keymaps, and install QMK's udev rules:

```sh
git clone git@github.com:marcschlienger/qmk.git ~/Repos/qmk
git clone --branch 2025q3 --recurse-submodules https://github.com/Keychron/qmk_firmware.git ~/Repos/qmk/keychron-qmk
cd ~/Repos/qmk/keychron-qmk
qmk setup -H "$PWD"

cp -R ../keymaps/keychron/v3_max/ansi_encoder/marc keyboards/keychron/v3_max/ansi_encoder/keymaps/
cp -R ../keymaps/keychron/q3/ansi_encoder/marc keyboards/keychron/q3/ansi_encoder/keymaps/

sudo util/install_udev.sh
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Reconnect the keyboard, then verify and compile both exact targets without
using root for the build:

```sh
qmk doctor
qmk list-keyboards | rg '^keychron/(v3_max|q3)/'
qmk compile -kb keychron/v3_max/ansi_encoder -km marc
qmk compile -kb keychron/q3/ansi_encoder -km marc
```

The checkout intentionally uses Keychron's fork. A `qmk doctor` warning that
the official repository is not configured as the `upstream` remote is not a
build failure. Compilation also reports `VIA_INSECURE is enabled` because the
personal keymaps preserve VIA support. Treat host-side VIA tools as trusted
software.

### Flash one keyboard at a time

The two boards expose the same generic STM32 DFU bootloader identity.
Disconnect the other Keychron before flashing so its bootloader cannot receive
the wrong firmware. Put the selected keyboard in Cable mode, connect it
directly by USB, and start only its matching command:

```sh
qmk flash -kb keychron/v3_max/ansi_encoder -km marc
qmk flash -kb keychron/q3/ansi_encoder -km marc
```

When the command reports that it is waiting, unplug that keyboard, hold
Escape, reconnect it, and release Escape after about two seconds. The reset
button below the space bar is the alternative. Leave the cable connected until
`dfu-util` reports `File downloaded successfully` and the command exits with
status zero.

VIA can preserve a dynamic keymap across a firmware flash. If the new layout
is not active, hold **Fn + J + Z** for about four seconds. The new layout puts
Fn on the far-right modifier; an older retained layout may still put it on the
key labelled Fn.

Test both Mac/Windows switch positions, all one-shot modifiers, Right Alt, Fn,
the encoder, USB, the V3 Max wireless modes, and an iPad. The personal QMK
repository contains the factory firmware recovery files and their checksums.
Do not flash Bluetooth or receiver firmware merely to change a keymap.
