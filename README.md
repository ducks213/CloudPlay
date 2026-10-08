# CloudPlay

A cloud gaming service for your own PC games: they run on a server, video streams to the browser, input streams back. There are no built-in games; the server's owner imports them.

```
 Browser ──HTTPS──▶ Control plane ◀──heartbeats── Stream host(s)
   │                accounts, game library,        run games on hidden screens,
   │                placement                      encode + stream video
   └────────────── WebSocket (signed ticket) ──────────▶┘
```

## Install it

On a Linux computer (Ubuntu, Linux Mint, Debian…), from this folder:

```bash
./install.sh        # asks for your password once, for the system packages
```

That computer becomes your CloudPlay server, and the first account you create on it is the owner. It installs the packages CloudPlay needs, copies CloudPlay to `~/.local/share/cloudplay/app` (accounts and pictures go in `~/.local/share/cloudplay/data`), sets up Wine for Windows games, runs the server in the background (a systemd user service that starts when you log in), and adds the CloudPlay app to your menu and desktop.

**The CloudPlay app** (`client/app.py`) is CloudPlay in a window of its own, without a browser: WebKitGTK shows the pages and decodes the H.264 video and Opus sound. It keeps you signed in, starts the server if it isn't running, and its Fullscreen button makes the window truly full screen (Esc always reaches the game). Browsers work too: open `http://localhost:8080`. `./uninstall.sh` removes it again (`--purge` also deletes accounts and pictures).

## Run it (for development)

Requires Python 3.11+, `websockets` (10.x+), Pillow, PyGObject with GStreamer, and the system packages listed under *Importing your own games*.

```bash
python3 run.py                  # open http://localhost:8080
python3 -m unittest -v          # 17 tests, including a full end-to-end session
```

By default only this computer can reach CloudPlay. To play from other devices on your home network:

```bash
python3 run.py --lan            # listens on the network and prints the address to open
```

## Security

- **This computer only** unless you start it with `--lan`. Over the network it's plain `http://`, so use `--lan` only on networks you trust.
- **Invite-only accounts:** the first account is the owner. Everyone after needs a one-time invite code (valid 7 days) that the owner creates with **Invite a player**.
- **Games are kept away from private data:** on hidden screens, games run in a bubblewrap sandbox that hides browser profiles, keyrings, SSH/GPG keys and CloudPlay's own folder (and its database). The rest of the home folder stays visible so saves and settings work. Steam games and `--on-screen` games aren't sandboxed.
- **Private database:** `cloudplay.db` is readable only by its owner. Passwords are PBKDF2 hashes and sign-in tokens are stored hashed; sign-in attempts are rate-limited.

### Running the parts separately (the production layout)

```bash
export CLOUDPLAY_SECRET=$(openssl rand -hex 32)       # shared by control plane and hosts
python3 -m control.server --port 8080 --db /var/lib/cloudplay/cloudplay.db
python3 -m host.host --id gpu-fra-1 --region eu-central --capacity 8 \
    --control https://play.example.com --public-url wss://gpu-fra-1.play.example.com
```

You can add hosts at any time; they register themselves through heartbeats.

## Importing your own games

The first account created on a server is its **owner**. The owner sees an **Import a game** card in the library:

1. Browse to the game's program on this computer: a Windows `.exe` (runs with Wine) or a Linux executable or launcher script.
2. Give it a name, and optionally launch options (e.g. `-windowed`). Press **Import**.
3. Anyone signed in can now press **Play**. The game starts on a hidden screen on this computer and only shows up in the player's browser. Nothing opens on your monitor, and your mouse and keyboard stay yours.

Notes:
- Each game gets its own hidden 1920×1080 screen, so several people can play PC games at once, up to the host's capacity.
- **Stream quality:** PC games stream as **1080p at 60 fps in H.264**, encoded on the GPU through VA-API (`gstreamer1.0-plugins-bad`) or on the CPU with x264 (`gstreamer1.0-plugins-ugly`), at about 12 Mbit/s for full HD. The browser decodes it with WebCodecs, which only works on `localhost` or over HTTPS. Players on other devices over plain `http://` get the older JPEG stream (720p, 30 fps) automatically. If the connection falls more than half a second behind, the backlog is dropped and the stream continues from a fresh keyframe.
- **PlayStation:** in **Settings → Console**, the owner can add a PS4/PS5 that's linked in [chiaki-ng](https://github.com/streetpea/chiaki-ng) (PlayStation Remote Play; install it with `flatpak install flathub io.github.streetpea.Chiaki4deck` and link the console with the PIN from its Remote Play settings). It appears in the library as **PlayStation**: CloudPlay runs chiaki-ng on a hidden screen, connected to the console, and streams it with sound and the player's controller. One person at a time, as with Remote Play itself.
- **Steam games:** the import dialog lists the games installed in Steam on this computer. CloudPlay runs the Steam client itself on the game's hidden screen (`steam -silent -applaunch <appid>`), so Steam handles Proton, launch options, updates and the license as usual. The session ends when Steam reports the game closed (`RunningAppID` in `~/.steam/registry.vdf`); then Steam is shut down and anything left on that screen is ended. Steam runs once per account, so it can't be open on the desktop at the same time, and one Steam game runs at a time.
- **Windows games** run in CloudPlay's own Wine prefix (`~/.local/share/cloudplay/wine`), separate from `~/.wine` and anything you run on the desktop, with the translation layers Proton uses: **DXVK** 2.7.1 (Direct3D 8–11) and **VKD3D-Proton** 3.0.1 (Direct3D 12), both drawing through Vulkan. Set it up with `python3 -m host.wine` (downloads the pinned releases and checks their SHA-256). DXVK stays on 2.7 because 3.x needs a Vulkan extension that Wine 9.0 doesn't pass through.
- **GPU:** with `weston` installed (`sudo apt install weston`), each hidden screen is a rootful Xwayland inside a headless Weston that renders on the GPU. **OpenGL and Vulkan** games (so DirectX 9–12 under Wine too) draw with the graphics card directly. The screen refreshes at 60 Hz, so games are capped at 60 fps, matching the stream. Without Weston it falls back to Xvfb, where only OpenGL reaches the GPU, through [VirtualGL](https://github.com/VirtualGL/virtualgl/releases) (`vglrun -d egl`) if installed, and everything else is drawn by the CPU.
- **Sound:** each game gets its own virtual PulseAudio/PipeWire device (named through `PULSE_SINK`, `PIPEWIRE_PROPS` and `PIPEWIRE_NODE`, so PulseAudio, native PipeWire and ALSA games all use it), so its sound never plays on the host's speakers or reaches another player. It's captured and sent over the same WebSocket as **Opus** (48 kHz stereo, 128 kbit/s, decoded with WebCodecs) or as raw 16-bit PCM (about 1.5 Mbit/s) to browsers without WebCodecs. The browser schedules it about 60 ms ahead and starts over if it falls behind.
- **Controllers:** each game gets its own virtual **Xbox 360 controller** (uinput, same name and IDs as the kernel's xpad driver), so Linux games and Windows games (XInput/DirectInput through Wine) see a real pad with analog sticks and triggers. The browser sends the Gamepad API's standard layout whenever it changes. Needs write access to `/dev/uinput` (Steam's udev rules grant it) and `python3-evdev`; without them gamepads fall back to keys.
- **Sandbox:** games on hidden screens run under bubblewrap with their own `/dev/input` (only their own controller), their own Wine server (so Wine's controller handling isn't shared between players) and their own process list (everything they start ends with them).
- **Real fps:** the HUD shows the game's real frame rate (counted on the server from the X Damage events of its window) next to the rate the browser actually draws.
- **On your own screen instead:** start with `--on-screen` to open games on this computer's desktop. Then it's one PC game at a time, and keys and clicks are only sent while the game's own window is active (players can't type into your other apps). If the game is minimised, the stream pauses and input is dropped until you restore it.
- The Super/Windows key is never forwarded.
- **Gamepad** in PC games: D-pad/stick → arrow keys, A/B/X/Y → Z/X/C/V, Start → Enter, Back → Escape.
- Needs `weston` + `Xwayland` (or `Xvfb` as a fallback), GStreamer with `ximagesrc`, and `wine` for Windows games. `--on-screen` needs an X11 desktop session (not Wayland).
- Importing lets the owner run any program on the machine from the website, so use a strong password for the owner account if the site is reachable from other computers.

## How it works

**Control plane** (`control/server.py`)
- Accounts: PBKDF2-SHA256 passwords, opaque bearer tokens (stored hashed), and rate-limited login.
- Placement: new sessions go to the least-loaded host that sent a heartbeat in the last 20 s. Each account gets one stream; starting a new game ends the old one.
- Session lifecycle: `pending → active → (ending) → ended`. Heartbeats report started and ended sessions and the seconds played. The reply tells hosts which sessions to terminate. Sessions on hosts that stop heartbeating are closed as `host_lost`. Tickets nobody redeems expire after 60 s.

**Stream host** (`host/host.py`)
- The host verifies the HMAC-signed ticket (right host, not expired, not already used) and then launches the game.
- Stopping the host (Ctrl+C or SIGTERM) closes every running game and its hidden screen.
- There's no play time limit. Sessions end on quit, disconnect (closing the tab), 15 minutes without any input, or a server request.

**Web client** (`web/`)
- Sign up / sign in and the game library.
- Player: H.264 decoded with WebCodecs (JPEG as the fallback) onto a canvas. Input comes from keyboard, mouse, Gamepad API and an on-screen touch pad. The HUD shows the game's real fps, the fps actually drawn, round-trip time and bitrate.

**PC games** (`host/desktop.py`): launched in their own process group on a private screen (Weston + Xwayland on the GPU, or Xvfb), or with `--on-screen` found on the desktop through the window manager's client list. They're captured with GStreamer (`ximagesrc → videoconvertscale → vah264lpenc → h264parse → appsink` for 1080p60 H.264, or `jpegenc` at 720p30 as the fallback) and followed as they move, resize or go fullscreen. Colour conversion runs on the CPU because Intel's VA-API converter (`vapostproc`) shears the picture at some widths. Input is replayed through XTest. Held keys are released and the whole process group is terminated when the session ends.

Measured on an Intel Alder Lake laptop: 1080p at 60 fps and about 12 Mbit/s while a 3D program renders on the same GPU, with the server using about 60% of one CPU core.

## Path to production

This codebase has the right service boundaries. The following still has to change before real users and commercial games:

| Area | Now | Production |
|---|---|---|
| Video | H.264 over WebSocket, 1080p60 | H.264/AV1 over **WebRTC** (GStreamer `webrtcbin` or Pion), TURN servers |
| Games | Imported games on hidden screens on one machine | Games in GPU VMs/containers (Linux + Proton or Windows) with injected virtual gamepads |
| Hosts | Any machine | GPU instances (e.g. AWS g5/g6, GCP L4) in several regions; autoscaling on free capacity; a latency probe to pick the closest region |
| Data | SQLite | Postgres; Redis for host presence |
| Accounts | Username + password (no email or other personal data) | Password reset, OAuth |
| Ops | `run.py` | TLS (`wss://`), containers, metrics (fps, RTT, drops per session), central logs, abuse limits |
| Content | Your own games | Publisher licensing deals, or bring-your-own-library (Steam) integration |
Also if you are on Linux it won't be able to play games with EAC(easy anti-cheat) because I couldn't find a way to make it be like other cloud gaming services, so if you find a way to make it run these game let me know.
