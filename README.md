# Sweep

One command that finds the junk piling up on an Arch Linux or Omarchy system, shows how
much space each kind takes, and deletes only what you tick. Nothing is deleted until you
confirm, and `sweep -n` never deletes anything.

The part general cleaners miss: **on Arch, the biggest junk is Arch-specific.** BleachBit
and friends clear browser caches and temp files, but they don't know that yay keeps every
AUR package it has ever built, that mise keeps every old version of your tools, or that the
pacman cache grows with every update. That is usually where the gigabytes are.

```
$ sweep -n
Found 2.9G you can free:

    1.6G  AUR build cache
          5 packages: old builds, downloads, -git sources (fetched again on the next update)
    1.1G  Old mise tool versions
          claude@2.1.278 claude@2.1.282 codex@0.155.1 codex@0.157.0 (no config uses them)
   74.7M  Developer download caches
          npm 74.7M (1 npx package in use, kept)
   21.7M  Crash dumps  (sudo)
          1 file for debugging crashes (systemd deletes them after 2 weeks anyway)
   14.4M  Pacman package cache  (sudo)
          2 package files of old versions or removed packages (keeps the newest 2 of each)
    3.8K  Old config backups
          2 files named *.bak.<timestamp>, left behind by 'omarchy refresh'

Tips:
  • Clean the pacman cache automatically every week: sudo systemctl enable --now paccache.timer
  • To hunt big files yourself: dua i ~
```

Run plain `sweep` and the same list becomes a checklist
(via [gum](https://github.com/charmbracelet/gum)) with the safe kinds already ticked.

## Install

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/skylightlim/sweep/main/sweep -o ~/.local/bin/sweep
chmod +x ~/.local/bin/sweep
```

`~/.local/bin` is already on your `PATH` on Omarchy. On plain Arch, add it, or put `sweep`
in a folder that is.

Sweep only needs bash and pacman. Each optional tool lets it clean one more kind of junk:
`pacman-contrib` (pacman cache), `yay` or `paru` (AUR cache), `mise` with `jq` (old tool
versions), and `gum` (the checklist; without it you get plain y/n prompts). On Omarchy,
`pacman-contrib`, `yay`, `jq` and `gum` come preinstalled.

### Update

Run the install lines again.

### Remove

```bash
rm ~/.local/bin/sweep
```

Sweep keeps no config or state, so that is all there is to it.

## Usage

```bash
sweep        # scan, tick what to clean, confirm, clean
sweep -n     # scan and report only
```

Run it as your normal user. Steps that touch system files are marked `(sudo)` and ask for
your password once.

## What it cleans

| Junk | Ticked by default | How |
|---|---|---|
| Pacman package cache | yes | `paccache`: keeps the newest 2 versions of each package, drops packages you removed |
| Orphaned packages | no | `pacman -Rns` on what `pacman -Qdtq` lists; pacman asks again before removing |
| System journal | yes | `journalctl --vacuum-size=200M`, listed only when the journal is bigger than that |
| Crash dumps | no | the `core.*` files in `/var/lib/systemd/coredump` |
| AUR build cache | yes | installed packages: everything git doesn't track (old builds, downloads, `-git` clones); removed packages: the whole folder |
| Old mise tool versions | yes | `mise prune --tools`: versions no config file uses |
| Developer download caches | yes | npm, npx, pip, uv, go, cargo, bun, yarn and pnpm caches |
| Stale app caches | yes | files in `~/.cache` not read or changed for 30 days |
| Trash | no | `gio trash --empty` |
| Old config backups | no | the `*.bak.<timestamp>` files `omarchy refresh` leaves in `~/.config` |

The one trade-off: `-git` AUR packages download their source again on their next update.

## Is it safe?

- **It won't `rm -rf` anything outside your home folder.** Folder deletions go through a
  guard that refuses such paths, and system files are only touched through pacman's and
  systemd's own tools.
- **It skips work in progress:** the AUR cache while yay, paru or makepkg is running,
  package steps while pacman holds its lock, and npx packages that a running program (an
  MCP server, say) is using.
- **It leaves git repos in `~/.cache` alone**, because deleting only some of their files
  would break them.
- **What you might want back starts unticked:** trash, orphaned packages, crash dumps and
  config backups.

## Settings

| Variable | Default | What it sets |
|---|---|---|
| `SWEEP_DAYS` | `30` | days a `~/.cache` file must be unused to count as stale |
| `SWEEP_KEEP` | `2` | versions of each package kept in the pacman cache |
| `SWEEP_JOURNAL` | `200M` | size the system journal is trimmed down to |

```bash
SWEEP_DAYS=14 sweep
```

## What it doesn't do

- **Clear whole browser caches.** The browser downloads them again within days, and pages
  load slower until it has. Sweep only removes cache files nobody has used for a month.
- **Delete history, cookies or passwords.** That is privacy cleaning; use your browser's
  "Clear browsing data" or BleachBit.
- **Clean temp files.** `/tmp` lives in RAM on Arch and is wiped at every reboot, and
  systemd already deletes old files in `/tmp` (after 10 days) and `/var/tmp` (after 30).
- **Shred files or wipe free space.** Neither works on SSDs or on Btrfs, which don't
  overwrite data in place. Full-disk encryption plus TRIM is what protects deleted data.
- **Prune Docker.** `docker system prune` also removes stopped containers you may still
  want, so sweep only points you to `docker system df` when Docker is running.

## License

[MIT](LICENSE)
