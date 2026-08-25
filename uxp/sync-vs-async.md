# Sync and async in the Premiere UXP API: the real rules

The most common cause of defective UXP panel code is an incorrect guess
about which calls return Promises. The API is a mix. The documentation is
not consistent. An incorrect guess causes errors such as
`project.getActiveSequence is not a function`. That error occurs when you
call a method on a Promise that is not resolved.

## The rule table (verified against the Adobe `premiere-api` sample)

| Category | Sync/Async | Examples |
|---|---|---|
| `get*()` methods that cross the JS-to-native bridge | **ASYNC** | `Project.getActiveProject()`, `project.getActiveSequence()`, all `sequence.get*()` getters, `settings.getVideoFrameRate()`, `trackItem.getStartTime/getEndTime/getName/getProjectItem/getInPoint/getOutPoint()`, `Markers.getMarkers(owner)` |
| Action factories (`create*Action`) | **SYNC** | `editor.createOverwriteItemAction()`, `clipProjectItem.createSetInOutPointsAction()`, `markers.createAddMarkerAction()` |
| Transaction wrappers | **SYNC** | `project.lockedAccess(cb)`, `project.executeTransaction(cb, undoLabel)`, `compoundAction.addAction(action)` |
| Static factories (no read of active state) | **SYNC** | `ppro.SequenceEditor.getEditor(sequence)`, `ppro.TickTime.createWithSeconds(s)`, `ppro.FrameRate.createWithValue(v)` |
| Property access | **SYNC** | `.name`, `.value`, `.seconds`, `.ticks`, `frameRate.value`, `track.name`, `projectItem.name` |

General rule: if the call needs native code to give an answer, the call is
async. Local calculations and factories are sync. This rule has exceptions.
See below.

## Verified exceptions (sync `get*()` methods)

- `videoTrack.getTrackItems(type, includeEmpty)` is **sync**. The Adobe
  sample assigns the result directly, with no `await`.
- `clipProjectItem.getMediaFilePath()` is **sync**. It returns `""` for
  multicam sources and nested-sequence sources. These sources do not refer
  to one file.
- Each getter on a `Marker` instance is **sync** (verified with a live
  probe): `getName()`, `getStart()`, `getDuration()`, `getComments()`,
  `getType()`, `getColorIndex()`. These calls cross the bridge but are
  synchronous.

## Verified opposite exceptions (async, but documented as sync)

- `sequence.getCaptionTrackCount()` and `sequence.getCaptionTrack(i)`
  (v25.0 and later). The Adobe documentation shows them as sync. A live
  probe shows that they are **async**. Use `await` on them.

## Recommended procedure

1. Before you add a call, find the same call in the
   [Adobe `premiere-api` sample](https://github.com/AdobeDocs/uxp-premiere-pro-samples/tree/main/sample-panels/premiere-api).
   Copy the await pattern exactly. The sample is a more reliable reference
   than the reference pages.
2. Do not write speculative fallbacks. Examples of speculative fallbacks:
   `obj.foo || obj.getFoo()`, or a wrapper that accepts sync and async
   results. These patterns hide the true shape of the API. Do one probe,
   record the result, and write code for the verified shape.
3. If the sample does not include the class, do a probe. Log
   `typeof result.then` for each call from a test button. A short probe
   prevents long debug sessions.
