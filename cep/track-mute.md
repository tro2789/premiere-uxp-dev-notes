# Track mute from ExtendScript

Verified 2026-08-26 in Premiere Pro 26.3.2 (Windows 11): a live
read/write/restore test, then production use in a panel
([A/B Audio Toggle](https://github.com/tro2789/ABAudioToggle)).

## The surface

```javascript
var seq = app.project.activeSequence;
var track = seq.audioTracks[i];   // 0-based
track.isMuted();                  // boolean; agrees with the timeline mute button
track.setMute(1);                 // mute the track
track.setMute(0);                 // unmute the track
```

## Verified behavior

- This is the standard DOM. QE is not necessary.
- `isMuted()` returns a real boolean, not a string.
- `setMute()` applies immediately. The timeline mute button updates
  immediately. The change is audible during playback, with no stop and no
  restart. This makes a one-button A/B comparison of two audio chains
  possible.
- Track mute is sequence state, not an edit. Premiere keeps the state in
  the project file. A panel that mutes a "B chain" must offer a reset
  (unmute all) so that an export does not go out with a muted track.
- Track mute is not clip mute (`clip.disabled`). Track mute makes the full
  track silent.

## Limits

- Pass `0` or `1` explicitly. `setMute()` without an argument is not
  tested here. Boolean arguments are not tested here.
- NEEDS-VERIFY: `setMute()` does not appear to create an undo step, which
  agrees with mute as sequence state. Not tested. Do not design around
  undo.
- `videoTracks[i]` has the same methods in the object model, but video
  track behavior is not tested here.

## The panel pattern

One `evalScript` call toggles the full selection and returns the new
states as a JSON string (ExtendScript has no JSON object — build the
string by hand, see [the QE DOM note](qe-dom.md#call-extendscript-without-csinterfacejs)):

```javascript
function toggle(csv) {              // "0,2,3"
  var seq = app.project.activeSequence;
  var idx = String(csv).split(",");
  for (var i = 0; i < idx.length; i++) {
    var t = seq.audioTracks[parseInt(idx[i], 10)];
    t.setMute(t.isMuted() ? 0 : 1);
  }
  // return a JSON string of the new states here
}
```

Read the states again after each write. Do not trust a cached state: the
user can click the timeline mute buttons at any time.
