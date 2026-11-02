# couchlaunch

A small, opinionated command-line launcher that makes **Windows games run on a Linux
desktop** (Proton via [umu-launcher](https://github.com/Open-Wine-Components/umu-launcher))
and makes them **launchable from Moonlight clients** through a
[Sunshine](https://github.com/LizardByte/Sunshine) game-streaming host.

One command registers a game; the tool then keeps the game registry, the Proton
prefix, the resolution hand-off and the Sunshine app list in sync — so the game shows
up next to "Desktop" in the Moonlight app grid, with a cover image and the correct
resolution for whatever screen is asking for it.

```
couchlaunch add "Game Name" ~/Downloads/game-dir     # register (auto-detects the exe)
couchlaunch list                                     # slug, name, type, state, target
couchlaunch run game-name                            # what Sunshine calls
couchlaunch stop game-name                           # stop it again, restore resolution
```

---

## Why this exists

Two machines, one GPU:

```
┌──────────────────────────────┐        Sunshine (host)        ┌────────────────────────────┐
│  GPU desktop: RTX-class card │  ──── HEVC/H.264 stream ────▶ │  thin laptop: integrated   │
│  4K display, games installed │      1080p @ 60 fps           │  GPU, 1080p panel, no     │
│  Proton, prefixes, saves     │  ◀──── keyboard / gamepad ──── │  chance to run AAA games  │
└──────────────────────────────┘                               └────────────────────────────┘
```

The desktop is where the games live. The laptop is where you actually sit. Streaming
solves the hardware problem, but it creates a pile of small, boring problems that this
tool exists to remove:

| Problem | What `couchlaunch` does |
|---|---|
| Every Windows game needs its own Wine prefix, its own Proton invocation and its own launch flags. | `add` records the exe, the prefix and the arguments once; `run` replays them. |
| Sunshine needs an entry in `apps.json` per game, with the right `detached` command and an `undo` action. | `sync` regenerates those entries from the registry and restarts Sunshine safely. |
| The client may be a 1080p laptop while the host has a 4K panel: rendering 4K and downscaling looks worse *and* costs GPU time. | On a Sunshine launch the host monitor is switched to the **client's** resolution and restored afterwards (including when the client disconnects abruptly). |
| Installers put a 100 GB game inside the hidden Wine prefix, where nobody finds it again. | `install` + automatic `relocate` move the game into a normal directory and leave only Wine state in the prefix. |
| Moonlight tiles are faceless grey boxes without artwork. | `icon` fetches cover art (Steam, or SteamGridDB for portrait 2:3 art) and wires it up as an absolute `image-path`. |
| "Is something running? How do I kill it from the couch?" | `running` and `stop` (window close → SIGTERM → SIGKILL, then restore the display mode). |

The result: sit down at the laptop, open Moonlight, click the game's cover, play. Sit
down at the desktop, click the same entry in the application menu, play natively in 4K.

---

## What it supports

| Source | How it is handled |
|---|---|
| **Steam** (installed via Steam) | `couchlaunch add "Name" --steam-appid <ID>` — Sunshine launches `steam://rungameid/<ID>`. |
| **GOG offline installers** (Inno Setup `.exe` + `.bin`) | extracted with `innoextract` (no Wine needed), then registered. |
| **DRM-free / standalone games**, already unpacked | point `add` at the directory or the exe. |
| **Archives** (`.7z`, `.zip`, `.rar`, `.tar.*`) | extracted to the games directory, then registered. |
| **Installer-based releases** (ISO images, FitGirl/DODI-style repacks) | mounted/extracted, installer run under Proton, game files relocated, crack files copied **if present**. |

### Legal notice

This project is a **launcher**. It contains no game content, no DRM circumvention code
and no keys. Officially supported sources are the ones you can legally own a licence
for: **Steam libraries, GOG/offline installers and other DRM-free builds**.

The installer-ISO and repack paths exist because plenty of people legitimately own
older or offline copies distributed as disk images, and because the same code path is
what runs any Windows installer. If you use those paths with material you did not
legally obtain, that is **your** responsibility, not the project's. The authors do not
condone or support piracy.

---

## Requirements

- Linux with a Wayland compositor that speaks the `wlr-screencopy` protocol
  (developed against **Hyprland**; `hyprctl` is used for output switching).
- [umu-launcher](https://github.com/Open-Wine-Components/umu-launcher) (`umu-run`) and
  a Proton build (UMU-Proton or GE-Proton).
- [Sunshine](https://github.com/LizardByte/Sunshine) as a **user** systemd service.
- Optional: `innoextract` (GOG/Inno installers), `7z`/`bsdtar` (archives),
  ImageMagick (cover art), `curl` (cover download), `rsync` (relocation).

```
# Arch / CachyOS
sudo pacman -S umu-launcher sunshine innoextract p7zip imagemagick rsync curl
```

## Install

```bash
git clone https://github.com/<you>/couchlaunch.git ~/code/couchlaunch
ln -sf ~/code/couchlaunch/couchlaunch ~/.local/bin/couchlaunch
```

Optional: the resolution watchdog (restores the desktop mode if a stream dies
without a clean shutdown):

```bash
ln -sf ~/code/couchlaunch/bin/couchlaunch-display-guard ~/.local/bin/couchlaunch-display-guard
cp ~/code/couchlaunch/systemd/couchlaunch-display-guard.{service,timer} ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now couchlaunch-display-guard.timer
```

## Configuration

`~/.config/game-launcher/config.json` (all keys optional):

```json
{
  "games_root":  "~/Games",
  "prefix_root": "~/.local/share/game-prefixes",
  "covers_root": "~/.local/share/couchlaunch/covers"
}
```

Environment overrides: `GAMECLI_GAMES_ROOT`, `GAMECLI_PREFIX_ROOT`.
Per-game settings live in `~/.config/game-launcher/games/<slug>.json`.

| Key | Meaning |
|---|---|
| `path` | absolute path of the exe to run |
| `prefix` | Proton/Wine prefix directory |
| `args` | extra launch arguments |
| `display` | force this mode on **local** launches, e.g. `"1920x1080@60"` (default: leave the desktop alone) |
| `display_stream` | force this mode when **streamed** (default: follow the client) |
| `image` | absolute path to the Moonlight cover PNG |
| `sunshine` | `false` hides the entry from Moonlight |

Logs: `~/.cache/game-launcher/<slug>.log`.

---

## Commands

```
add          register a game (dir, exe, archive, ISO, installer folder, or --steam-appid)
install      run an installer (ISO / repack) under Proton, copy crack files, register
relocate     move an in-prefix installed game into the games directory
list         registered games (with running state)
running      what is running right now (pid + uptime)
run          launch a game (Sunshine calls this)
stop         stop a running game, restore the display mode
icon         set the Moonlight cover (Steam search, SteamGridDB, --file, --url)
desktop      create a .desktop entry so the game appears in the app menu
sync         regenerate Sunshine apps.json (--restart to apply immediately)
setres       switch the monitor to the client resolution  (Sunshine prep-cmd)
restoreres   restore the previous display mode            (Sunshine undo)
remove       drop a game from the registry (--purge also deletes the prefix)
```

### Typical flows

```bash
# a game already installed by Steam
couchlaunch add "Some Game" --steam-appid 123456
couchlaunch icon some-game            # cover art for the Moonlight tile

# a GOG offline installer
couchlaunch add "Some Game" ~/Downloads/setup_some_game_1.2_(12345).exe

# an installer ISO (mount -> install under Proton -> relocate)
couchlaunch add "Some Game" ~/Downloads/some_game.iso
couchlaunch install some-game
sudo umount /mnt/scene/some-game

# a FitGirl/DODI-style repack folder (setup.exe + fg-*.bin)
couchlaunch add "Some Game" "~/Downloads/Some Game [Repack]"
couchlaunch install some-game          # long; runs the GUI installer under Proton
```

After every `add`/`install`/`icon` the Sunshine app list is regenerated; Sunshine is
restarted only when no stream is active (`sync --restart` forces it).

## Streaming notes

- Game entries are launched with `detached`, so closing the stream does **not** kill the
  game. Use `couchlaunch stop <slug>` (works over SSH) to end it.
- On a Sunshine launch the host monitor follows `SUNSHINE_CLIENT_WIDTH/HEIGHT/FPS`;
  the previous mode is stored and restored on exit, on client disconnect, and — as a
  safety net — by the watchdog timer.
- Cover art must be a PNG referenced by an **absolute** path: Sunshine resolves
  relative `image-path` names inside its own assets directory, not next to `apps.json`.
  Tiles are 2:3, so `couchlaunch icon` normalises everything to 600×900 (crop for portrait
  sources, blurred backdrop for landscape ones).
- On the client, a Super/Win key bound to a desktop shortcut (e.g. GNOME's
  `overlay-key`) never reaches the host. Add non-conflicting duplicate bindings on the
  host (the author uses `ALT+Enter` for a terminal and `ALT+1..0` for workspaces).

## Troubleshooting

| Symptom | Fix |
|---|---|
| Moonlight tile has no image | `couchlaunch icon <slug>` then check that `apps.json` holds an absolute path; the Sunshine log prints `Couldn't find app image at path [...]` when it cannot resolve one. |
| Wrong exe launched | edit `path` in `~/.config/game-launcher/games/<slug>.json`, then `couchlaunch sync`. |
| Game will not start | read `~/.cache/game-launcher/<slug>.log` (Proton output). |
| Resolution stays at the stream size | `couchlaunch restoreres`, and enable the watchdog timer. |
| Sunshine stuck in `failed` after repeated restarts | `systemctl --user reset-failed app-dev.lizardbyte.app.Sunshine.service` (the tool already does this before restarting). |
| Two virtual gamepads / game ignores the controller | connect and start the controller *before* launching, and keep a single Moonlight instance. |

## Layout

```
couchlaunch                          the tool (single-file Python 3, stdlib only)
bin/couchlaunch-display-guard        resolution watchdog (systemd timer)
systemd/*.service,*.timer        watchdog units
```

Registry/config stays in `~/.config/game-launcher/` for backwards compatibility with
earlier private versions of this tool.

## License

MIT — see [LICENSE](LICENSE). The tool itself ships no game content and no DRM
circumvention code; see the legal notice above.
