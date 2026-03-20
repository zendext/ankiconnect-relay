# anki-remote-api — v0 Design

## 1. Goals and scope

### Goals

Build a v0 remote card creation system for Anki that supports:

- Receiving structured flashcard data from external callers (Discord skill, CLI, etc.)
- Routing requests to an Anki Desktop runtime that lives inside Docker
- Writing cards via `bridge API -> AnkiConnect -> Anki Desktop`
- Reporting whether manual intervention is currently required

### In scope for v0

- Single-user validated first; multi-user by running multiple isolated stacks
- Non-root Anki Desktop container with in-container virtual desktop (VNC/noVNC)
- First bridge API for runtime probing and basic AnkiConnect passthrough
- Manual first-run GUI setup is acceptable in v0
- AnkiConnect installed manually inside the desktop runtime is acceptable in v0

### Out of scope for v0

- Full automation of every first-run GUI interaction
- Single-instance multi-tenancy
- Custom Anki addon development
- Complex template migration
- Production orchestration / scheduler / autoscaling

---

## 2. Architecture

```text
HTTP caller
    ↓
anki-remote-api bridge
    ↓
AnkiConnect (inside desktop container)
    ↓
Anki Desktop
```

The bridge is responsible for reporting runtime readiness and proxying safe AnkiConnect operations.
The desktop runtime is responsible for running the actual Anki GUI environment.

---

## 3. Runtime model

The validated runtime model is:

- Anki Desktop in Docker
- non-root user (`uid=1000`, `gid=1000`)
- TigerVNC + openbox + noVNC
- persistent bind mounts for:
  - `/anki-data`
  - `/home/anki/.local/share/AnkiProgramFiles`
  - `/home/anki/.cache/uv`

The container uses a single runtime model:

- if the launcher-installed runtime is missing, it starts launcher bootstrap
- if the installed runtime exists, it starts the real Anki binary
- the virtual desktop stays available even if Anki exits, so manual GUI recovery remains possible

This avoids splitting GUI lifecycle concerns into multiple externally visible container modes.

---

## 4. Bridge API (current v0 slice)

Currently implemented / being stabilized:

- `GET /health`
- `GET /status`
- `POST /anki/version`
- `POST /anki/deck-names`

### `/status` goals

The bridge should expose enough state for orchestration to tell whether the runtime is usable.

Current state signals:

- whether program files are installed
- whether AnkiConnect answers
- whether manual intervention is likely still required
- recent startup log tail

---

## 5. Near-term roadmap

1. Stabilize bridge runtime probing
2. Add auth / token gate to bridge API
3. Add generic AnkiConnect passthrough or selected safe actions
4. Add deck ensure helpers
5. Add note create / lookup / update / upsert logic
