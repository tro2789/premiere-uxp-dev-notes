# The QE DOM: undocumented, unsupported, still necessary

QE ("Quality Engineering") is the undocumented ExtendScript test API in
Premiere. Adobe does not support it. A future release can remove it. But
as of Premiere 26.2.x, it is the **only** method to razor clips from code.
The UXP API has no razor function. See
[uxp/timeline-editing.md](../uxp/timeline-editing.md#the-missing-razor-function).
The Adobe PProPanel sample uses QE itself:
[Adobe-CEP/Samples PProPanel/jsx/PPRO/Premiere.jsx](https://github.com/Adobe-CEP/Samples/blob/master/PProPanel/jsx/PPRO/Premiere.jsx).

## The surface (the razor subset, verified on 26.2.2)

```javascript
app.enableQE();                                  // one time per ExtendScript session; idempotent
var qeSeq = qe.project.getActiveSequence();
qeSeq.getVideoTrackAt(i)                         // 0-based, same order as sequence.videoTracks[i]
qeSeq.getAudioTrackAt(i)
qeTrack.razor(tcString)                          // cut ONE track at a timecode
qeSeq.razor(tcString)                            // cut ALL tracks (video and audio)
qeSeq.CTI.timecode                               // playhead position as a timecode string
```

## The timecode format

`razor()` accepts a **timecode string**: `"HH:MM:SS:FF"`. It does not
accept a Time object, a ticks string, or seconds. The documentation does
not show this. The known reference is
[this Adobe forum thread](https://community.adobe.com/questions-729/premiere-pro-scripting-extendscript-question-1396395).
For non-drop-frame rates, convert seconds to frames with integer
calculation ([formula](../uxp/sequences-and-time.md#seconds-to-timecode-ndf)).
Drop-frame sequences need drop-frame conversion. That conversion is not
verified here. Test it before you trust it.

## Behavior facts

- **Per-track `razor()` disconnects audio from video** at the cut
  ([community thread](https://community.adobe.com/t5/premiere-pro-discussions/qe-razor-link-clips/td-p/14848454)).
  This can be a problem or a feature. To cut camera video tracks
  independently of speaker audio tracks, it is the wanted behavior.
- `qeSeq.razor(tc)` also cuts **caption tracks** (verified in a live
  test). This is useful when a removal step needs caption clips divided at
  range boundaries.
- `razor()` returns no documented value. Treat the return as void. Read
  the timeline again to verify the result.
- **Send razors in batches.** Each ExtendScript call from a CEP panel has
  overhead. Send a list of (track, timecode) pairs in ONE ExtendScript
  call, and loop inside ExtendScript. This scales to thousands of razors
  in one call (verified: approximately 1,300 razors on 3 tracks in one
  call, on a 3.5-hour sequence). One call per cut does not scale.
- **A long ExtendScript operation cannot be stopped from the JS side.** If
  a panel-side timeout occurs, ExtendScript continues, and the timeline
  continues to change. Set timeouts for the worst case (minutes, not
  seconds, for batches with thousands of cuts). Design a recovery path for
  an edit that is applied only in part.

## Call ExtendScript without CSInterface.js

`CSInterface` is a thin wrapper. The global below operates directly:

```js
window.__adobe_cep__.evalScript(script, callback);  // callback receives a STRING
```

The return value of the ExtendScript expression becomes a string. Design
your helper functions to return JSON strings. Parse them on the JS side.

## Risk position

Treat each QE call as a risk with a planned exit. Isolate the calls in the
smallest possible component (a helper extension with one function). Keep
the interface small. Be prepared to delete the component when Adobe
supplies a native UXP equivalent, or when Adobe removes QE.

## Address a sequence without a timeline tab

Verified 2026-09-15 (26.5.0) on a batch of 40 sequences:

```js
app.project.activeSequence = seq;            // in ExtendScript
var qeSeq = qe.project.getActiveSequence();  // now addresses seq
```

This does not open a timeline tab. `qe.project.getSequenceAt(i)` throws.

## Effects

```js
var fx = qe.project.getVideoEffectByName("Lumetri Color");
qeClip.addVideoEffect(fx);
qeClip.getComponentAt(k).remove();   // returns true
```

- `addVideoFX` does not exist. Use `addVideoEffect`.
- A new effect is inserted at index 2, after Opacity and Motion. It is
  not added at the end. To get a final order, add the effects in reverse
  order.
- To remove an effect, use the QE component `remove()`. The DOM
  `clip.components[k].remove()` does not exist.
- After a remove, QE and DOM component reads are stale for the rest of
  the same script. Read again in a new call.
- To match a QE item to a DOM clip, use `item.type === "Clip"` and
  `Math.abs(item.start.secs - clip.start.seconds) < 0.02`.
- In Lumetri Color, the Basic Correction values (Exposure, Contrast,
  Highlights, Shadows, Whites, Blacks) are the FIRST property with each
  display name. The same names occur again later for other sections.

Do the structure changes (add, remove) in one call, the values in a
second call, and the check in a third call.

### Premiere 27: check the effect handle

Verified 2026-11-10 (27.0): the Crop effect is now "Crop (Legacy)".
`getVideoEffectByName("Crop")` returns an object with an empty `name`.
`addVideoEffect` with that object returns `false`, with no error. Check
`fx.name` before you use an effect handle.

## Export a frame

```js
qe.project.getActiveSequence().exportFramePNG("00:00:12:00", "D:\\frames\\shot01");
```

Verified 2026-09-22 (26.5):

- The time must be a timecode string in sequence frames. A ticks value or
  a number throws "Illegal Parameter type".
- The path must be a Windows path with backslashes and no extension. QE
  adds `.png`. Forward slashes or a `.png` suffix throw "Unknown error
  exception".
- The sequence must be the active sequence.

## Add a transition

Verified 2026-10-04:

```js
var t = qe.project.getVideoTransitionByName("Blur Dissolve");
qeClip.addTransition(t, true, "00:00:00:10", "00:00:00:00", 0, true, true);
```

Alignment `0` puts the transition on the head of the clip. `0.5` put it
on the tail of the previous clip.
