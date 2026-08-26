# Sequence markers in ExtendScript (CEP)

Verified 2026-08-26 in Premiere Pro 26.2.x (Windows 11) during the
development of [Spec Check](https://github.com/tro2789/SpecCheck). For
the UXP marker API, see [Markers](../uxp/markers.md); the ExtendScript
API below is separate and behaves differently.

## Iteration is by linked list, not by index

```js
var ms = app.project.activeSequence.markers;
if (ms && ms.numMarkers > 0) {
  var m = ms.getFirstMarker();
  while (m) {
    // m.name, m.start.seconds, m.end.seconds
    m = ms.getNextMarker(m);
  }
}
```

- `getFirstMarker()` / `getNextMarker(m)` returned markers in timeline
  order in live tests.
- `m.name` is a plain string. Compare case-insensitively if users type
  the name.

## Time is in seconds via Time objects

`m.start` and `m.end` are Time objects. `.seconds` gives a float. For
an entire-sequence render, these seconds map 1:1 onto the rendered
file (see [Render a sequence with app.encoder](encoder.md)).

## Point markers: `end` equals `start`

A point marker (no duration) reports `m.end.seconds === m.start.seconds`.
Test `end - start` against a small epsilon to tell point markers from
duration markers. Do not treat a point marker as a region: in Spec
Check, sampling a fixed window forward from a point marker read
adjacent program audio instead of the quiet gap the user meant.
