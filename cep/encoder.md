# Render a sequence with `app.encoder` from a CEP panel

Verified 2026-08-26 in Premiere Pro 26.2.x (Windows 11) during the
development of [Spec Check](https://github.com/tro2789/SpecCheck). The
panel renders the active sequence's audio to a temp WAV and analyzes it.

## The working call sequence

```js
app.encoder.launchEncoder();
var jobId = app.encoder.encodeSequence(
  seq,          // a Sequence object, e.g. app.project.activeSequence
  outPath,      // absolute output path
  presetPath,   // absolute path to an .epr preset on disk
  workAreaType, // integer, see below
  0             // removeOnCompletion
);
if (!jobId) return "err:AME rejected the job";
app.encoder.startBatch();
```

- Call `launchEncoder()` first. Call `startBatch()` after queueing, or
  the job sits in the AME queue and does not run.
- `encodeSequence` returns a job id string. A falsy return means AME
  rejected the job.

## `workAreaType` values

- `0` = entire sequence. **Verified**: real renders, measured.
- `1` = in point to out point. **Verified**: real renders, measured.
- `2` = work area bar. NEEDS-VERIFY: not yet exercised in a live test.

## The preset must be an `.epr` file on disk

There is no API that lists system presets. Scan Premiere's install
directory for `.epr` files and match by content, not by filename: preset
display names drift across Premiere versions. For an audio-only WAV
render, an auto-discovered system "Waveform Audio" preset works.

## Completion detection

`encodeSequence` gives no reliable completion callback to the panel.
Poll the output file instead: size unchanged for 2 s plus a successful
exclusive open. The mechanics are in
[CEP panel Node.js access](panel-node.md).

## Timeline alignment

For an entire-sequence render (`workAreaType` 0), sequence time maps
1:1 onto the rendered file: a region at marker seconds N..M in the
sequence is at N..M seconds in the WAV. Verified by measuring a marked
region in the render. NEEDS-VERIFY: sequences with a nonzero starting
timecode.
