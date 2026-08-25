# Premiere Pro UXP and CEP Development Notes

This repository contains verified facts about Premiere Pro plugin
development. It records behavior that the Adobe documentation shows
incorrectly, does not show, or that fails without an error message.

Each entry comes from the development of a production plugin
([Scout](https://trevorohare.com), an AI-assisted podcast editing panel, in
development). Each entry has one of two sources:

- A live test in Premiere Pro 26.2.x on Windows.
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
  resolution, `await require()` for addons, CSS and DOM limits
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
  of version ranges, lifecycle events, and panel geometry
- **[The QE DOM](cep/qe-dom.md)**: the undocumented API that is necessary
  for razor operations, and its timecode format

### Distribution (Windows)

- **[UPIA: the Adobe plugin installer](distribution/upia.md)**: the `-160`
  drive error, silent same-version installs, and how to detect success
- **[Sign CEP extensions](distribution/zxp-signing.md)**: a self-signed
  `.zxp` that loads without PlayerDebugMode
- **[Windows PowerShell 5.1 problems](distribution/powershell-51.md)**:
  why an installer script operates in `pwsh` but fails for end users

## Conventions

- "Verified" means: observed in a live test in Premiere Pro 26.2.x
  (Windows 11) on real projects, or quoted from a specified Adobe document
  or sample.
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
- More at [trevorohare.com](https://trevorohare.com).

## License

[CC BY 4.0](LICENSE.md). Use the material freely. Attribution is
appreciated. Send corrections and additions through issues or pull
requests.
