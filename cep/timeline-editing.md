# Timeline edits from ExtendScript

Verified September to November 2026 in Premiere Pro 26.3 to 27.0
(Windows 11), on real podcast projects. For the UXP edit actions, see
[Timeline edit actions](../uxp/timeline-editing.md).

## `insertClip` ripples one track only

- `track.insertClip(item, time)` and
  `sequence.insertClip(item, time, vIndex, aIndex)` ripple only the target
  track. Clips on the other tracks do not move.
- If `time` is in a gap, the shift is the clip length minus the gap. The
  next clip then starts at the end of the inserted clip.
- `clip.remove(true, false)` (ripple delete) also ripples only its own
  track. It closes exactly the removed span.

To keep tracks in sync, insert or remove on each track yourself. Then
read the sequence again and compare it with the edit you intended.

## Sequence markers move once for each insert

Markers after the edit point move one time for each `insertClip` call. If
you insert placeholders on four tracks, each marker moves four times. In
one test, a marker ended 184 seconds late. A ripple delete moves the
markers one time. Record the marker positions before an insert pass, and
set them again after it. See
[Sequence markers in ExtendScript](extendscript-markers.md#move-a-marker).

## Change the range of a placed clip

Set `clip.inPoint` and `clip.outPoint` (Time objects), then `clip.start`
and `clip.end`. Effects, crop, and motion stay on the clip.

- When the clip gets longer, set `outPoint` first.
- When the clip gets shorter, set `inPoint` first.

To trim the head or tail of a raw clip before you place it, set the
source in and out on the placement call. This is easier than a razor and
a ripple delete.

## `move()` removes transitions

`trackItem.move(delta)` moves a clip by a delta. For a positive delta,
move the later clips first. For a negative delta, move the earlier clips
first.

`move()` removes the transitions from the clip. Verified 2026-10-04: a
nested-sequence clip lost its head transition this way. After that, the
clip rendered as invisible on its track. A remove and a new
`overwriteClip` at the same position did not fix it. The clip rendered
correctly on a different track. A new transition from QE fixed it (see
[Add a transition](qe-dom.md#add-a-transition)). After each scripted move,
compare the transitions with the state before the move.

## Many quick moves can corrupt a sequence

Verified 2026-08-26: approximately 32 scripted move calls (a reorder of
podcast sections) corrupted a sequence. The timeline UI lost video and
audio clips step by step. The DOM continued to report every clip at the
correct position. Recovery needed a Premiere auto-save.

- Do not reorder sections with many move calls.
- Before a structural change, make sure that an auto-save or a copy of
  the sequence exists.
- Check the timeline UI, not only the DOM. A DOM read is not proof.

## The DOM can be stale after a razor

The first ExtendScript call after a razor can still see the clips before
the cut. In one test, an insert went onto the wrong clip and overwrote
voice audio. Read the sequence in a separate call to confirm the cut.
Then make the edit in the next call.

## Keyframe times are source-relative

`prop.addKey(t)` and `prop.setValueAtKey(t, value)` take a source time:
the clip `inPoint` plus an offset. They do not take a sequence time. Keys
written at 0 to N seconds on a clip with an `inPoint` of 239 seconds are
all before the visible range, and they have no effect. Interpolation type
4 is Hold.

## Scale to frame size is not applied

Verified 2026-10-02 (26.5.2): a 3840x2160 clip placed by `overwriteClip`
or `insertClip` into a 1920x1080 sequence has Motion scale 100. It shows
the center quarter of the image. The "Scale to frame size" preference is
not applied on this path. Set the scale to 50 for a full frame.

To center a source point (cx, cy), given as fractions of the width and
height, at scale P:

```
x = 960 - (cx - 0.5) * 3840 * P / 100
y = 540 - (cy - 0.5) * 2160 * P / 100
```

Export a frame to check the framing before you trust the numbers (see
[Export a frame](qe-dom.md#export-a-frame)).

## A/V items on audio tracks, and nests

- `audioTracks[i].overwriteClip(item, time)` with an item that has
  picture and sound also puts the picture on a new video track. Remove the
  video clips after the call.
- `overwriteClip` of a nested sequence puts its (empty) audio on the
  matching audio track, and overwrites the audio that is there.

## Remix clips look broken in the DOM

A music clip changed with Remix (Duration > Remix) shows in the DOM as
several pieces with unusual in points. After other edits, it can show as
one clip again. This is not corruption. Check the UI before you "repair"
it.
