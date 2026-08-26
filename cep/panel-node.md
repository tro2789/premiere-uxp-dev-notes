# CEP panel Node.js access

Verified 2026-08-26 with a live probe panel in Premiere Pro 26.2.x
(Windows 11), CEP 12. The manifest enables Node with:

```xml
<CEFCommandLine>
  <Parameter>--enable-nodejs</Parameter>
  <Parameter>--mixed-context</Parameter>
</CEFCommandLine>
```

## `cep_node` is the verified accessor

With the flags above, the global `cep_node` object exists in the panel
page. `cep_node.require` loads Node modules (`fs`, `path`,
`child_process` all verified). `cep_node.process.version` reported
**Node v17.7.2** in Premiere 26.2.x.

A bare global `require` may also exist in mixed context. NEEDS-VERIFY:
the probe found `cep_node` first and did not test the global.

## `getSystemPath` returns a `file:///` URL

The raw API call:

```js
window.__adobe_cep__.getSystemPath("extension")
// "file:///C:/Users/<user>/AppData/Roaming/Adobe/CEP/extensions/<id>"
```

The result is a URL, not a filesystem path. Code that joins it with
`path.join` produces `file:\C:\Users\...` and every `fs` or `spawn`
call on it fails with `ENOENT`.

Adobe's `CSInterface.js` wrapper strips the scheme before returning,
which is why sample code that uses `CSInterface` never shows this
problem. If you call the raw API, strip it yourself:

```js
var extDir = decodeURIComponent(
  window.__adobe_cep__.getSystemPath("extension")
    .replace(/^file:\/{2,3}/, "")
);
```

## A failed `spawn` fires both `error` and `close`

`child_process.spawn` of a path that does not exist emits an `error`
event (`ENOENT`) **and** a `close` event (exit code `-4058`, the libuv
ENOENT value). A callback that advances a candidate list on either
event runs twice and can skip a candidate. Guard with a settled flag so
only the first event acts.

## Watch-folder completion detection for AME renders

Verified with a ~100 s Adobe Media Encoder MP3 encode (174 MB output)
into a subfolder of the watched directory:

- `fs.watch(dir, { recursive: true })` works on win32 from a CEP
  panel. The new output file arrived as a `rename` event whose `name`
  is the path relative to the watched root (for example
  `prproj files\episode.mp3`). Without `recursive: true` the panel
  never sees files that land in subfolders, which is where AME
  normally writes.
- Completion was detected by polling: file size unchanged for 2 s AND
  `fs.openSync(path, "r+")` succeeds (then close the fd). During the
  encode the size never held stable for 2 s, so no false completion
  fired.
- NEEDS-VERIFY: whether AME holds an exclusive write lock for the
  whole encode. The probe never observed a blocked open, because size
  stability and lock release resolved in the same poll tick. Treat the
  open probe as a supplementary check, not as the sole completion
  signal.
