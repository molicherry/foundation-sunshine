# Plan4: per-session window-follow capture (local control API)

This branch adds an **opt-in, per-session capture override** on top of `AlkaidLab/foundation-sunshine`:

- Normally Sunshine captures a whole display, so windowed games launched by a launcher (Playnite/Steam) only fill a small area of the stream.
- A trusted local client (for example a Playnite extension) can now ask **one streaming session** to follow a single window instead of the display.
- Default behaviour, `sunshine.conf`, `apps.json` and other sessions are untouched. The feature is off unless explicitly enabled.

## What changed

| File | Purpose |
|---|---|
| `src/streaming/capture_target.h` | per-session window-follow state (mode/title/pid/generation/observed) |
| `src/confighttp/capture_control.{h,cpp}` | `GET/POST /api/capture/window-follow` (localhost-only, authenticated, multi-session safe) |
| `src/confighttp/routes.cpp` | route registration |
| `src/stream.h` / `src/stream.cpp` | list active sessions together with their shared capture target |
| `src/video.cpp` | capture thread reinitializes when the override generation changes |
| `src/platform/windows/display.h` | WGC backend: override state + polling |
| `src/platform/windows/display_wgc.cpp` | foreground/title/pid window resolution, eligibility filter, fallback + runtime switch |
| `src/platform/windows/display_vram.cpp` | VRAM capture path polls the override too |
| `src/platform/windows/display_base.cpp` | force WGC for override sessions only (DDX fallback), global config untouched |
| `cmake/sources/host.cmake` | build the new source file |
| `docs/zh-CN/capture-window-follow.md` | API/error/limitations/rollback reference |

## Behaviour

- `capture_target == "window"`-like capture, but **runtime and per session** instead of a global config.
- Modes: `foreground` (follow current foreground window), `title` (substring match), `pid` (window of a given process).
- Window eligibility: visible, non-minimized, titled, non-tool, non-cloaked; shell surfaces and Sunshine's own windows are excluded.
- No eligible window -> falls back to display capture and retries; the stream never breaks.
- Disable / session end -> back to normal display capture.
- Exactly one active video session may be auto-selected; multiple sessions require an explicit `session_id` (HTTP 409 otherwise).

## API

```http
GET  /api/capture/window-follow
POST /api/capture/window-follow
```

POST body:

```json
{"action":"enable","mode":"foreground","window_title":"","process_id":0,"session_id":0}
```

`action`: `enable` | `disable` | `status`. Responses are JSON; errors: 400 (bad request), 401 (auth), 403 (non-local), 404 (no active session), 409 (multiple sessions / WGC unavailable in service mode), 500.

Localhost-only: remote callers are rejected even with valid credentials. WGC requires user-mode Sunshine (not the SYSTEM service).

## Build

```bash
git submodule update --init --recursive
cmake -B build -G Ninja -S .
ninja -C build
```

Requires MSYS2 UCRT64 (Windows) with the packages listed in `docs/building.md`.

## Verification

- Compiles with GCC 16 / CMake 4.4 / Ninja on Windows (all modified translation units, no warnings introduced).
- Runtime API checks against a running `sunshine.exe`: `GET` 200, no-session 404, bad action/JSON 400, non-local 401/403, concurrent requests stable.
- Full end-to-end window switching requires a real Moonlight session on a Windows host with a user-mode Sunshine instance; that path is not covered by the automated checks.

## Rollback

```bash
git revert <this commit>
```
