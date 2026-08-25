# Markers

The API surface:

```
Markers.getMarkers(owner)                       ASYNC (owner: Sequence | ClipProjectItem)
markers.getMarkers(filters?)                    ASYNC, returns Marker[]
markers.createAddMarkerAction(name, type, start, duration, comments)   SYNC, 5 args
markers.createMoveMarkerAction(marker, tickTime)                       SYNC
markers.createRemoveMarkerAction(marker)                               SYNC
marker.createSetColorByIndexAction(idx)                                SYNC
```

All getters on a `Marker` instance are **sync** (verified with a live
probe). This is an exception to the usual rule that bridge calls are
async. The getters: `getName, getComments, getStart, getDuration, getType,
getColorIndex, getColor, getUrl, getTarget`.

## Marker identity is available only through methods

Markers have **no `.name` property, no `.start` property, and no id**.
Identity is `getName()` (string) plus `getStart().seconds` (number). There
is no other identity.

Why this is important: a read of `m.name` returns `undefined` with no
error. A before/after comparison that uses these properties sees all
markers as identical. A positional fallback ("the last N markers in the
list are the new markers") then selects the incorrect markers, also with
no error. Marker lists are sorted by time. "Last" means last in the
timeline, not created most recently. If you must find markers that you
wrote (for example, to set their color), match by content: record the
`(name, startSeconds)` pairs that you wrote, read the list again, and
match with a sub-frame tolerance. Claim each marker one time. Do not
identify a marker by its list position.

One more trap: `MarkerColor.GREEN === 0`, and 0 is also the Premiere
default marker color. Code that sets the color of the incorrect markers to
GREEN shows no visible change. The defect stays hidden until you use a
different color.

## The `MAGNETA` spelling

The Adobe `MarkerColor` enum spells magenta as **`MAGNETA`** (verified in
a live test). Code that needs magenta must use this spelling. The full key
set: `GREEN, RED, MAGNETA, ORANGE, YELLOW, BLUE, CYAN`. Resolve the value
at run time with `ppro.Constants.MarkerColor[name]` and include a
fallback. If Adobe corrects the spelling, a hardcoded `MAGNETA` fails.

## Types and durations

- The `type` argument is a constant, not a string:
  `ppro.Marker.MARKER_TYPE_COMMENT / _CHAPTER / _WEBLINK / _FLVCUEPOINT`.
  There is **no duration or segmentation type**. A duration marker is a
  `MARKER_TYPE_COMMENT` with `duration > 0`. It renders as a colored bar.
- But `marker.getType()` returns the **string** `"Comment"`, not the
  constant. When you filter by type, compare with the string.
- Duration markers with colors are a good replacement for clip color
  labels. **Track items have no `createSetColorLabelAction`** (verified by
  enumeration of the available methods). Color labels exist only on
  ProjectItems (bin assets), not on timeline instances. Colored duration
  markers are the timeline-visible alternative for "mark these ranges"
  functions.

## The Windows GDI error on marker text

Marker names or comments that contain em dashes, curly quotes, or the
ellipsis character cause a modal error on Windows: **"Unable to measure
string, GDI Status: GenericError"**. Premiere creates the marker. The
dialog is a render complaint, but it appears each time. Convert all text
to ASCII before `createAddMarkerAction`. See
[panel-javascript](panel-javascript.md#text-that-you-write-into-premiere-the-gdi-error).
