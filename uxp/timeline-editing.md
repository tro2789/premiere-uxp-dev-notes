# Timeline edit actions: where the documentation and the implementation differ

Timeline changes in UXP use actions: sync `create*Action()` factories,
composed in
`project.lockedAccess(() => project.executeTransaction(cmpd => {...}, "label"))`.
The factories are the least reliable part of the documentation. All
behavior below was observed in live tests on Premiere 26.2.x, on real
multi-track projects.

## `createSetInPointAction(tickTime)` has two undocumented modes

The documentation implies an absolute source-side in-point. The real
behavior depends on the position of the requested in-point:

| Condition | Behavior | Side effect |
|---|---|---|
| Requested in-point > current source out | Extends the source (a true slip) | The clip does not move on the timeline |
| Requested in-point <= current source out | Trims the head | **The clip's sequence start moves forward** by `(newIn - oldIn)` |

No flag selects one mode. If you calculate segment boundaries, the
trim-head mode moves your clip with no error and makes later calculations
incorrect.

## `createSetOutPointAction(tickTime)`: a documentation error

The documentation says: the same coordinate basis as `setInPoint`
(source-side absolute). The real behavior: the argument is a **delta,
added to the sequence start of the track item**. Observed example: a clip
at sequence `[150.58, 150.62]` received `createSetOutPointAction(162.60)`.
The clip end became sequence `301.22` (= 150.58 + 150.64). It did not
become source out `162.60`.

## Other action facts

- `createMoveAction(tickTime)`: the documentation says it "moves the
  inPoint". The words do not make clear if this is a slip or a slide. It
  is a **slide**: it moves the full clip on the timeline by the given
  amount.
- `createSetStartAction(tickTime)`: in our tests, this had **no observed
  effect**. The test case was: move a clip to cancel a shift caused by
  setInPoint. Do not use it for that purpose.
- `SequenceEditor.createOverwriteItemAction` and
  `createInsertProjectItemAction` **ignore source in/out marks** set on
  the source projectItem in the same transaction. Insert applies the marks
  only partially, with offset and duration errors. `limitShift=true` on
  insert does **not** reliably prevent ripple in multi-track cases.
- When you place a **nested-sequence** projectItem, Premiere always
  creates an audio track item on the target audio track. This occurs also
  when the nest contains no audio clips. A Sequence has an audio side in
  its structure. There is no video-only placement. Plan a spare audio
  track, or accept the silent item.
- `sequence.createCloneAction()` can return **`null`**. The cause is not
  documented. Also, `executeTransaction` returns `true` when it commits a
  compound action that is empty. If you use a clone as a backup before a
  destructive edit, examine the action object itself. A `true` transaction
  result does not show that the clone occurred.

## Removal of ranges: `createRemoveItemsAction` and `TrackItemSelection`

```
editor.createRemoveItemsAction(selection, ripple, mediaType, shiftOverLapping)
```

- `ripple: false` = Lift (a gap stays). `ripple: true` = Extract (the gap
  closes). Note: "Lift" means "remove the content and keep the gap". To
  disable clips is a different function (Mute).
- `mediaType: Constants.MediaType.ANY` removes video, audio, and
  **caption** items in one call. See below.
- `shiftOverLapping`: Adobe does not explain the meaning of "overlapping"
  here. This parameter is not verified. Start with `false`.

Selection construction (pattern verified in the Adobe sample):

```js
const sel = await sequence.getSelection();   // ASYNC, the current selection
sel.addItem(item, false);                    // sync, changes the object in place
await sequence.setSelection(sel);
```

Call `clearSelection()` first if you do not want to include the user's
current selection.

## Caption tracks (v25.0 and later)

- `getCaptionTrackCount()` and `getCaptionTrack(i)` are **async**. The
  documentation says sync. Caption track items have the same accessor
  surface as video clip items.
- `selection.addItem(captionItem, false)` is accepted at run time. The
  typed signature lists only video and audio items.
- `MediaType.ANY` includes caption items. The enum has no CAPTION member,
  and the documentation describes ANY as "audio and video". The live
  behavior includes captions.
- If you remove ranges and do not include the caption tracks, the captions
  stay at their original timecodes. An Extract then looks like a Lift with
  captions above the incorrect content. Process the caption tracks
  together with the video and audio tracks.

## The missing razor function

The UXP API has **no razor or blade function** as of Premiere 26.2.x. The
action factories above cannot divide a clip at a timecode in a clean way.
The setInPoint and setOutPoint problems are what you find when you try.
The method that operates today: a CEP companion extension that calls
`razor()` in the QE DOM. See [cep/qe-dom.md](../cep/qe-dom.md). Keep that
interface small. Remove it when Adobe supplies a native function.
