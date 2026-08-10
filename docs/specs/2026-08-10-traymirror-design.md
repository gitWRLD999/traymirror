# traymirror design

**Date:** 2026-08-10
**Status:** draft, pending approval
**Author:** Lattice Labs

## Problem

Windows 11 can show the taskbar on every display, but the secondary taskbars it draws are not full
copies. They carry Start, task buttons, the clock and the notification bell, and they omit the
notification area entirely. If you run a tray-resident app, the only place its icon exists is the
primary monitor, so you look away from whatever you are working on to check it or click it.

traymirror fills exactly that gap and nothing else. It does not replace the secondary taskbar, it
does not reimplement the tray, and it leaves everything Windows already gets right alone.

### Measured baseline

All of the following was measured on the development machine on 2026-08-10, not assumed.

- Windows 11 Pro, build 26100 (24H2)
- Two monitors. `\\.\DISPLAY2` is primary at 3440x1440 with origin (0,0). `\\.\DISPLAY1` is
  secondary at 2560x1440 with origin (3440,0). Both are top aligned.
- Primary taskbar `Shell_TrayWnd` occupies (0,1392) to (3440,1440), so it is 48px tall.
- Secondary taskbar `Shell_SecondaryTrayWnd` occupies (3440,1392) to (6000,1440) and is present and
  visible, confirming "show taskbar on all displays" is enabled.

Walking the UI Automation tree of both taskbars gives the precise delta:

| Tray item | Primary x | Secondary | Kind |
|---|---|---|---|
| Show Hidden Icons | 3156 | absent | overflow host |
| BlueBubbles | 3188 | absent | app icon |
| Private Internet Access | 3220 | absent | app icon |
| Privacy (microphone in use) | 3252 | absent | system indicator |
| Network | 3284 | absent | system indicator |
| Volume | 3312 | absent | system indicator |
| Clock | 3340 | present at 5900 | system |
| Notifications | 3400 | present at 5960 | system |
| Show Desktop | 3428 | see note | system |

The six absent items form one contiguous run from x=3156 to x=3340, which turns out to matter a
great deal for the chosen approach.

On Show Desktop: the secondary taskbar's last UIA element ends at x=5988 and the monitor's right
edge is 6000, leaving a 12px sliver exactly matching the width of the primary's Show Desktop button.
The peek target appears to be functional there but simply not published as a UIA element, so it is
treated as already working and is out of scope.

## Approach

### What was rejected, and why

**Injecting a hook DLL into explorer.exe.** This is the highest-fidelity way to observe tray icon
registrations, because it can intercept the `Shell_NotifyIcon` protocol directly. It is rejected on
purpose and permanently. The intended audience includes people on managed corporate machines where
EDR and application control will either block the injection or flag the process, and a utility that
trips security tooling is not a utility anyone can use at work. This is a standing design
constraint, not a phase 1 shortcut.

**Reading explorer.exe memory for the tray icon table.** The classic technique reads `TBBUTTON`
entries out of a `ToolbarWindow32` inside `TrayNotifyWnd` via `ReadProcessMemory`. This was probed
directly and **it no longer exists on Windows 11 24H2**: `TrayNotifyWnd` (hwnd 65780) has zero child
windows, and an exhaustive walk of every window in the session found exactly one
`ToolbarWindow32`, belonging to Tailscale's own UI rather than the shell. The Windows 11 tray is
XAML, and its icon list lives in C++ and XAML objects rather than a flat array. Reading those is
possible but would mean re-deriving object layouts on essentially every cumulative update, which is
a maintenance burden with no upside given the alternative below.

**Enumerate icons and re-render them ourselves.** Workable, and it was the original plan: use UIA to
enumerate, `PrintWindow` to capture each icon's pixels, cache the bitmaps and invalidate on change.
It carries a whole capture subsystem, a cache, invalidation logic, DPI handling and theme handling,
and its fidelity is only ever approximate.

### What was chosen

`DwmRegisterThumbnail` and `DwmUpdateThumbnailProperties`, the documented DWM APIs behind taskbar
hover previews and Alt-Tab. They take a source window and a destination window plus an optional
source crop rectangle, and the compositor then renders a live copy on the GPU continuously.

This was validated end to end before the design was written. A thumbnail of `Shell_TrayWnd` cropped
to window-relative x 3156 to 3340 was rendered into a 184x48 borderless window at (5716,1392) on
DISPLAY1, right aligned to that monitor's clock at 5900. Both DWM calls returned `S_OK`. The result
is pixel identical to the real tray and sits flush against the native secondary clock with no
visible seam. Successive captures show state changing live (the volume icon changed to muted and the
privacy indicator changed between probes), confirming it is a compositor feed rather than a static
image.

The consequences are large. There is no capture loop, no bitmap cache, no invalidation logic, no
per-icon rendering, no theme handling and no icon DPI code, because DWM does all of it. Fidelity is
perfect by construction because the pixels *are* the real taskbar's pixels. Cost at rest is
effectively zero. It also means the Volume, Network and Privacy indicators are mirrored for free,
which removes most of what "full parity" was originally going to cost.

The one thing it does not give us is interaction. DWM thumbnails are visual only and do not hit
test, so input has to be handled separately.

### Why not mirror the whole taskbar

Cloning the entire primary taskbar is the obvious simplification and it does not work. The primary
is 3440 wide and the secondary is 2560, and with `TaskbarAl=1` the Start button and task buttons are
centre aligned, so a straight copy lands Start in the wrong position. It would also duplicate task
buttons that the native secondary taskbar already renders correctly, and renders for the correct
monitor. Mirroring only the missing strip keeps every working thing native.

## Architecture

Three projects. `TrayMirror.Core` holds interop and logic and must never reference WPF, so it stays
unit testable without a UI thread. `TrayMirror` is the WPF application shell. `TrayMirror.Core.Tests`
covers the logic.

### Components

**`TrayGeometry`** resolves the crop. It queries UIA for two boundaries on the primary taskbar: the
left edge of the first notification-area element, and the left edge of the clock. The difference is
the strip. It converts screen coordinates to source-window-relative coordinates by subtracting the
source window's rect origin, which is the single easiest mistake to make with `rcSource`.

**`MonitorTargets`** decides where each mirror goes. For each target monitor it finds that monitor's
`Shell_SecondaryTrayWnd`, locates its clock, and right aligns a strip-width window so it ends where
the clock begins. If a monitor has no secondary taskbar, it falls back to a floating bar in that
monitor's bottom right corner.

**`ThumbnailHost`** owns the DWM handles. One registration per mirror window, updated when the crop
changes, and unregistered on monitor removal, taskbar recreation and shutdown. Every `dwmapi` call
returns an `HRESULT` and every one is checked.

**`InputRouter`** translates and synthesises. A click at mirror-local offset `dx` maps to primary
screen x of `stripLeft + dx`, where the click is synthesised. Left click and right click follow the
same translation.

**`MenuRelocator`** moves the resulting menu. Two out-of-process `SetWinEventHook` registrations
catch the menu as it is created, and `SetWindowPos` moves it beside the mirrored icon on the target
monitor. Both hooks are needed: classic Win32 tray menus are `#32768` popups and raise
`EVENT_SYSTEM_MENUPOPUPSTART`, while XAML island flyouts such as the quick-settings panels never
raise that event and are only visible through the broader `EVENT_OBJECT_SHOW`. Neither hook requires
injection, because both are registered out of process.

**`ShellWatcher`** rebuilds everything on the events that invalidate it: the registered
`TaskbarCreated` message when Explorer restarts, display changes, and DPI changes.

**`Settings`** persists which monitors are enabled, as JSON under `%APPDATA%`.

### Data flow

Boundary probe via UIA produces a crop rectangle. The crop plus a target monitor produces a mirror
window with a registered thumbnail. DWM then renders continuously with no further involvement from
us. Separately, a click on a mirror window produces a translated screen coordinate, a synthesised
click on the primary, and a relocated menu.

## Scope

**Phase 1, the mirror core.** Boundary detection, mirror windows on selected monitors, live DWM
thumbnails, monitor and DPI and Explorer-restart resilience, settings, autostart. At the end of
phase 1 every missing icon including Volume, Network and Privacy is visible on secondary monitors,
but not yet clickable.

**Phase 2, input routing.** Coordinate translation, click synthesis, and menu relocation for
ordinary Win32 tray menus.

**Phase 3, the overflow flyout.** The hidden-icons chevron opens
`TopLevelWindowForOverflowXamlIsland`, which was observed in the probe. Either mirror it the same
way or relocate it.

**Phase 4, flyout parity.** Volume and Network quick-settings flyouts. See the open risk below.

## Open risk

Phase 4 is not yet costed, because one question is unanswered: whether XAML island flyouts can be
relocated the way ordinary `#32768` menus can. If they can, phase 4 is nearly free and amounts to
reusing `MenuRelocator`. If they cannot, phase 4 means writing our own volume panel against Core
Audio (`IMMDeviceEnumerator`, `IAudioEndpointVolume`, which is stable and documented) and our own
network panel against the WLAN API (documented for wifi, but the real panel also fronts VPN,
ethernet, airplane mode and hotspot).

This should be settled by a spike before phase 4 is planned, and it does not block phases 1 to 3.

## Non-goals

- Injecting into or modifying Explorer in any way
- Reading undocumented Explorer memory structures
- Replacing or reimplementing the secondary taskbar
- Reimplementing the notification area as a host that apps register with directly
- Supporting Windows 10

## Testing

`TrayGeometry` and `MonitorTargets` are pure functions over rectangles and are unit tested directly
across DPI scales, taskbar edges and monitor arrangements, including the arrangement measured above.

Model diffing is unit tested: icons appearing and disappearing must produce the correct crop width
change and the correct mirror repositioning.

Integration testing uses a harness application that registers synthetic `Shell_NotifyIcon` icons, so
tests assert against a known tray rather than whatever happens to be running. The harness asserts
that a registered icon widens the strip, that the mirror repositions, and that a synthesised click
reaches the owning window.

DWM behaviour itself is not unit testable and is covered by manual verification against a checklist:
Explorer restart, monitor hotplug, resolution change, DPI change, light and dark theme, taskbar
auto-hide, and a full-screen exclusive application on the primary.

## Decisions taken

| Decision | Choice | Why |
|---|---|---|
| Rendering | DWM thumbnail | Validated working, perfect fidelity, near zero cost, deletes an entire subsystem |
| Enumeration | UI Automation | Documented and stable, and only two boundary queries are needed |
| Explorer injection | Never | Must stay safe on managed corporate machines |
| Memory reading | Never | The legacy table is gone on 24H2 and the replacement is not stable |
| Runtime | .NET 10 | .NET 8 LTS ends November 2026, .NET 10 runs to November 2028 |
| UI framework | WPF | Non-activating topmost overlays with exact placement are well trodden here |
| Minimum OS | Windows 11 21H2, build 22000 | The XAML tray shape this depends on begins with Windows 11 |
