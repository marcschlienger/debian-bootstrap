# Installation walkthrough

Debian stable (Trixie) + Sway on a ThinkPad T14 Gen 3.

This is the "do this" document. For *why* each choice was made and what breaks
when you get it wrong, see [`debian-sway-guide.md`](debian-sway-guide.md) —
section references below point into it.

**Time budget:** phases 0–4 are an evening. Phases 5–8 are a second session.
Don't compress them: verifying phase 4 properly is what stops you debugging
three layers at once later.

---

## Phase 0 — Before touching the laptop

- [ ] **Back up anything on the machine.** This wipes it.
- [ ] **Download the Debian Trixie netinst image** and verify the checksum
      against the signed `SHA256SUMS`. Firmware is included in the official
      images now — there's no separate unofficial image to hunt for.
- [ ] **Write it to a USB stick:**
      `dd bs=4M if=debian-13-netinst.iso of=/dev/sdX status=progress oflag=sync`
- [ ] **Get these scripts onto a second stick**, or plan to `git clone` them
      after first boot: `bootstrap-sway`, `extras`, `install-thirdparty`,
      `appimage-manage`, `build-emacs`.
- [ ] **Decide the hibernate question now** — it sets swap size. Hibernate
      means swap ≥ 32 GB; without it, 8 GB is plenty. Resizing later under
      LUKS is painful. → guide §1
- [ ] **Update firmware from BIOS** if Lenovo has an update. Easier now than
      later.

---

## Phase 1 — Debian installer

- [ ] Boot the USB, choose **Graphical install**.
- [ ] Connect to wifi — the AX211 works, firmware is on the image.
- [ ] **Partition manually:**

      /boot/efi    512 MB   FAT32, unencrypted
      /boot        1 GB     ext4, unencrypted
      remainder             LUKS2 → / (ext4 or btrfs) + swap per above

- [ ] **Leave the root password blank.** This disables root and puts your user
      in `sudo`. Set one, and you'll have to add yourself to `sudo` by hand
      from a root shell before anything works. → guide §1
- [ ] **tasksel: deselect every desktop environment. Leave "standard system
      utilities" CHECKED.** → guide §1
- [ ] Install GRUB to the EFI partition. Reboot, remove the USB.

---

## Phase 2 — First boot

- [ ] Log in at the TTY. You have almost nothing; that's expected.
- [ ] **Check apt sources.** `/etc/apt/sources.list.d/debian.sources` needs
      `Components: main contrib non-free-firmware`. → guide §2

      sudo apt update

- [ ] Get the scripts on the machine (mount the USB, or
      `sudo apt install git` and clone).
- [ ] `chmod +x bootstrap-sway extras install-thirdparty appimage-manage build-emacs`

---

## Phase 3 — The desktop

- [ ] **Preview before committing:**

      ./bootstrap-sway --compare

      Read the numbers. Confirm no desktop metapackage appears in the
      resolved set. → guide §2

- [ ] **Install:**

      ./bootstrap-sway --tty-autostart

- [ ] **Log out and back in.** Non-negotiable — `video`/`input` group changes
      don't apply to an existing session, and without them brightness keys
      fail silently. → guide §3
- [ ] **Put your dotfiles in place:** `~/.config/sway`, `~/.config/kitty`.
- [ ] **Your sway config must contain `include ~/.config/sway/config.d/*`** or
      the portal drop-in is ignored and screen sharing breaks with no error.
      → guide §3
- [ ] **Start sway** with `~/.local/bin/start-sway`, never bare `sway` — the
      wrapper is what loads the session environment. → guide §3

---

## Phase 4 — Verify before building on it

Run all of these. Fix anything that fails *now*.

- [ ] `echo $XDG_CURRENT_DESKTOP` → `sway`
- [ ] `wpctl status` → your audio card is listed (needs `firmware-sof-signed`)
- [ ] `brightnessctl s 50%` → no permission error
- [ ] `swaymsg -t get_outputs` → your panel
- [ ] **Firmware updates**, plugged in:

      sudo fwupdmgr refresh && sudo fwupdmgr update

- [ ] **Battery thresholds** if it mostly lives on a desk — `/etc/tlp.d/01-battery.conf`,
      start 75 / stop 80. → guide §5
- [ ] **Add waybar's `tray` module now**, before phase 6. Nextcloud and
      Cryptomator are tray-only and will look like they failed without it.
      → guide §10

---

## Phase 5 — Software

- [ ] **Look first:** `./extras --list`
- [ ] **Core:**

      ./extras cli editors lsp cpp python

- [ ] **Desktop and documents:**

      ./extras desktop office media docs sync

- [ ] **Containers**, for building against your Ubuntu servers:

      ./extras container

- [ ] **Rust toolchain** — needed for `rust-analyzer`, and for `texlab` if apt
      didn't have it:

      curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
      rustup component add rust-analyzer

- [ ] **Verify every language server resolves:**

      command -v ty ruff rust-analyzer clangd texlab lua-language-server

      A blank line is a missing server, not an Emacs problem. Re-run
      `./extras lsp` and take the fallback it offers. → guide §8

---

## Phase 6 — Network and applications

- [ ] **Tailscale:**

      ./install-thirdparty tailscale
      sudo tailscale up --operator=$USER
      tailscale status
      ssh user@yourserver

- [ ] **If services live on devices without Tailscale**, set up a subnet
      router — and remember `--accept-routes` on this laptop, which is *not*
      the default on Linux. → guide §10
- [ ] **Mullvad Browser:** `./install-thirdparty mullvad-browser`
- [ ] **Cryptomator:** download the AppImage, verify the signature, then
      `./install-thirdparty cryptomator` and choose AppImage. → guide §10
- [ ] **Point Nextcloud at the ENCRYPTED vault directory**, never the unlocked
      mountpoint.

---

## Phase 7 — Emacs

- [ ] **Check the package first:** `apt policy emacs-pgtk`. Debian's build
      already has native-comp, tree-sitter and pgtk. If the version is
      acceptable, you're done. → guide §8
- [ ] **Otherwise build:**

      ./build-emacs 30.2

      It pauses after `configure` — confirm pgtk, native-comp and tree-sitter
      all report yes before committing to 40 minutes.

---

## Phase 8 — Make it repeatable

- [ ] **Record what you chose:**

      apt-mark showmanual > packages-manual.txt

      Commit it alongside the scripts. That plus `bootstrap-sway` rebuilds
      this laptop in about half an hour.
- [ ] **Security updates:** install `unattended-upgrades`, configured for
      `-security` only. → guide §7
- [ ] **Freshness layer** when you want it — Nix for CLI tooling, Flatpak for
      a current browser. → guide §6

---

## Optional extras

Not installed by any script; add them if you want them.

### SSH server, bound to Tailscale

Worth it on a laptop *only* if you pin it to the tailnet. A recovery shell you
can reach from your phone is genuinely useful on a machine with no display
manager and a WM you're still iterating on.

```bash
sudo apt install openssh-server
```

```
# /etc/ssh/sshd_config.d/10-tailscale.conf
ListenAddress 100.x.y.z          # your tailnet IP, from `tailscale ip -4`
PasswordAuthentication no
PermitRootLogin no
```

```bash
sudo systemctl restart ssh
```

With `ListenAddress` pinned to the tailnet address, the daemon isn't listening
on café wifi at all. Combined with key-only auth, exposure is close to zero.

Two refinements:

- **`ssh.socket` instead of `ssh.service`** means no daemon running until a
  connection arrives. Marginal, but free.
- **Tailscale SSH** (`tailscale up --ssh`) is the alternative: SSH with no
  `sshd` at all, access governed by tailnet ACLs instead of `authorized_keys`.
  Fewer moving parts, and the better fit *on your servers*. The catch for the
  laptop is that if Tailscale itself is what's broken, you've lost the recovery
  path — a pinned `sshd` survives a `tailscaled` restart.

A plain `apt install openssh-server` with no `ListenAddress` is what I'd argue
against: a listening service on every hostile network you join, for a
capability you'd use a few times a year.

### Others

- **Bluetooth:** `sudo apt install blueman` (tray applet; needs waybar's tray)
- **Printing:** `sudo apt install cups`, then `localhost:631`.
  Install with recommends — see the guide's table of what breaks without them.
- **A rescue sway config:** keep a known-good `~/.config/sway/rescue.conf` and
  start it with `sway -c ~/.config/sway/rescue.conf` when your main config
  breaks. `sway -C` validates a config without loading it. → guide §12

---

## If something is wrong

Guide §11 has the full diagnostic command reference. The three failures that
account for most problems:

| Symptom | Cause |
|---|---|
| Screen sharing shows black, no error | `XDG_CURRENT_DESKTOP` not set before sway started, or the config.d drop-in isn't included |
| No audio at all | `firmware-sof-signed` missing |
| Brightness keys do nothing | Not in the `video` group, or haven't logged out since |
