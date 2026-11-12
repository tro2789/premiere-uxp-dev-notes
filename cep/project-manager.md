# Project Manager from ExtendScript

Verified November 2026 in Premiere Pro 26.5.2 (Windows 11), for a client
project handoff.

`app.project.consolidateAndTranscode` does not exist, although some
scripts and tools call it. Project Manager is available as
`app.projectManager`:

```js
var pm = app.projectManager;
var o = pm.options;
o.clipTransferOption = o.CLIP_TRANSFER_COPY;
o.excludeUnused = false;
o.renameMedia = false;
o.includePreviews = false;
o.includeConformedAudio = false;
o.destinationPath = "D:\\handoff\\";   // keep the trailing backslash
pm.process(app.project);
```

## The defaults are bad for a handoff

The default options are `excludeUnused = true`, `renameMedia = true`,
`includePreviews = true`, and `clipTransferOption = TRANSCODE`. A handoff
with these values removes media, renames files, and transcodes. Set each
option yourself.

## Result

- Project Manager writes `<destination>\Copied_<project name>\`.
- The source project stays open.

## Check the result in the XML

Project Manager can skip a Media object that has no project item but is
still used by clips. Its `FilePath` stays on the old drive. Open the new
`.prproj` (gzipped XML), find each `FilePath` outside the destination,
and correct it. `ActualMediaFilePath` is not changed. That is normal.
