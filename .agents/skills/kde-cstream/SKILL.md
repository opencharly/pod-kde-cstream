---
name: kde-cstream
description: 'The pod-kde-cstream GPU pod: a fullscreen, hardware-accelerated KDE Plasma desktop streamed to a browser over cstream/WebRTC on :8080, on the CDI pod base.

  MUST be invoked before building, deploying, configuring or troubleshooting the kde-cstream image, its kde-cstream-session candy, or the check-kde-cstream-pod / check-kde-cstream-browser beds.'
---

# kde-cstream — fullscreen KDE Plasma in a browser, GPU-accelerated

## The stack

| layer | provided by | detail |
|---|---|---|
| GPU | pod-cdi (base) | the host driver arrives through CDI (`/etc/cdi/nvidia.yaml`, `/etc/cdi/amd.json`); nothing NVIDIA is baked in |
| Wayland parent + encode + WebRTC | pod-cstream (`cstream-streamer`) | `waylanddisplaysrc` IS the compositor KWin nests into; NVENC through CUDA on an `nvidia` node, VA-API on `amdgpu`/`i915`/`xe` |
| browser surface | pod-cstream (`cstream-gateway`, :8080) | the player, PAM login (`cstream-leader`), the authenticated `/signalling` proxy, `/healthz` |
| desktop | pod-kde-nested (`kde-nested`) | `kwin_wayland --wayland-display wayland-1 --socket wayland-kde` + `plasmashell` |
| this repo | `kde-cstream-session` | points the nested KWin at the streamer: host `wayland-1` in `/tmp/cstream-rt`, KWin on `wayland-kde`, the shared session bus |

There is no sway and no kiosk rule. The streamer's compositor configures its one toplevel at the full output size, so KWin is fullscreen by construction.

## Using it

```bash
charly box build kde-cstream
charly deploy add my-desktop kde-cstream    # then: charly config / charly start
# open http://<host>:8080/ , log in with a local account of the pod
```

| env | meaning |
|---|---|
| `CSTREAM_RENDER_NODE` | the `/dev/dri/renderD*` to composite and encode on. Default: charly's `DRINODE` |
| `CSTREAM_ENCODE` | `auto` (default), `hardware` (fail rather than fall back), `software` |
| `CSTREAM_LOCK_AFTER` | seconds with no viewer before `org.freedesktop.ScreenSaver.Lock`. Default: off |

## The beds (R10)

- **`check-kde-cstream-pod`** checks, in order:
  - the gateway is ready and the streamer reports a **hardware** encoder;
  - KWin composites with OpenGL on the GPU (not llvmpipe), and its output is 1920x1080;
  - a frame decoded **through the WebRTC track** is 1920x1080 and non-uniform;
  - PAM accepts the probe account and rejects a wrong password;
  - **every resolution, live:** for 1366x768, portrait 390x844, 3840x2160, 1918x1078,
    800x600 and back to 1920x1080, the exact size applies, the stream's frames carry it,
    KWin's output follows it, and the FIRST pointer move after the resize lands on the
    exact pixel;
  - the streaming gate: a window maps on the streamer's own compositor, the stream
    changes, and the window goes away;
  - after that, a Konsole gets focus and `cstream: type` / `key` run a command in it (the
    marker is absent before and present after). This proves the desktop got its input
    back after the parent window closed;
  - `cstream: move` puts KWin's cursor where the stream sent it;
  - one wheel notch through the stream reaches a KDE client (`wev`) as exactly Wayland's
    one-notch-down (`value120: 120`).
- Every step that changes the deployment is a `run:` step; `check:` steps only observe.
- **`check-kde-cstream-browser`** runs the same pod plus a headless Chrome member. In stage order:
  - **setup:** Chrome is refused `/signalling` without a session and opens the player, while the pod starts a focused Konsole;
  - **browser:**
    - the wrong password is refused in the page;
    - the probe account logs in;
    - `<video>` plays at `1920x1080`, its decoded frame count rises, and a screenshot shows Plasma;
    - `cdp: type` and Return type a command through the page;
    - the gateway counts the viewer;
  - **verify:** the command ran in the pod;
  - **wheel-observe / wheel-aim / wheel-reached:** Chrome's own mouse and wheel events
    reach `wev` in the pod as one notch down. The pointer's `enter` is asserted before
    the wheel is turned.

Both beds deploy `kde-cstream-check`, which is `kde-cstream` plus **bed-only** fixtures: the `cstream:` verb's probes (including `stream-probe`, which maps a window), a probe account with a known password, and `wev`. Never deploy that image for real use.

## Things that were measured, not assumed

- KWin nested in `waylanddisplaysrc` composites with OpenGL ES on the RTX 4080 SUPER via CDI and on AMD radeonsi. KWin needs `WAYLAND_DISPLAY=<host>` as well as `--wayland-display`; without it the output stays black (pod-kde-nested sets both).
- On NVIDIA, DMABuf does not negotiate into `nvh264enc`, so the CUDA path captures `memory:CUDAMemory` with `cuda-device-id` in PCI bus order.
- Input typed through the stream reaches Konsole verbatim: webrtcsink → GstNavigation → `waylanddisplaysrc`'s seat, keysym names mapped to evdev scancodes.
- Input must be addressed to the video stream's `mid`. With `mid: null`, webrtcsink pushes an event into the audio stream too and acknowledges it if any stream took it (traced with `GST_EVENT:7`), so the acknowledgement proved nothing.
- A window closing on the parent compositor used to strand keyboard and pointer focus. KWin then got 0 keys (traced with `WAYLAND_DEBUG=client`). gst-wayland-display now hands focus back to the topmost remaining window.
- KWin's nested backend ignores the position in `wl_pointer.enter` (`src/backends/wayland/wayland_backend.cpp`, `Pointer::entered`), so the first move after a focus change was lost. gst-wayland-display now follows every enter with a motion at the same position.
- Every size inside the encoder's limits works live, including portrait and 4096 wide. Sizes are fitted only to the encoder's own sink-template limits and to even dimensions.
- The wheel is 120 per notch, positive = up/left (GStreamer's sink convention). The element maps it onto the seat's v120 axis, and `wev` sees one notch down as `value120: 120`.

## Cross-references

- `/charly-selkies:kde-nested`: the nested Plasma session and its `KDE_NESTED_*` knobs.
- `/charly-selkies:kde-vnc`: the same desktop served over VNC instead.
- `/charly-check:check`, `/charly-check:cdp`, `/charly-check:dbus`: the beds' verbs. The `cstream:` verb is in opencharly/plugin-cstream.
