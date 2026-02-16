# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Local Clipboard is a self-contained local network clipboard/chat app for sharing text and files between devices on the same network. It compiles to a single Go binary with all web assets embedded — no external dependencies at runtime.

## Commands

```bash
# Run locally (default port 8080)
make run

# Run on a custom port
make run PORT=3000

# Build release binaries for macOS (Intel/Silicon), Linux, Windows
make build

# Build with a specific version
make build VERSION=1.2.3

# Update Go dependencies
make update
```

Binaries are output to `./build/`. Releases are triggered by pushing a `v*` git tag.

There are no tests or linting configurations in this project.

## Architecture

The entire backend is in `main.go`. Web assets (`web/index.html`, `web/script.js`, `web/styles.css`) are served directly by the Go HTTP server — there is no build step for the frontend.

**Backend core components:**

- **Hub** — central WebSocket connection manager. Runs in its own goroutine with a `select` loop over `register`, `unregister`, and `broadcast` channels. Maintains a `clients` map of `*websocket.Conn → sender IP`.
- **FileStore** — in-memory file storage protected by `sync.RWMutex`. Files are stored as base64-encoded content and cleared on server restart.

**HTTP endpoints:** `/ws` (WebSocket), `POST /upload`, `GET /file/:id`, `/qr` (QR code PNG), `/api/version`, `POST /clear`, `POST /set-interval`, `POST /toggle-pause`.

**File sharing flow:** Client uploads file via `POST /upload` → backend stores in FileStore and returns a file ID → frontend sends a WebSocket message with the file ID (no content) → Hub broadcasts to all clients → each client fetches `/file/:id` to download.

**Auto-clear timer:** Hub owns a `ClearConfig` (interval, paused, nextClearTime) and communicates with three buffered channels (`clearNowCh`, `setIntervalCh`, `togglePauseCh`). A nil `<-chan time.Time` (the nil-channel trick) disables the timer without a separate flag. On any clear or config change, Hub broadcasts a `type:"clear"` or `type:"config"` WS message to all clients. New clients receive the current config immediately on connect.

**Frontend (`web/script.js`):** Manages the WebSocket connection with auto-reconnect (2s interval). Own messages are filtered client-side to prevent echo. The update checker fetches from the GitHub API on page load to detect new releases and is platform/arch-aware. Includes an auto-clear control bar with interval selector, pause/resume button, live countdown display, and manual clear button.

**Versioning:** The version string is injected at build time via `-ldflags "-X main.Version=$(VERSION)"` and exposed via `/api/version`.
