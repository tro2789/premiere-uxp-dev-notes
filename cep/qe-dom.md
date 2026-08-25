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
