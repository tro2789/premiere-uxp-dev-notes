# Import FCP 7 XML (xmeml) from a script

Verified October 2026 in Premiere Pro 26.5 on real projects. The import
call is:

```js
app.project.importFiles([xmlPath], true, app.project.rootItem, false);
```

`true` (suppressUI) prevents import warning dialogs. A dialog blocks the
CEP `evalScript` bridge.

## What imports exactly

- A hand-built xmeml v4 file imports with each clip at the exact frame.
  Marker names and comments stay.
- The media and the sequence go into a new bin named `<file>.prproj`.
  Move the items to your own bins after the import.
- `pathurl` in the form `file://localhost/C%3a/...` operates correctly.
- An audio-only `clipitem` that points to a file with picture and sound
  places only the audio. A subclip is not necessary.

## Stereo audio imports as mono

A `clipitem` with `<sourcetrack>` track index 1 imports as a mono clip
(channel 1). `premiereTrackType="Stereo"` on the track does not change
this. A second track that refers to the same clip was dropped. This is
correct for dual-mono speech. Place stereo music with a script
(`importFiles`, then `overwriteClip`).

## Mixed frame rates: write sequence frames

Verified 2026-10-23: for a 24 fps clip in a 30 fps sequence, write the
`clipitem` in and out values in SEQUENCE frames. With the clip's own
`<rate>` of 24 and in and out values in 24 fps frames, Premiere read the
values as 30 fps frames. All in points were at 0.8 times the correct
value.

## Media smaller than the frame

Media smaller than the sequence frame (for example a 1398x2160 panel in
a 3840x2160 sequence) imports centered. Set the Motion position after the
import.

## Do not delete a master clip that a second import used

Verified 2026-10-31: a second import of the same file path can use the
master clip that is already in the project. In one test, a bin that held
such a master was deleted as an "orphan". All clips that used it were
removed from the sequence, with no error. Before you delete a master or
a bin, read the clip project items from a new sequence read. Delete only
empty bins.
