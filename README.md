# Premiere Pro UXP and CEP Development Notes

This repository contains verified facts about Premiere Pro plugin
development. It records behavior that the Adobe documentation shows
incorrectly, does not show, or that fails without an error message.

Each entry comes from the development of a production plugin
([Scout](https://www.trevorohare.com/dev), an AI-assisted podcast editing panel, in
development). Each entry has one of two sources:

- A live test in Premiere Pro 26.2 to 27.0 on Windows.
- A specified Adobe document or Adobe sample.

No entry is a guess. If an entry is not verified, the entry says so. The
date on an entry shows when the behavior was observed. Adobe can change
behavior in each release.

## Why this repository exists

The Premiere plugin developer community is small. The official
documentation is not complete. Many documented APIs do not operate as
documented. Many incorrect manifest values fail with no error message.
These notes record the verified behavior so that you do not lose the same
time.

## Contents

### UXP (the current API)

- **[Sync and async: the real rules](uxp/sync-vs-async.md)**: which calls
  return Promises, which calls do not, and the known exceptions
- **[Manifest problems](uxp/manifest-gotchas.md)**: the `entrypoints`
  spelling, the loopback whitelist rejection, silent clipboard-permission
  failures
- **[Panel JavaScript problems](uxp/panel-javascript.md)**: CommonJS
  resolution, `await require()` for addons, CSS and DOM limits, and
  plugin data that is stored per Premiere major version
- **[Sequences, time, and duration](uxp/sequences-and-time.md)**: why
  `getEndTime()` is the duration and why the documented formula is
  incorrect
- **[Markers](uxp/markers.md)**: method-only identity, the `MAGNETA`
  spelling, the Windows GDI error on special characters
- **[Timeline edit actions](uxp/timeline-editing.md)**: the true behavior
  of `createSetInPointAction`, the `createSetOutPointAction` documentation
  error, caption tracks, and the missing razor function
- **[Hybrid C++ addons](uxp/hybrid-addon.md)**: the thread model, Promises
  from native code, build facts

### CEP (the old API, still necessary)

- **[CEP manifest problems](cep/manifest-gotchas.md)**: silent rejections
  of version ranges, lifecycle events, panel geometry, and broken
  development installs through junctions
- **[CEP panel Node.js access](cep/panel-node.md)**: `cep_node.require`,
  the `file:///` URL that `getSystemPath` returns, double-firing failed
  spawns, watch-folder completion detection for AME renders, and why
  `localStorage` loses user data
- **[The QE DOM](cep/qe-dom.md)**: the undocumented API that is necessary
  for razor operations, its timecode format, effects, frame export, and
  transitions
- **[Timeline edits from ExtendScript](cep/timeline-editing.md)**:
  one-track ripple, marker drift, `move()` removing transitions,
  source-relative keyframe times, and scale to frame size
- **[ExtendScript scope, open projects, and missing APIs](cep/extendscript-state.md)**:
  persistent globals, several open projects, timeline color labels, a
  paste recipe, and graphics text
- **[Import FCP 7 XML](cep/fcp-xml-import.md)**: frame-exact import,
  stereo to mono, mixed frame rates, and reused master clips
- **[Project Manager](cep/project-manager.md)**: run it headless with
  options that are safe for a handoff
- **[Render a sequence with app.encoder](cep/encoder.md)**: the working
  `encodeSequence` call order, `workAreaType` values, `.epr` preset
  discovery, and completion detection
- **[Sequence markers in ExtendScript](cep/extendscript-markers.md)**:
  linked-list iteration, Time objects, the point-marker trap, and how to
  move a marker
- **[Track mute](cep/track-mute.md)**: `isMuted()` and `setMute()` on
  `audioTracks`, verified behavior, and the A/B toggle pattern

### Distribution (Windows)

- **[UPIA: the Adobe plugin installer](distribution/upia.md)**: the `-160`
  drive error, silent same-version installs, and how to detect success
- **[Sign CEP extensions](distribution/zxp-signing.md)**: a self-signed
  `.zxp` that loads without PlayerDebugMode
- **[Windows PowerShell 5.1 problems](distribution/powershell-51.md)**:
  why an installer script operates in `pwsh` but fails for end users

## Conventions

- "Verified" means: observed in a live test in Premiere Pro 26.2 to 27.0
  (Windows 11) on real projects, or quoted from a specified Adobe document
  or sample. Each entry gives the version that it was tested in.
- "NEEDS-VERIFY" means: a possible interpretation of the documentation
  that is not tested. Treat it as a hypothesis.
- Sample-verified patterns refer to
  [AdobeDocs/uxp-premiere-pro-samples](https://github.com/AdobeDocs/uxp-premiere-pro-samples).
  When the documentation is not clear, the Adobe sample code is the better
  reference.

## Related

- [Sequence Batch Exporter](https://github.com/tro2789/SequenceBatchExporter):
  a free CEP panel. It sends multiple sequences with multiple export
  presets to Adobe Media Encoder in one operation.
- [Spec Check](https://github.com/tro2789/SpecCheck): a free CEP panel.
  It measures a mix against a delivery spec (ACX/Audible, Apple
  Podcasts, Spotify, EBU R128, YouTube, custom) and shows a pass/fail
  scoreboard.
- [A/B Audio Toggle](https://github.com/tro2789/ABAudioToggle): a free CEP
  panel. It changes the mute state of selected audio tracks with one
  button, for A/B comparison of two audio chains.
- More at [trevorohare.com/dev](https://www.trevorohare.com/dev).

## License

[CC BY 4.0](LICENSE.md). Use the material freely. Attribution is
appreciated. Send corrections and additions through issues or pull
requests.
