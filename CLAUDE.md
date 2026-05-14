# Project Notes

## Constraints
- **No hand-rolled JSON** — never manually construct or parse JSON with string concatenation, manual escaping, or homebrew parsers. Always use a proper JSON library or API (e.g. CEF's `CefParseJSON`/`CefWriteJSON`, or a vendored library if CEF isn't available in that context).
- **No artificial heartbeats/polling** - event-driven architecture only. Never use timeouts as a workaround for proper event integration. No arbitrary timeout-based bailouts in shutdown paths either — fix the root cause instead.
- **No texture stretching during resize** - CEF content must always render at 1:1 pixel mapping. Never scale/stretch textures to fill the viewport. Gaps from stale texture sizes are acceptable; stretching is not.

## Build / Run
Use `just` — recipes are OS-gated via `[macos]`/`[linux]`/`[windows]` attributes, so the same command works everywhere:
```
just deps         # one-time: submodules, CEF download, macOS brew packages
just build        # configure (if needed) + build
just test         # ctest
just run          # run with debug logging → logs to build/run.log
just clean        # remove build/ and dist/ (keeps CEF SDK)
just update-deps  # update all vendored/fetched deps (CEF, doctest, quill)
just dmg          # [macos] build app bundle + distributable DMG
just appimage     # [linux] build AppImage via podman/docker
just flatpak      # [linux] build Flatpak bundle
```
Platform-specific entry points live in `dev/linux/`, `dev/macos/`, `dev/windows/`, imported by the top-level justfile.

## Architecture
- **CEF** (Chromium Embedded Framework) — hosts jellyfin-web as an embedded browser; handles JS-to-C++ IPC for player control commands and renders the UI as an overlay texture above the video layer. Multi-process: browser process (main app, owns CefBrowser), renderer process (V8/Blink), GPU process. IPC via `CefProcessMessage`.
- **mpv** (fork in `third_party/mpv`) — video playback; the desktop client injects native shims to override browser media playback
- Wayland subsurface for video layer

### Threading Model
Six threads with distinct responsibilities — do not cross data between them without atomics or the event queue:

| Thread | Role |
|--------|------|
| **mpv digest thread** | `mpv_wait_event` → normalize → fan-out to `EventQueue` |
| **CEF consumer thread** | drains `EventQueue` → `execJs` / resize |
| **CEF render thread** | `multi_threaded_message_loop` (autonomous) |
| **Input thread** | Wayland dispatch → CEF key/mouse → mpv async commands |
| **mpv VO thread** | configure/close callbacks → surface ops |
| **Main thread** | startup → `waitForClose` → cleanup |

## Source Layout

```
src/
  main.cpp                 # app entry point; threading model in file header comment
  common.h                 # global singletons: g_mpv, g_platform, g_media_type,
                           #   g_playback_state, g_shutting_down, g_display_hz
  event_queue.h            # lock-free SPSC ring buffer EventQueue<T,N>
  cef/
    cef_app.cpp/h          # CEF process bootstrap (subprocess dispatch, multi-process)
    cef_client.cpp/h       # CefLayer — generic CEF client; app logic injected via callbacks
    resource_handler.cpp/h # custom resource loading
  mpv/
    handle.h               # MpvHandle — typed C++ wrapper around mpv_handle
    event.cpp/h            # MpvEvent struct, MpvObserveId enum, mpv::* atomic accessors
    options.h              # mpv option name constants
    capabilities.cpp/h     # codec/device capability enumeration (libavcodec)
  browser/
    web_browser.cpp/h      # main jellyfin-web browser
    overlay_browser.cpp/h  # OSD overlay for player controls
    about_browser.cpp/h    # about/splash screen
  web/
    native-shim.js         # base IPC bridge — injects jmpNative into the renderer
    mpv-player-base.js     # player abstraction layer
    mpv-video-player.js    # video playback control
    mpv-audio-player.js    # audio playback control
    input-plugin.js        # keyboard / gamepad input
    client-settings.js     # settings UI integration
    context-menu.js        # right-click menu
    connectivityHelper.js  # server discovery
  player/
    media_session.cpp/h         # cross-platform media session interface
    media_session_thread.cpp/h  # async media session event loop
    mpris/                      # Linux MPRIS D-Bus backend
    macos/                      # macOS MediaPlayer framework backend
    windows/                    # Windows SMTC backend
  platform/
    platform.h             # abstract Platform interface
    wayland.cpp            # Linux Wayland backend (primary)
    x11.cpp                # Linux X11 fallback (optional, ENABLE_X11)
    macos.mm               # macOS Cocoa/Metal/IOSurface
    windows.cpp            # Windows DirectComposition/D3D11
  jellyfin/
    api.cpp/h              # server URL normalization, validation
    device_profile.cpp/h   # codec/media device profile (libavcodec-sourced)
  input/
    dispatch.cpp/h         # input event routing
    hotkeys.cpp/h          # media key / hotkey mapping
    input_wayland.cpp/h    # Wayland input backend
    input_macos.mm/h       # macOS input backend
    input_windows.cpp/h    # Windows input backend
  paths/
    paths.cpp/h            # path abstraction (config, cache dirs)
    linux.cpp / macos.cpp / windows.cpp

tests/                     # doctest unit tests
third_party/
  mpv/                     # git submodule — custom fork; built with meson
  cef/                     # CEF binary distribution — downloaded, not in git
  quill/                   # header-only logging library (vendored)
  doctest/                 # header-only test framework (vendored)
cmake/
  FindCEF.cmake            # CEF package detection
  embed_resources.cmake    # embeds JS shims into C++ headers at build time
  GenerateVersion.cmake    # version from VERSION file + git hash
dev/tools/
  download_cef.py          # fetches CEF binary for current platform
  update_deps.py           # updates CEF / doctest / quill; pass --check to verify only
```

## IPC / Message Passing

- **JS → C++**: the `native-shim.js` injects a `jmpNative` object into the renderer; calls arrive in the browser process as `CefProcessMessage` and are dispatched through the `CefLayer::setMessageHandler` callback.
- **C++ → JS**: `CefFrame::ExecuteJavaScript` called from the CEF consumer thread.
- **Browser ↔ renderer process**: `CefProcessMessage` via `SendProcessMessage(PID_RENDERER / PID_BROWSER)`.
- **`CefLayer`** (`src/cef/cef_client.h`) is policy-free — it handles rendering, lifecycle, context menus, and keyboard events. All app logic is injected at construction time via:
  - `setMessageHandler` — IPC from renderer
  - `setCreatedCallback` / `setBeforeCloseCallback` — lifecycle
  - `setContextMenuBuilder` / `setContextMenuDispatcher` — context menu items

## Key API Patterns

### MpvHandle (`src/mpv/handle.h`)
Typed C++ wrapper around `mpv_handle*`. For properties that are already observed, **prefer the `mpv::*` atomic accessors** (`mpv::fullscreen()`, `mpv::osd_pw()`, `mpv::display_scale()`, etc.) — they read atomics seeded by the event handler and never go to the mpv API. Sync reads via `MpvHandle::GetProperty*` are only for init-time or one-shot values that mpv doesn't expose as observable properties.

### EventQueue<T, N> (`src/event_queue.h`)
Lock-free SPSC ring buffer (N must be power of 2, default 256).
- Producer: `try_push(item)` — non-blocking, auto-signals the wake event on success.
- Consumer: block on `wake().fd()` (Linux) / `wake_handle()` (Windows), drain with `try_pop`, then call `drain_wake()` before re-entering the wait.

### MpvObserveId (`src/mpv/event.h`)
Use these numeric IDs as `reply_userdata` when calling `mpv_observe_property`. `digest_property` switches on them instead of string-comparing property names, so new properties **must** get a new `MpvObserveId` constant.

### JS shims
The 8 `src/web/*.js` files are embedded as C++ headers at build time by `cmake/embed_resources.cmake`. After editing a JS shim, a plain `just build` re-embeds it — no manual step needed.

## mpv Integration
- **Never call sync mpv API (`mpv_get_property`, etc.) from event callbacks** - causes deadlock during video init. Use property observation or async variants instead.

## mpv Event Flow
mpv is the authoritative source of playback state. All state (position, speed, pause, seeking, etc.) flows from mpv property observations outward to the JS UI and OS media sessions. The JS UI and MPRIS/macOS media sessions are consumers — they never determine playback state, they only reflect what mpv reports. This means things like rate changes, seek completion, and position updates come from mpv, not from JS round-trips or manual bookkeeping.

## Testing
```
just test    # build (if needed) + ctest --output-on-failure
```
- Tests live in `tests/`; framework is **doctest** (vendored in `third_party/doctest/`).
- `BUILD_TESTING=ON` is set by the default `just build` on Linux.
- Current test: `tests/jellyfin_api_test.cpp` — covers `jellyfin_api::normalize_input`, `extract_base_url`, `is_valid_public_info`.

## Dependency Management
```
just update-deps           # update CEF, doctest, quill to latest
just update-deps --check   # verify versions without modifying files
```
- CEF version is pinned in the `CEF_VERSION` file; after editing it run `python3 dev/tools/download_cef.py`.
- mpv is a git submodule at `third_party/mpv`; update with `git submodule update --remote third_party/mpv`.

## Debugging
- For mpv (third_party/mpv), jellyfin-web, and CEF: investigate source code directly before suggesting debug logs that require manual user action
