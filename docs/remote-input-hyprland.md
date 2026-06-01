# Remote Input on Hyprland via KDE Connect — Architecture & Upstream Fixes

## Overview

KDE Connect's mousepad plugin allows a phone to act as a remote mouse/keyboard
for a desktop. On Wayland, this requires compositor-level input injection. The
standard approach is through `xdg-desktop-portal` — but the portal stack has a
chain of missing pieces across three separate repositories.

This document describes each layer of the problem and the changes needed at
each level to make phone-as-mouse work on Hyprland without patching KDE Connect.

---

## Architecture (Target End State)

```
Phone touch input
  → KDE Connect (stock, unpatched)
    → org.freedesktop.portal.RemoteDesktop.CreateSession  (frontend portal)
      → org.freedesktop.impl.portal.RemoteDesktop           (backend portal)
        → xdg-desktop-portal-hyprland
          → wlr-virtual-pointer-unstable-v1 / zwp_virtual_keyboard_v1
            → Hyprland compositor
              → Cursor moves, clicks register, keys type
```

### The Three Repositories

| Layer | Repository | PR/Status |
|-------|-----------|-----------|
| App | `kdeconnect-kde` (KDE) | MR #963: CLOSED |
| Backend portal | `xdg-desktop-portal-hyprland` (Hyprland) | PR #402: OPEN |
| Frontend portal | `xdg-desktop-portal` (flatpak) | PR #2011: OPEN |

---

## 1. `kdeconnect-kde` — The Application Layer

### What was proposed
A new `HyprlandRemoteInput` backend in the mousepad plugin that opens a direct
Wayland connection to Hyprland and uses `wlr-virtual-pointer` / `virtual-keyboard`
protocols to inject input, completely bypassing the portal stack.

### What the upstream said
> "We are already using a standard, desktop-agnostic solution for the input
> emulation [the RemoteDesktop portal]. I understand that Hyprland etc don't
> support it, but that's their deficiency, not something we should work around
> with non-standard protocols."

**Verdict:** Fixed upstream is the correct approach, not per-compositor hacks
in KDE Connect. The proper solution is to fix the portal stack so KDE Connect's
existing `WaylandRemoteInput` (portalbased) works on Hyprland out of the box.

### What was created (for local use)
A `plugins/mousepad/hyprlandremoteinput.cpp` backend is available at:
- Branch: `wlr-virtual-pointer-backend`
- Still works on this machine, protected from updates via `NoUpgrade`

---

## 2. `xdg-desktop-portal-hyprland` — The Backend Portal

### What is missing
The `org.freedesktop.impl.portal.RemoteDesktop` interface is not implemented.
The Hyprland backend only provides:
- `org.freedesktop.impl.portal.ScreenCast`   ✅
- `org.freedesktop.impl.portal.Screenshot`   ✅
- `org.freedesktop.impl.portal.GlobalShortcuts`  ✅
- `org.freedesktop.impl.portal.RemoteDesktop`    ❌  ← missing

### What was created
PR #402 adds a complete `CRemoteDesktopPortal` class with:

**D-Bus method handlers:**
- `CreateSession` / `SelectDevices` / `Start` — session lifecycle
- `ConnectToEIS` — libei/EIS file descriptor server (used by KDE Connect)
- `NotifyPointerMotion` / `NotifyPointerButton` / `NotifyPointerAxis` — D-Bus fallback
- `NotifyKeyboardKeycode` / `NotifyKeyboardKeysym` — keyboard input

**Input injection:**
- `wlr-virtual-pointer-unstable-v1` protocol bindings for mouse
- `virtual-keyboard-unstable-v1` protocol bindings for keyboard
- xkbcommon keysym → evdev keycode conversion (P2 fix)
- `sendFrame()` on EIS_EVENT_FRAME (scroll fix)

**Protocol files (MIT license):**
- `protocols/wlr-virtual-pointer-unstable-v1.xml`
- `protocols/virtual-keyboard-unstable-v1.xml`

**Build changes:**
- CMakeLists.txt: `protocolnew()` entries, libeis-1.0 dep, xkbcommon dep
- PortalManager: global bindings for pointer/keyboard managers, EIS fd polling
- `hyprland.portal`: interface declaration added

### Files changed
```
src/portals/RemoteDesktop.hpp       — Header (new)
src/portals/RemoteDesktop.cpp       — Implementation (new, 590+ lines)
src/core/PortalManager.hpp          — Protocol pointers, portal member
src/core/PortalManager.cpp          — Bindings, init, EIS poll handling
protocols/wlr-virtual-pointer-*.xml — Protocol (new)
protocols/virtual-keyboard-*.xml    — Protocol (new)
CMakeLists.txt                      — Build rules
hyprland.portal                     — Interface declaration
```

### PR review items addressed
| Item | Description | Commit |
|------|-------------|--------|
| P1   | EIS devices not advertised (`eis_device_add()` before `start_emulating()`) | `a747e1c` |
| P2   | Keysyms need keymap translation, not forwarded as keycodes | `a747e1c` |
| P3   | TOUCHSCREEN advertised but not implemented, removed | `ff2650f` |

### Remaining concern
Session handle must be serialized as D-Bus ObjectPath type (`o`), not string (`s`).
Fixed in commit `17d3e68` using `sdbus::ObjectPath{}` wrapper.

---

## 3. `xdg-desktop-portal` — The Frontend Portal

### What is missing
The main `xdg-desktop-portal` daemon creates a single D-Bus proxy to each
backend at startup with `G_DBUS_PROXY_FLAGS_NONE`. When the backend process
restarts (e.g., after crash, package update, or explicit restart), the frontend
holds a stale proxy connected to the dead backend process.

Async D-Bus calls (`g_dbus_proxy_call`) through a stale proxy **never complete**
— the callback never fires because the D-Bus connection's reply dispatcher
cannot match the reply (the backend is gone). The session is left in limbo:
created in `handle_create_session` but never finalized in `create_session_done`,
so it eventually times out and closes.

### The fix (PR #2011)
Monitor the `g-name-owner` property on the impl proxy. When the name owner
changes (the backend reconnects), recreate the proxy on the new backend:

```c
g_signal_connect(impl, "notify::g-name-owner",
                 G_CALLBACK(on_name_owner_changed), remote_desktop);
```

The handler `on_name_owner_changed`:
1. Gets the new name owner
2. Creates a fresh `XdpDbusImplRemoteDesktop` proxy via
   `xdp_dbus_impl_remote_desktop_proxy_new_sync()`
3. Sets default timeout to `G_MAXINT`
4. Replaces `remote_desktop->impl` with the new proxy
5. Logs the reconnection

The same pattern is applied to `screen-cast.c` for consistency.

### Files changed
```
src/remote-desktop.c   — +73 lines (name-owner handler, proxy rebuild)
src/screen-cast.c      — +121 lines (same pattern for ScreenCast)
```

### Why this matters for our stack
Without PR #2011, any restart of `xdg-desktop-portal-hyprland` (or any backend)
breaks the RemoteDesktop portal for all subsequent calls until `xdg-desktop-portal`
itself is restarted. This is critical for:
- Package updates (backend binary replaced)
- Crash recovery
- Development/iteration cycles
- Desktop session starts (race between frontend and backend)

## 4. `xdg-desktop-portal-hyprland` — Session Token Assertion Crash

### What was found
During testing, the main portal process (`xdg-desktop-portal`) crashed with:

```
xdg-desktop-portal:ERROR:../src/xdp-session.c:296:xdp_session_initable_init:
  assertion failed: (session->token != NULL)
Bail out! xdg-desktop-portal: ... ABORTING
```

This happens when a RemoteDesktop session is created without a valid
`session_handle_token` in the options dictionary, or when a stale client
connection triggers session cleanup without a proper token. The crash
terminates the main portal, which then needs to be restarted.

### Impact
After the crash:
1. The portal process dies (core dump / SIGABRT)
2. D-Bus activation restarts the main portal if configured
3. On restart, the main portal must reconnect to all backends
4. PR #2011's reconnect logic in `remote-desktop.c` handles this reconnection
5. The hyprland backend must also be running when the main portal restarts
   — if the systemd service is masked, auto-restart fails

### Fix
This is an assertion in `xdp_session_initable_init` that was triggered by
a session with NULL token. The callers of `xdp_session_create` must ensure
a token is always provided. This is already the case for normal KDE Connect
usage (KDE Connect provides `session_handle_token`), but edge cases like
rapid reconnections or gdbus test clients can trigger it.

A defensive fix would add a NULL check in `xdp_session_initable_init`:
```c
if (session->token == NULL)
    return g_error_new (..., "Session created without token");
```
instead of the assertion that crashes the process.

However, this is a separate issue from the RemoteDesktop implementation.
The crash only occurs during edge cases (test tools, rapid restarts). With
normal KDE Connect usage and stable portal uptime, the assertion is not hit.

---

## Current Status

| Component | Location | Status | Link |
|-----------|----------|--------|------|
| KDE Connect backend | `kdeconnect-kde` | CLOSED — upstream prefers portal path | [MR #963](https://invent.kde.org/network/kdeconnect-kde/-/merge_requests/963) |
| Portal backend | `xdg-desktop-portal-hyprland` | OPEN — all review items addressed | [PR #402](https://github.com/hyprwm/xdg-desktop-portal-hyprland/pull/402) |
| Session assert crash | `xdp-session.c:296` | assertion crash kills main portal on edge cases | Not yet filed |
| Frontend proxy fix | `xdg-desktop-portal` | OPEN — proxy reconnect for backends | [PR #2011](https://github.com/flatpak/xdg-desktop-portal/pull/2011) |

---

## Working Around the Incomplete Chain

On this machine, three system files are protected from `pacman` updates via
`NoUpgrade` in `/etc/pacman.conf`:

| File | Purpose |
|------|---------|
| `/usr/lib/xdg-desktop-portal-hyprland` | Portal backend with RemoteDesktop |
| `/usr/lib/qt6/plugins/kdeconnect/kdeconnect_mousepad.so` | Direct wlr-virtual-pointer backend |
| `/usr/share/xdg-desktop-portal/portals/hyprland.portal` | Interface declarations |
| `/usr/share/dbus-1/services/org.freedesktop.impl.portal.desktop.hyprland.service` | D-Bus activation |

### Two working paths coexist

**Path A — Direct Backend (always works):**
```
Phone → KDE Connect (patched mousepad plugin)
  → HyprlandRemoteInput::handlePacket()
    → wlr-virtual-pointer protocol
      → Hyprland
```

**Path B — Portal Backend (when startup timing permits):**
```
Phone → KDE Connect (stock mousepad plugin)
  → org.freedesktop.portal.RemoteDesktop
    → xdg-desktop-portal (frontend, with PR #2011)
      → xdg-desktop-portal-hyprland (backend, with PR #402)
        → wlr-virtual-pointer protocol
          → Hyprland
```

### What is required for upstream-only operation

1. **PR #402 merged into `hyprwm/xdg-desktop-portal-hyprland`**
   — Adds RemoteDesktop portal implementation to the Hyprland backend.
   All review suggestions (P1, P2, P3) are addressed in the current revision.

2. **PR #2011 merged into `flatpak/xdg-desktop-portal`**
   — Adds proxy reconnect handling so backend restarts don't break the
   portal connection. Without this, any restart of `xdg-desktop-portal-hyprland`
   (including at system startup) can leave the frontend with a stale proxy.

3. **No changes needed to `kdeconnect-kde` itself.**
   The stock `WaylandRemoteInput` uses `org.freedesktop.portal.RemoteDesktop`
   with `ConnectToEIS` (libei), which PR #402 fully implements.

### Remaining unknowns

- **The `exported=0` issue is a test-tool artifact.** The debug build confirmed
  `create_session_done` fires, `finish` returns `response=0`, and the only
  reason the session closed was `request->exported == 0`. With `gdbus call`,
  the tool exits immediately after printing the request path, which triggers
  `xdp_request_unexport()` before the backend's async callback arrives.

- **`exported=1` with KDE Connect is a logical inference, not verified.**
  KDE Connect's `WaylandRemoteInput` is a long-running daemon that connects
  to the portal and listens for the `Response` signal via D-Bus signal
  subscription. It *should* stay connected long enough for the async callback
  to fire. However, this was never empirically confirmed — every attempt to
  test the full chain with stock KDE Connect hit startup-timing issues where
  the portal backend and KDE Connect weren't both ready simultaneously.

- **Startup ordering is the real unsolved problem.** The three processes
  (frontend `xdg-desktop-portal`, backend `xdg-desktop-portal-hyprland`,
  and `kdeconnectd`) start independently via D-Bus activation. If the backend
  starts after the frontend has already cached its proxy, the frontend needs
  PR #2011's reconnect logic to discover it. Even then, KDE Connect must
  initialize its portal connection after both portals are ready.

- PR #2011 also fixes the same proxy-reconnect issue for `ScreenCast`,
  which is a general quality-of-life improvement for the portal infrastructure.

---

## Repositories

| Repo | Fork | Branch |
|------|------|--------|
| `KDE/kdeconnect-kde` | `SnotacusNexus/kdeconnect-kde` | `wlr-virtual-pointer-backend` |
| `hyprwm/xdg-desktop-portal-hyprland` | `SnotacusNexus/xdg-desktop-portal-hyprland` | `remotedesktop-portal` |
| `flatpak/xdg-desktop-portal` | `SnotacusNexus/xdg-desktop-portal` | `pr-2011` |
