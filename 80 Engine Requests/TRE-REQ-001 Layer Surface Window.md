---
title: TRE-REQ-001 Layer Surface Window
aliases: [TRE-REQ-001, LayerWindow Request, Layer Surface Request]
type: engine-request
status: draft
owner: phil
created: 2026-10-18
updated: 2026-10-18
tags: [engine-request, spec/surface, integration/wayland, needs-verification]
related: ["[[ADR-0001 Shell Surface Strategy]]", "[[Prior Art - Linux Shell Surfaces]]", "[[80 Engine Requests]]"]
depends_on: []
---

# TRE-REQ-001: Layer surface window

> [!summary]
> Add a `LayerWindow` type to tre. It is a `Window` whose Wayland role is `zwlr_layer_shell_v1` instead of xdg-toplevel, so it can anchor to screen edges, reserve space, sit on a stacking layer, take a chosen keyboard mode and bind to one output. A companion `PopupWindow` type opens menus and flyouts parented to a layer surface. A small output API lets the shell place one surface per monitor.

> [!danger] Blocks the shell
> No bar, panel, dock, OSD or wallpaper can ship without this. Every note under `40 Specifications/Surfaces/` carries `> [!warning] Blocked by [[TRE-REQ-001 Layer Surface Window]]` until tre ships it. Decision record: [[ADR-0001 Shell Surface Strategy]] (Option A, `proposed`).

This note is for the maintainer to carry to the tre project ([[AGENTS#6. Engine requests (tre / Tesserae)]]). Nothing here is filed upstream.

## Shell need

| Need | Mechanism ([protocol](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)) |
|---|---|
| Stacking layer | `set_layer`: background 0, bottom 1, top 2, overlay 3 |
| Edge placement | `set_anchor` bitfield: top 1, bottom 2, left 4, right 8 |
| Stretch along an edge | `set_size` with 0 on an axis, valid only when anchored to both opposite edges on that axis |
| Reserved space | `set_exclusive_zone`: positive pixels, 0, or -1. A positive value counts only for one edge, or one edge plus both perpendicular edges. |
| Corner exclusive zone | `set_exclusive_edge` (since v5) |
| Offsets | `set_margin` (top, right, bottom, left) |
| Keyboard focus | `set_keyboard_interactivity`: `none`, `exclusive`, `on_demand` (since v4) |
| Per-output placement | `get_layer_surface` takes a `wl_output`, or NULL to let the compositor choose |
| Popups | `get_popup` parents an `xdg_popup` to the layer surface |
| Compositor dismissal | `closed` event, for example when the output is removed |

## Why this belongs in the engine

Surface roles are windowing mechanism: they decide how the compositor places and stacks a window, not what the window contains. [[AGENTS#3. Layering principle]] puts mechanism in tre. If the shell created the surface (ADR-0001 Option B), it would own configure, scale, output hot-plug and input routing, which tre already owns for ordinary windows. Python sets properties and never touches a frame; `LayerWindow` keeps that split.

## What tre has today

Source: [Python API](https://mindderivative.github.io/tre/api/python/), [Custom windowing](https://mindderivative.github.io/tre/design/custom-windowing/), [Architecture](https://mindderivative.github.io/tre/architecture/), read 2026-10-18.

| Area | Present | Absent from the docs |
|---|---|---|
| Window | `Window(width, height, title, decorations)`; several windows via `App.add_window` then `App.run()` | Role selection, parent or transient windows |
| Platform | `window.get("platform")` returns `"wayland"`, `"x11"`, `"windows"` or `"macos"`; `scale_factor`; `center()` | Output enumeration, targeting an output |
| Overlays | `show_layer` / `hide_layer` draw nodes inside the window's own content | A separate OS surface for popups |
| Transparency and input | `transparent`, `blur_behind`, `click_through` | Input regions per area |
| Undocumented keys | `window.set(...)` accepts `x`, `y`, `always_on_top`, `skip_taskbar` but the property tables do not describe them | Whether they work on Wayland |
| Backend | `engine-platform` owns the `winit` loop, input translation and the AccessKit adapter | Surface creation details; `wgpu` and raw-window-handle use |

#needs-verification: confirm the absences in the tre source and `ARCHITECTURE.md`. The Window reference and Layers guide were not read in full.

> [!warning] Name collision
> tre already uses "layer" for in-window overlays (`show_layer`). The new type uses the protocol's term for a different thing. Keep `layer=` on `LayerWindow` to match the protocol and Fabric, and define both senses in the Glossary. See [Open items](#open-items).

## Proposed API

Illustrative only. None of this exists in tre.

```python
bar = LayerWindow(
    namespace="tesserae-shell.bar",   # protocol namespace, identifies the surface's purpose
    layer="top",                      # "background" | "bottom" | "top" | "overlay"
    anchor=("top", "left", "right"),
    size=(0, 32),                     # 0 stretches between opposite anchors
    exclusive_zone=32,                # int; 0; -1
    exclusive_edge=None,              # edge name; needed for corner anchors (compositor protocol v5)
    margin=(0, 0, 0, 0),              # top, right, bottom, left
    keyboard="none",                  # "none" | "on_demand" | "exclusive"
    output=None,                      # Output | None (compositor picks)
)
app.add_window(bar)
```

### Type and inheritance

`LayerWindow` subclasses `Window`. `root`, `create`, `show_layer`, `after`, `every`, `on`, `simulate`, `advance` and `snapshot` behave as they do today, so the existing headless testing story applies unchanged.

Constructor-only: `namespace`, `output`. Live through `set()` (one atomic commit, like today's `set()` semantics): `layer`, `anchor`, `size`, `exclusive_zone`, `exclusive_edge`, `margin`, `keyboard`.

Window keys with no meaning for a layer surface (`decorations`, `maximized`, `minimized`, `fullscreen`, `resize_border`, `system_menu`, `title`-as-title-bar) raise on `set()` rather than being ignored.

### Validation

The constructor and `set()` reject the protocol errors before any request goes to the compositor, so the shell sees a Python error with the offending key:

- `invalid_size`: a 0 dimension without anchors on both opposite edges.
- `invalid_anchor`: a bitfield outside the four edges.
- `invalid_exclusive_edge`: the edge is not among the anchors.
- `invalid_keyboard_interactivity`, `invalid_layer`: unknown name.
- A positive `exclusive_zone` with an anchor set the protocol treats as 0 produces a warning event, not an error, because the protocol accepts it.

### Lifecycle and events

| Event | Payload | Fires when |
|---|---|---|
| `configure` | `width`, `height` | The compositor assigns a size. tre acks and resizes the render target. Existing `resize` also fires. |
| `closed` | `reason`: `"compositor"` \| `"output_removed"` \| `"app"` | The compositor sends `closed`, or the app closes the window. |
| `output_changed` | `output` | The surface enters a different output or its scale changes. |

After `closed` with reason `"compositor"` or `"output_removed"` the window is dead (protocol: further changes are ignored). The shell creates a new `LayerWindow` for a new output.

A window must not attach a buffer before the first configure is acked ([protocol](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)). tre owns that sequencing; `App.run()` opens a `LayerWindow` without any shell involvement.

### Outputs

```python
for out in app.outputs():          # list[Output]
    ...
app.on("output_added", fn)         # fn(output)
app.on("output_removed", fn)       # fn(output)
```

`Output` is a read-only value with: `name` (stable connector name such as `DP-1`), `description`, `x`, `y`, `width`, `height` in logical pixels, `scale_factor`, `transform`. The shell keys per-output surfaces on `name`.

#needs-verification: `wl_output` alone does not provide a stable name before version 4. Check whether `xdg-output-unstable-v1` or `wl_output` v4 is required, and what KWin 6.7 advertises.

### Capability query

```python
tre.supports("layer_shell")        # False on GNOME/Mutter, X11, Windows, macOS
```

`LayerWindow(...)` raises `UnsupportedPlatform` when the query is false. The request recommends raising, not silently degrading to a plain window as Quickshell's `PanelWindow` does on X11: a bar that quietly lacks an exclusive zone is a worse failure than a clear error. The shell runtime can build its own degraded fallback from `Window` (see [If declined or infeasible](#if-declined-or-infeasible)). The platform-neutral wrapper question from ADR-0001 is therefore settled here as a recommendation, for the maintainer to confirm.

### Popups

`show_layer` cannot serve tray menus or flyouts on a 32 px bar, because it draws inside the window's own bounds. Popups need their own surface.

```python
menu = PopupWindow(
    parent=bar,
    anchor_rect=(x, y, w, h),         # in the parent's coordinates
    anchor="bottom_left",             # point on the rect
    gravity="bottom_right",           # direction the popup extends
    constraint_adjustment=("flip_y", "slide_x"),
    grab=True,                        # dismiss on outside click / Escape
)
```

- Role: `xdg_popup`, parented through `zwlr_layer_surface_v1.get_popup`. The popup is created with a NULL xdg parent and attached before its initial commit (protocol requirement).
- Keyboard mode is inherited from the layer surface by the protocol. A bar with `keyboard="none"` cannot give a popup keyboard focus. The shell sets the bar to `on_demand` while a menu is open, or the request adds a popup-level override. #needs-verification: whether `xdg_popup.grab` gives keyboard focus independently of the parent's interactivity on KWin.
- `closed` fires with reason `"dismissed"` on outside click, Escape or parent destruction.
- Positioning uses `xdg_positioner`, so the compositor flips and slides the popup to stay on screen. The shell does not compute screen edges.
- Nested popups (submenus) take another `PopupWindow` as `parent`.

This part is separable. If the layer surface lands first, ship popups as a second milestone; the bar and OSD do not need them.

## Required behaviour (non-API)

| Area | Requirement |
|---|---|
| Render | Same pipeline as `Window`: damage tracking, persistent target, no per-frame Python. Idle surfaces draw no frames. |
| Scale | Fractional scale and mixed-DPI outputs. Surface scale follows the output it is on. #needs-verification: `wp_fractional_scale_v1` and `wp_viewporter` on KWin. |
| Input | Pointer, touch, keyboard with key repeat and XKB layout, text input, scroll axes, cursor shape. Input matches `Window` as seen by `engine-core`. |
| Input region | An empty or partial input region so a transparent OSD does not capture clicks. Extends the existing `click_through` key. |
| Clipboard | `read_clipboard` / `write_clipboard` work from a layer surface that has keyboard focus. |
| Accessibility | The AccessKit tree is exposed for layer surfaces. #needs-verification: whether the current AccessKit adapter requires a `winit` window. |
| Transparency | `transparent` works so rounded or floating bars render without a background rectangle. |
| Threading | `LayerWindow` methods follow the existing rule: cross threads only through `LoopHandle.call_soon`. |
| Headless | `simulate`, `advance` and `snapshot` work without a compositor, using a mock backend for CI. |

## Implementation candidates

tre's backend is `winit` in `engine-platform`. `winit` has no layer-shell role. The design request leaves the choice to the maintainer; these are the candidates, none verified.

| | Approach | For | Against |
|---|---|---|---|
| I1 | Separate Wayland backend in `engine-platform` for `LayerWindow`, built on `smithay-client-toolkit` and driven from the same loop beside `winit`. The `wgpu` surface comes from the raw Wayland handles. | Contained. Ordinary windows are untouched. | Two input and event sources in one loop. Input, text input, clipboard and AccessKit are reimplemented for this path. |
| I2 | Same as I1, but share one `wl_display` connection with `winit`. | One seat, one clipboard, one focus owner. | #needs-verification: `winit` may not expose its connection or event queue in a form that allows this. |
| I3 | Replace `winit` with a native Wayland backend on Linux. | One path, no duplication. | Large. Regresses X11 unless `winit` stays for it. |

Recommendation: investigate I2 first; if the connection cannot be shared, take I1. Treat I3 as out of scope for this request.

> [!danger] Two input sources
> I1 puts seat, focus and clipboard handling in two places. A shell where bar clicks work but the clipboard or focus behaves differently from a tre dialog is a defect users will see. Record in [[Risk Register]] when that note exists.

## Alternatives

| Alternative | Where it lives | Verdict |
|---|---|---|
| External surface: the shell creates the layer surface, tre renders into a handle | ADR-0001 Option B, `TRE-REQ-002` (not written) | Smaller engine change, but the shell inherits configure, scale, hot-plug and input routing. Rejected unless this request proves infeasible. |
| Plain `Window` plus `always_on_top`, `skip_taskbar`, `x`, `y` and KWin rules | ADR-0001 Option C | No exclusive zone, so maximised windows cover the bar. Prototype only. |
| `org_kde_plasma_shell` | ADR-0001 Option D | KDE-only, thinly documented, no prior art. #needs-verification |

## If declined or infeasible

1. Write `TRE-REQ-002` for an external-surface hand-off (ADR-0001 Option B) and re-score the ADR.
2. Until then, the shell runtime builds a throwaway prototype from `Window` with `decorations=False`, `always_on_top` and `skip_taskbar`, kept in `90 Research/` and labelled as not shippable. This lets layout and design-system work proceed.
3. A degraded `Window`-based mode for compositors without layer-shell is a separate product decision and stays out of v1 unless an ADR adds it.

## Acceptance criteria

- [ ] On KWin 6.7 Wayland, `LayerWindow(layer="top", anchor=("top","left","right"), size=(0,32), exclusive_zone=32)` creates a bar; a maximised ordinary window does not overlap it.
- [ ] Changing `exclusive_zone` or `anchor` through `set()` reaches the compositor in one commit with no intermediate configure at a wrong size.
- [ ] A `(0, 32)` size without both left and right anchors raises a Python error naming `size` before any protocol request is sent.
- [ ] A surface bound to output `DP-1` stays on `DP-1`. Unplugging it fires `closed` with reason `"output_removed"`, and the app continues without crashing.
- [ ] `app.outputs()` lists every connected output with a stable `name`; hot-plug fires `output_added` and `output_removed`.
- [ ] A transparent `LayerWindow` with an empty input region passes pointer clicks through to the window below.
- [ ] A `keyboard="on_demand"` surface receives key events when focused; a `"none"` surface never does.
- [ ] A `PopupWindow` opens next to its `anchor_rect`, flips at a screen edge, and fires `closed` with reason `"dismissed"` on outside click and Escape.
- [ ] A bar on a 1.0 scale output and a bar on a 1.5 scale output both render crisp text.
- [ ] `tre.supports("layer_shell")` is `False` on a compositor without the protocol, and `LayerWindow(...)` raises `UnsupportedPlatform` there.
- [ ] `simulate`, `advance` and `snapshot` work on a `LayerWindow` in a CI run with no display.
- [ ] An idle bar draws zero frames and wakes the loop only for its own timers.
- [ ] The AccessKit tree for a bar is visible to a screen reader.

## Open items

- [ ] Confirm the absences listed under [What tre has today](#what-tre-has-today) in the tre source.
- [ ] Choose the implementation path (I1, I2, I3) after checking whether `winit` can share its Wayland connection.
- [ ] Check which of `wl_output` v4 and `xdg-output-unstable-v1` KWin 6.7 exposes for stable output names.
- [ ] Check whether `window.set(x=, y=, always_on_top=, skip_taskbar=)` does anything on Wayland; it decides how useful the Option C prototype is.
- [ ] Decide keyboard focus for popups under a `"none"` parent.
- [ ] Define "layer" in the Glossary in both senses (tre in-window overlay, protocol stacking layer) and decide whether `LayerWindow` should rename its `layer=` parameter.
- [ ] Find the first KWin release exposing `zwlr_layer_shell_v1` (shared with ADR-0001).
- [ ] Decide whether v1 targets wlroots-family compositors as well as KWin, which affects which compositors the acceptance tests cover.

## Links

- [[ADR-0001 Shell Surface Strategy]]
- [[Prior Art - Linux Shell Surfaces]]
- [wlr-layer-shell-unstable-v1](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)
- [tre Python API](https://mindderivative.github.io/tre/api/python/) · [Custom windowing](https://mindderivative.github.io/tre/design/custom-windowing/) · [Architecture](https://mindderivative.github.io/tre/architecture/)
- [[80 Engine Requests]] · [[Index]]
