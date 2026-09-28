# Noxia

Pinterest PFP router + Discord poster.

Downloads images from Pinterest boards and drops them into local `pfps/<channel>/`
folders, where each folder name **is** a Discord channel name. A running bot watches
`pfps/` and auto-sends each batch to its matching channel, then moves the files to
`pfps/<channel>/sent/`.

```
Pinterest board  →  pin  →  pfps/thighs/  →  bot watcher  →  #thighs
```

## Layout

| Path | What it is |
| --- | --- |
| `pinterest_tui.py` | The Textual TUI dashboard (jobs, queue, favorites, channels, dedup, log) |
| `bin/pin` | Downloads a board and routes images into existing channel folders |
| `bin/pfp-channels` | Channel manifest CRUD — `list` / `add` / `remove` / `rename` / `match` |
| `bin/pfp-add` | Adds images manually into a channel folder (dedup + collision-safe) |
| `bin/pfp-dedup` | Hash-DB maintenance — `stats` / `scan` / `rebuild` / `cleanup` |
| `favorites.json` | Saved TUI job presets (board URLs) |
| `.pinterest_tui_config.json` | Last-used TUI form defaults |

## Runtime paths (outside this repo)

These scripts read from your home directory:

| Path | Purpose |
| --- | --- |
| `~/pfp-bot/pfps/` | `WATCH_FOLDER` — the routed image folders |
| `~/pfp-bot/pfps/.channels.json` | Channel manifest: `id`, `name`, `folder` per channel |
| `~/pfp-bot/image-hashes.txt` | Dedup DB — `md5hash|filename|date` |
| `~/pfp-bot/.env` | Bot token + `RANDOM_PFP_CHANNELS` (never committed) |
| `~/pfp-bot/index.js` | The Discord bot that watches `pfps/` and sends |
| `~/noxia/download-history.json` | Recent download runs shown in the TUI |
| `~/.cache/pin-download/` | Staging area, flattened then cleared each run |
| `~/.local/bin/` | Where the scripts are installed (symlink or copy from `bin/`) |

## How routing works

`pin` never creates folders. It only drops into **existing** top-level
`pfps/<channel>/` folders, matching a downloaded image's board/section name:

1. **Explicit aliases** — the `ALIASES` dict in `bin/pin` is checked first. Keys are
   matched as a substring of the lowercased board name, so `"ghetto"` routes
   `ghetto pfps` → `darkskin-female` even though the names share nothing.
2. **Fuzzy leading-token match** — strips a showcase prefix (`@ % = ! ~ # & + * . -`),
   lowercases, collapses punctuation, then checks whether the folder's tokens are a
   leading sequence of the board name's tokens. Token-prefix tolerant, so
   `thigh`↔`thighs` and `pfp`↔`pfps` both match.
3. **No match** → the image lands in the `pfps/` root, where the bot posts a
   dropdown asking where to send it.

The bot's sender (`flushFolderBatch` → `resolveChannelByNameOrId` in `index.js`)
resolves by **Discord channel name or ID** — it does not read `.channels.json`. So a
folder must be named exactly after a real Discord channel to auto-send.

### Adding a channel

```sh
pfp-channels add <channel-id> <name>     # creates pfps/<name>/, registers manifest, adds to /random
pfp-channels rename <id-or-folder> <new-name>
pfp-channels list
```

`rename` renames the Discord channel (best-effort; needs the bot token and
`MANAGE_CHANNELS`), the local folder, the manifest, and any `pin` alias **targets**
that pointed at the old folder. It keeps `sent/` intact. If the Discord API call
fails it warns and still completes the local rename.

If a board's display name doesn't naturally match its channel name, add an alias in
`bin/pin`:

```python
ALIASES = {
    "ghetto": "darkskin-female",
    "female": "white-female-pfp",
}
```

## Usage

```sh
pin <user>                                  # whole account
pin <user>/<board>                          # one board
pin <user>/<board>/<section>                # one section

pfp-add <image>... --channel <name|id>      # manual add
pfp-dedup stats | scan | rebuild | cleanup [--apply]
```

`pfp-dedup cleanup` is a dry run unless you pass `--apply`.

## TUI

```sh
python -m venv venv && venv/bin/pip install textual
venv/bin/python pinterest_tui.py
```

Panels: bot control, new job, favorites (download or remove), identity rotation,
add channel, rename channel, stats, dedup, last downloads, live log, queue.

## Install

```sh
cp bin/* ~/.local/bin/
chmod +x ~/.local/bin/{pin,pfp-channels,pfp-add,pfp-dedup}
```

## Notes

- Requires `python3`, `md5sum`, `curl`, and `~/pinterest-downloader/pinterest-downloader.py`
  on `$PATH` for `pin` to work.
- Nothing here creates Discord channels automatically — create them in Discord
  first, then add the matching folder.
- Bot tokens live in `~/pfp-bot/.env` and are never committed.
