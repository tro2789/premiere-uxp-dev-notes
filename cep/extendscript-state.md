# ExtendScript scope, open projects, and missing APIs

Verified September to November 2026 in Premiere Pro 26.3 to 27.0
(Windows 11).

## Global variables persist between calls

Premiere runs all ExtendScript in one global scope. A variable that one
`evalScript` call declares is still defined in all later calls. This
includes calls from other panels and files loaded with `$.evalFile`.

Verified 2026-10-27: a test call set an export in point in a global
variable. A batch script that did not set that variable read the old
value. Two renders started at the wrong time.

- A script loaded with `$.evalFile` must not read a global that its
  caller does not set. Make each caller pass each parameter, with `null`
  for "off".
- Clear optional globals at the end of the script.
- Do not run test calls while a batch runs.

## More than one open project

- `app.project` can change to a different open project. Verified causes:
  `app.project.deleteSequence()`, and the user switching projects. Check
  `app.project.name` at the start of each script.
- `app.projects[i].openSequence(sequenceID)` on a project that is not
  active brings that project to the front. It becomes `app.project`.
- `app.projects[i].importSequences(prprojPath, [sequenceID])` operates on
  a project that is not active.
- `app.projects[i].save()` on a project that is not active does not write
  the file.
- QE sees only the active project.

## Timeline clip color labels have no API

Verified 2026-08-30 (26.3.2) with a reflection probe on a real clip. A
color label set on a timeline clip (right-click > Label) cannot be read
or written from UXP, from the ExtendScript DOM, or from QE.

- The ExtendScript `trackItem` has no label member. Its properties are
  `components, disabled, duration, end, inPoint, matchName, mediaType,
  name, nodeId, outPoint, parentTrackIndex, projectItem, start, type`.
  Its methods are `getLinkedItems, getMGTComponent, getSpeed,
  isAdjustmentLayer, isMGT, isSelected, isSpeedReversed, move, remove,
  setSelected`.
- The QE clip has `borderColor` and `setBorderColor`. These are the
  stroke, not the label.
- In UXP, the color label API is on `ProjectItem` only.

Effects of this:

- A razor (QE, or a multi-camera edit in the UI) makes new clip
  instances. The new instances use the label of their project item, so a
  timeline label is lost.
- A label set on the project item is scriptable. Timeline instances use
  it, and the two halves of a razored clip keep it.
- A tool that selects clips by the project item label selects the wrong
  clips when a user labels timeline instances.

To read timeline labels, save the project and parse the `.prproj` file
(gzipped XML). Each `VideoClipTrackItem` has `Start` (not present when
it is 0) and `End` in ticks (254016000000 per second). Follow its
`SubClip` reference to the `Clip`. The `asl.clip.label.name` value
(`BE.Prefs.LabelColors.N`) is the label.

## There is no clipboard API

ExtendScript cannot copy and paste clips. If you build clips again from
the source, you lose Essential Graphics parameters, keyframes, and speed
changes. A paste with keystrokes works (verified 2026-10-15):

1. Make sure that the Timeline panel has focus.
2. Open the source sequence with `project.openSequence(id)` and set
   `project.activeSequence`. The focus stays on the timeline.
3. Call `trackItem.setSelected(1, 1)` on the clips to copy and
   `setSelected(0, 1)` on all others. Send Ctrl+C.
4. Open the destination sequence. Set the target tracks with
   `track.setTargeted(true, true)` and the playhead with
   `sequence.setPlayerPosition(ticks)`. Send Ctrl+V. The paste puts the
   lowest copied track on the lowest targeted track.

Do not send Shift+3 to select the timeline. It moves through the timeline
tabs of all open projects, and it can bring a different project to the
front. Check `app.project.name` before each Ctrl+C and Ctrl+V.

## Graphics text

Set the text of an Essential Graphics template through its "Graphic
Parameters" component. Property "Text N" holds a JSON string. Replace
the `"textEditValue":"..."` value in that string and call
`setValue(json, true)`. If you set the "Source Text" of a native graphic
to a plain string, the graphic renders blank.

## Other facts

- `sequence.clone()` copies caption tracks. ExtendScript and QE cannot
  see caption tracks (`numCaptionTracks` is undefined), so a script
  cannot remove them from the copy.
- `projectItem.changeMediaPath(newPath, true)` replaces the media of an
  item in place. Clip timing on the timeline stays the same.
- `app.project.importFiles` with a plain `"C:\\..."` string in a file
  loaded with `$.evalFile` failed with an "unable to open files" dialog.
  A modal dialog blocks the `evalScript` bridge until a user closes it.
  Pass `new File(path).fsName`.
