# UXP manifest problems (Premiere Pro)

A manifest error in UXP usually does not cause an error message. It causes
silence: a permission does not apply, or an entrypoint does not appear.
All behavior below was verified on Premiere 26.2.x.

## Field-level facts

| Field | Important behavior |
|---|---|
| `manifestVersion: 6` | Necessary for hybrid (C++ addon) plugins. Version 5 is sufficient for panels without addons. |
| `host: [...]` | This is an **array of objects**, not one object: `[{ "app": "premierepro", "minVersion": "26.2.0" }]`. |
| `entrypoints` | Write it in **lowercase, as one word**. Premiere ignores the spelling `entryPoints` with no error. The panel does not appear. |
| `entrypoints[].label` | Necessary: `{ "default": "..." }`. |
| `addon: { name: "<file>.uxpaddon" }` | Give the filename only. UXP finds the file at `win/x64/<file>.uxpaddon` (or `mac/*`). This also requires `requiredPermissions.enableAddon: true`. |
| `main: "index.html"` | This is the entry for the full plugin. The HTML `<body>` is the panel content. There is no shadow DOM and no per-entrypoint main. |

An incorrect package layout causes "Plugin Manifest Validation Failed" in
the UXP Developer Tool. This is the correct layout:

```
plugin-root/
├── manifest.json
├── index.html
└── win/x64/<name>.uxpaddon      (mac/x64, mac/arm64 for cross-platform)
```

## The loopback whitelist rejection

`requiredPermissions.network.domains: ["http://127.0.0.1:8000"]` does not
operate. The UXP domain whitelist **rejects loopback addresses**
(`127.0.0.1`, `localhost`) with no error at load time. A `fetch()` call
then fails with "Permission denied to the url ... Manifest entry not
found". The address is in the array, but UXP does not accept it. Addresses
on the local network (for example `192.168.x.x`) are accepted. Only
loopback is rejected. Adobe tracks this at
[AdobeDocs/uxp-photoshop#321](https://github.com/AdobeDocs/uxp-photoshop/issues/321).

The workaround is `"domains": "all"`. This is a documented value that
Adobe supports. It is a wide permission: the panel can then connect to all
hosts. Reviewers and users can ask why. But it is the only method to
connect to a local helper process today.

`WebSocket` connections use the same `network.domains` gate as `fetch()`.
With `"all"` set, `ws://127.0.0.1:<port>` operates correctly (verified:
the constructor returns synchronously, and the `open` event occurs in
approximately 50 ms against a local server).

## Silent permission failure: clipboard

`navigator.clipboard.writeText()` requires
`"clipboard": "readAndWrite"` under `requiredPermissions`. Without this
permission, **the call completes with no error**. The Promise resolves. No
exception occurs. The clipboard does not change. This is the same silent
failure type as the loopback rejection. UXP usually hides permission
denials. It does not throw them.

Also: **UXP reads the manifest again only at plugin reload** (the reload
button in the UXP Developer Tool). It does not read the manifest when you
close and open the panel. If a new permission does not apply, reload the
plugin first.

## General lesson

When a UXP function does not operate, examine the manifest first. Do not
expect an error message. Compare your manifest with a known-good Adobe
sample
([uxp-premiere-pro-samples](https://github.com/AdobeDocs/uxp-premiere-pro-samples))
before you change the code.
