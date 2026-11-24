# AGENTS.md — pod-kde-cstream

The org rulebook at the umbrella (`opencharly/opencharly` `AGENTS.md`) applies in full.
This file adds only what is specific to this repo.

Canonical files:

- `charly.yml` holds:
  - the `kde-cstream-session` candy, which composes pod-cstream and pod-kde-nested and
    points the nested KWin at the streamer's compositor;
  - the `kde-cstream` image (based on `podcdi.pod-cdi`);
  - the bed-only `kde-cstream-check` image;
  - the disposable beds `check-kde-cstream-pod` and `check-kde-cstream-browser`;
  - the `kde-cstream-skill` routing node.
- `.agents/skills/kde-cstream/SKILL.md`: the skill body.
- `.github/workflows/tag-on-merge.yml`: CalVer tag and `CHANGELOG/` on merge, installed by
  `bootstrap-repo-main`.
- `README.md`: user overview only, never agent guidance.

## Load these skills first (R0)

- `/charly-selkies:kde-cstream`: this repo's own skill.
- `/charly-selkies:kde-nested`: the nested Plasma leaf this pod composes.
- `/charly-check:check`, `/charly-check:cdp`, `/charly-check:dbus`: the beds' verbs and the
  R10 sequence.
- `/charly-image:layer`, `/charly-image:image`: the candy and image authoring surface.
- `/charly-internals:git-workflow`: before any git or PR action.

## Rules specific to this repo

- **The pod's GPU comes from the host through CDI.** The base is `podcdi.pod-cdi`. Never
  compose `layer-nvidia` or bake a driver, or the NVENC/CUDA libraries, here.
- **No sway, no kiosk rule, no launcher of our own.** The streamer IS the Wayland parent
  and KWin nests directly in it. The Plasma session is pod-kde-nested's: change its knobs
  here, never copy it.
- **No render-node literal.** The streamer and gateway take `CSTREAM_RENDER_NODE`, else
  charly's `DRINODE`.
- **Input in the beds goes through the stream**: `cstream: type/key/move`, or the browser's
  page in `check-kde-cstream-browser`. Never `wtype`, whose private keymap the nested KWin
  garbles.
- **`kde-cstream-check` is bed-only.** It carries a probe account with a known password.
- **R10 gate:** `charly check run check-kde-cstream-pod` and
  `charly check run check-kde-cstream-browser`, on a host with a GPU exposed through CDI. A
  change to a composed leaf (pod-cstream, pod-kde-nested, layer-gst-wayland-display,
  plugin-cstream) is verified here with `CHARLY_REPO_OVERRIDE=opencharly/<leaf>=<local tree>`
  before its own PR.
