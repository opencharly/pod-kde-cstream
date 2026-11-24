# pod-kde-cstream

A fullscreen, **GPU-accelerated KDE Plasma desktop in your browser**, streamed over
[cstream](https://github.com/opencharly/charly-streamer) (WebRTC), in one pod on the CDI
GPU base.

```
cstream-streamer   the Wayland parent: composites, encodes (NVENC / VA-API), streams
  └─ KWin (nested, fullscreen) ─ plasmashell + KDE apps
cstream-gateway    :8080 — log in, watch, type
```

```bash
charly box build kde-cstream
charly deploy add my-desktop kde-cstream
charly config my-desktop && charly start my-desktop
# browse to http://<host>:8080/ and log in with a local account of the pod
```

The GPU comes from the host through CDI (NVIDIA or AMD). The render node is charly's
`DRINODE` unless you set `CSTREAM_RENDER_NODE`. `CSTREAM_ENCODE=hardware` refuses to fall
back to the CPU, and `CSTREAM_LOCK_AFTER=<seconds>` locks the desktop once nobody is
watching.

Built from [pod-cstream](https://github.com/opencharly/pod-cstream),
[pod-kde-nested](https://github.com/opencharly/pod-kde-nested) and
[pod-cdi](https://github.com/opencharly/pod-cdi).
