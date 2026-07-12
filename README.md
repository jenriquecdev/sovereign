# ┌─────────────────────────────────────────┐
# │  Sovereign :: personal music server  │
# └─────────────────────────────────────────┘

> Self-hosted audio streaming over Cloudflare Tunnels · FastAPI · PWA · MediaSession API

---

## Introduction

`Sovereign` is a self-hosted personal music streaming service designed to run on a low-resource home server and serve audio files to any browser or installed PWA instance over a secure Cloudflare Tunnel. The server is implemented in Python using FastAPI and handles HTTP Range Requests (RFC 7233) to deliver partial audio content, enabling seek operations and efficient network use without loading entire files into memory. The frontend is built with Vanilla JavaScript and the HTML5 `<audio>` element, structured as a Progressive Web App with MediaSession API integration for OS-level playback controls. Cloudflare Tunnels provide externally routable HTTPS access via an outbound-only daemon, removing the requirement for port forwarding, a static public IP, or a reverse proxy configuration on the home network.

---

## Architecture

```
  ┌───────────────────────────────────┐
  │  Client (Browser / PWA)           │
  │  HTML5 <audio> · MediaSession API │
  └────────────────┬──────────────────┘
                   │ HTTPS (Range requests)
                   ▼
         Cloudflare Tunnel
         (outbound-only daemon)
                   │
                   ▼
  ┌───────────────────────────────────┐
  │  FastAPI Streaming Server         │
  │  uvicorn · HTTP 206 handler       │
  │  /stream/{id} · /tracks           │
  └────────────────┬──────────────────┘
                   │ file I/O
                   ▼
  ┌───────────────────────────────────┐
  │  Local Storage                    │
  │  ~/music  (mp3 / mp4)             │
  └───────────────────────────────────┘
```

**Data flow:** `Client → Cloudflare Tunnel → FastAPI Streaming Server → Local Storage`

---

## Core Features

- **Background playback via MediaSession API** — registers `navigator.mediaSession` metadata and action handlers (`play`, `pause`, `previoustrack`, `nexttrack`), exposing controls on OS lock screens and notification trays; audio playback continues when the browser tab is backgrounded or the screen is locked on mobile.
- **Chunked audio streaming via HTTP Range Requests** — the `/stream/{id}` endpoint parses the `Range` request header, computes byte offsets, and returns an HTTP 206 Partial Content response with the requested file slice via FastAPI `StreamingResponse`; HTTP 200 is returned for full-file requests and HTTP 416 for out-of-bounds ranges.
- **PWA installation support** — a linked `manifest.json` declares `name`, `short_name`, `start_url`, `display: standalone`, `theme_color`, and icon assets (192 × 192 and 512 × 512), satisfying browser installability criteria and enabling the application to run in a standalone window without browser chrome.


---

## Prerequisites

Before installing, ensure the following are present on the host system:

- **Python ≥ 3.9** — the streaming server requires f-string support and current `asyncio` semantics.
- **Node.js** (any LTS release) — required only for the Tailwind CSS build step that generates the static stylesheet; not needed at runtime.
- **Cloudflare account** — a free account is sufficient; used to provision a persistent tunnel domain.
- **`cloudflared`** — Cloudflare's tunnel daemon; [installation instructions](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/) vary by OS. Verify with `cloudflared --version`.

---

## Installation

### 1. Clone the repository

```sh
git clone https://github.com/<jenriquecdev>/stream.local.git
cd stream.local
```

### 2. Create and activate a Python virtual environment

```sh
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

### 3. Install Python dependencies

```sh
pip install -r requirements.txt
```

### 4. Build the Tailwind CSS output

```sh
npx tailwindcss -i frontend/styles/input.css -o static/output.css --minify
```

This step requires Node.js and produces `static/output.css`, which is loaded by the frontend at runtime. Run it again whenever the Tailwind configuration or HTML templates change.

---

## Configuration

Copy `.env.example` to `.env` at the repository root and set each variable:

```dotenv
# Absolute or relative path to the directory containing audio files (MP3 / MP4).
# Required — the server refuses to start if this variable is unset or the path does not exist.
MUSIC_DIR=/home/user/music

# Network interface the uvicorn process binds to.
# Default: 0.0.0.0 (all interfaces). Set to 127.0.0.1 to restrict to localhost only.
HOST=0.0.0.0

# TCP port uvicorn listens on.
# Default: 8000. Change if the port is already in use on the host.
PORT=8000

# Name of the Cloudflare Tunnel created with `cloudflared tunnel create <name>`.
# Required for the tunnel to route traffic to the correct local service.
# Omitting this variable will cause `cloudflared tunnel run` to fail.
TUNNEL_NAME=music-tunnel

# Comma-separated list of allowed CORS origins.
# Must include the public tunnel domain so browser clients can reach the API.
# Default: none — omitting this variable blocks all cross-origin requests.
CORS_ORIGINS=https://<tunnel-domain>.trycloudflare.com
```

### Running the server

```sh
uvicorn backend.main:app --host "${HOST:-0.0.0.0}" --port "${PORT:-8000}"
```

Pass `--reload` during development to enable automatic reloading on file changes. Remove it in production.

### Cloudflare Tunnel setup

Authenticate `cloudflared` once (opens a browser window):

```sh
cloudflared tunnel login
```

Create the tunnel (only required on first run):

```sh
cloudflared tunnel create music-tunnel
```

Start the tunnel, routing traffic to the local server:

```sh
cloudflared tunnel run --url http://"${HOST:-0.0.0.0}":"${PORT:-8000}" "${TUNNEL_NAME}"
```

After the tunnel starts, Cloudflare assigns a public HTTPS URL. Add that URL to `CORS_ORIGINS` in `.env` and restart the server.

---

## Quickstart

This section assumes the server is running and the Cloudflare Tunnel is active. If not, complete the [Installation](#installation) and [Configuration](#configuration) steps first.

### 1. Access the web interface

Open a browser and navigate to the public HTTPS URL assigned by Cloudflare — for example:

```
https://<tunnel-domain>.trycloudflare.com
```

The URL is printed to stdout when `cloudflared tunnel run` starts. The interface loads the track list automatically by fetching `GET /tracks` from the same origin.

### 2. Install as a PWA

The application meets browser installability criteria via its `manifest.json`. Installation steps vary by platform:

- **Chrome / Edge (desktop)** — an install icon appears in the address bar on the right side. Click it and select **Install** in the confirmation dialog. The app opens in a standalone window without browser chrome.
- **Chrome (Android)** — tap the browser menu (⋮) and select **Add to Home screen**. The app icon is added to the launcher and opens in standalone mode.
- **Safari (iOS)** — tap the Share button (⬆) and select **Add to Home Screen**. The app is pinned to the home screen and launches without the Safari navigation bar.

Once installed, the PWA launches directly from the home screen or app launcher and retains its connection to the tunnel URL set at install time.

### 3. Background playback and MediaSession controls

Start playback by selecting a track from the library list. Audio continues uninterrupted when the browser tab is backgrounded or the device screen is locked — the HTML5 `<audio>` element holds the audio context across tab visibility changes.

The application registers `navigator.mediaSession` metadata (title, artist, album, artwork) and the following action handlers:

| Handler | Action |
|---|---|
| `play` | Resume playback |
| `pause` | Pause playback |
| `previoustrack` | Jump to the previous track |
| `nexttrack` | Jump to the next track |

These handlers surface on:

- **Mobile (iOS / Android)** — the lock screen media widget and notification tray control strip.
- **Desktop (macOS / Windows / Linux)** — OS media key bindings (⏮ ⏯ ⏭) and any media control panel that integrates with the browser (e.g., macOS Control Center, Windows system tray).

No additional configuration is required. The MediaSession state updates automatically on each track change.
