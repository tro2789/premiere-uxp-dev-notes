# Sequences, time, and duration

## `getEndTime().seconds` is the duration. The documented formula is incorrect.

The Adobe documentation implies: sequence duration = `endTime - zeroPoint`.
Do not use this formula.

- `sequence.getEndTime()` returns the duration directly. This was verified
  against a 40-minute sub-sequence with a first frame at timecode
  `02:10:08` (extracted from a longer master sequence).
  `getEndTime().seconds` was equal to the real duration.
- `sequence.getZeroPoint()` returns the **absolute timecode of the first
  frame of the sequence** (for that sub-sequence: `7808.426 s`). It is not
  an offset for subtraction. The formula `endTime - zeroPoint` appears
  correct only because most sequences start at 0. On a sequence with a
  start timecode that is not 0, the formula gives incorrect results.
- `sequence.getTimebase()` returns a string. Adobe does not specify its
  meaning. Do not use it. Get the frame rate with
  `(await sequence.getSettings()).getVideoFrameRate()`.

All `sequence.get*()` methods are async. Use `await` on each one. See
[sync-vs-async](sync-vs-async.md).

## TickTime and FrameRate

```
TickTime.createWithSeconds(seconds)
TickTime.createWithFrameAndFrameRate(frameCount, frameRate)
tickTime.seconds / .ticks / .ticksNumber        (read-only properties)
tickTime.add/subtract/multiply/divide(...)
tickTime.alignToFrame(frameRate) / alignToNearestFrame(frameRate)
FrameRate.createWithValue(fps)
frameRate.value                                  (fps as a number)
```

- `alignToNearestFrame` accepts a **FrameRate object**, not a number.
- `ppro.TickTime.TIME_ZERO` is the standard zero duration for point
  markers.

## Seconds to timecode (NDF)

For non-drop-frame rates, integer frame calculation is sufficient:

```
totalFrames = round(seconds * fps)
hh = floor(totalFrames / (fps*3600))
mm = floor(totalFrames / (fps*60)) % 60
ss = floor(totalFrames / fps) % 60
ff = totalFrames % fps
```

**Drop-frame rates (29.97 DF, 59.94 DF) need drop-frame calculation. That
calculation is not verified here.** If your users have drop-frame
sequences, do a live test before you trust a converter.

## Nested sequences

- For a nested-sequence clip, `trackItem.getProjectItem()` returns a
  generic `ProjectItem`, not a `Sequence`. To get the `Sequence`
  interface, find it by id:

  ```js
  const seqs = await project.getSequences();
  for (const s of seqs) {
    const pi = await s.getProjectItem();
    if (pi.getId() === projectItem.getId()) { /* s is the Sequence */ }
  }
  ```

  Compare with `getId()`. Object identity (`===`) is correct only in some
  cases.
- `getMediaFilePath()` returns `""` for multicam sources and
  nested-sequence sources. These sources have no single file. This is
  correct behavior, not an error.
