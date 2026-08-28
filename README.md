# debian-bootstrap

Scripts and notes for building a **Debian stable (Trixie) + Sway** laptop that
you work on rather than maintain. Written for a ThinkPad T14 Gen 3 (Alder Lake);
everything except the hardware section is generic Debian.

The design bet, in three lines:

- **Base system boring and frozen** — Debian stable, no testing/sid mixing.
- **Freshness layered on top, only where it's needed** — Flatpak for the
  browser, which `extras` sets up; Nix for current CLI tooling, which it
  deliberately doesn't. See guide §6 and install it when you want it.
- **Minimality by package selection, not dependency suppression** — you install
  `sway`, not a desktop task, and Recommends stay on because Debian Policy
  treats them as strong dependencies.

## What's here

| File | What it does |
|---|---|
| [`INSTALL.md`](INSTALL.md) | The walkthrough. Phases 0–8, from USB stick to a machine you can stop thinking about. Start here. |
| [`debian-sway-guide.md`](debian-sway-guide.md) | The reference. Why each choice was made and what breaks when it's made differently. INSTALL.md links into its sections. |
| [`bootstrap-sway`](bootstrap-sway) | Installs the desktop: compositor, session plumbing, portals, audio, fonts, hardware bits. Generates the `start-sway` wrapper and the portal drop-in. |
| [`extras`](extras) | Optional software in named groups — `cli`, `editors`, `lsp`, `cpp`, `python`, `desktop`, `office`, `media`, `container`, and more. Nothing here is required for the desktop to work. |
| [`install-thirdparty`](install-thirdparty) | The applications Debian doesn't ship: Tailscale, Mullvad Browser, Cryptomator, VeraCrypt, yazi, Obsidian, Claude Code, Codex. Each uses a different channel, and the rationale — including what can and cannot be verified — is documented per app. The two vendor shell installers are labelled as unverified and are fetched, hashed and offered for reading rather than piped into a shell — with the better channels that exist for both written down, and why they weren't taken. |
| [`appimage-manage`](appimage-manage) | Puts an AppImage on `$PATH`, in fuzzel, and in your icon theme, using the `.desktop` file and icon already inside it. Also diagnoses the libfuse2 failure. |
| [`install-fonts`](install-fonts) | Fonts apt doesn't have: patched Nerd Fonts by name (72 families upstream), Protesilaos's Aporetic, and a sweep of a local directory of files and archives. Per-user, idempotent, and it flags bitmap fonts that fontconfig ignores by default. |
| [`default-apps`](default-apps) | Sets which application opens which file type, discovering desktop-file IDs rather than assuming them. Fixes the "PDFs open in GIMP" default. `--list` and `--dry-run` included. |
| [`deploy-dotfiles`](deploy-dotfiles) | Clones the dotfiles repo to `~/.dotfiles` and stows it, moving Debian's own `~/.bashrc`/`~/.profile` aside first so stow doesn't abort. Also fixes the `~/.ssh` permissions that git and FAT32 always flatten (`--ssh-perms` on its own). Idempotent; `--list`, `--dry-run` and `--delete` do what you'd expect. |
| [`build-emacs`](build-emacs) | Builds Emacs (pgtk, native-comp, tree-sitter) into a GNU Stow prefix, so versions stay trackable and removable. Check `apt policy emacs-pgtk` first — you may not need it. |

## Quick start

On a freshly installed Debian with every desktop environment deselected in
tasksel:

```bash
chmod +x bootstrap-sway extras install-thirdparty appimage-manage build-emacs deploy-dotfiles default-apps install-fonts
```

Look before you leap — this prints the full resolved package list, total
download and disk size, and the cost of Recommends on versus off:

```bash
./bootstrap-sway --compare
```

Then install, log out, and log back in (the `video`/`input` group changes don't
reach an existing session):

```bash
./bootstrap-sway --tty-autostart
```

Start the session through the wrapper, never with a bare `sway`:

```bash
~/.local/bin/start-sway
```

Then your dotfiles, and your software:

```bash
./deploy-dotfiles
```

`bootstrap-sway` already installed a terminal font and the bar's glyph
fallback. For anything else — more Nerd Fonts, Aporetic, or a folder of
fonts you already own:

```bash
./install-fonts --list-nerd
```


```bash
./extras --list
```

```bash
./extras cli editors lsp python desktop
```

The full sequence, with the verification steps that stop you debugging three
layers at once later, is in [`INSTALL.md`](INSTALL.md).

## Conventions

Every script:

- runs as your normal user, not root — `sudo` is used where it's needed, and
  running as root is refused;
- supports `--help`, and prints its own header comment as the help text;
- supports `--dry-run` (where it makes sense) so you can see the resolved
  package set before committing to it.

`bootstrap-sway` and `extras` also take `--lean` to drop Recommends everywhere.
It's there for completeness and argued against in guide §2.

## The failures worth knowing in advance

| Symptom | Cause |
|---|---|
| Wifi is `unmanaged`, or `unavailable`, in nmtui | The ifupdown → NetworkManager handoff wasn't finished. `bootstrap-sway` offers to do it; guide §11 explains it |
| Screen sharing shows black, no error | `XDG_CURRENT_DESKTOP` wasn't set before sway started, or your sway config is missing `include ~/.config/sway/config.d/*` |
| No audio at all | `firmware-sof-signed` missing |
| Brightness keys do nothing | Not in the `video` group, or you haven't logged out since being added |

Guide §11 is the full diagnostic command reference.

## Scope

These are my own machine's scripts, published because the reasoning may be
useful. They're not a distribution or an installer framework: they make
specific choices for a specific laptop, and they'll happily install a desktop
you didn't want if you run them without reading them first. Read
`--dry-run` output. Fork rather than file feature requests.
