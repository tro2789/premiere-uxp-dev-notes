# UXP panel JavaScript problems

UXP looks like a browser plus Node-style CommonJS. It is not either of
these. The differences below caused real debug time. All behavior was
verified on Premiere 26.2.x.

## Module loading

- `require("./relative/path/module")` operates (CommonJS, the extension is
  optional). But **`require("./folder")` does not resolve to
  `./folder/index.js`**. The UXP loader does not have the Node directory
  resolution. The error is:
  `Error: Module not found: "./folder". Parent module folder was: "./"`.
  Always give the file name: `require("./folder/index")`.
- **`require("name.uxpaddon")` returns a Promise**, not the addon. Always
  write `await require("name.uxpaddon")`. Without `await`, the failure is
  difficult to find: the Promise object is truthy, it has no keys, and
  each method on it is `undefined`. No error occurs at the call site.
- A bundler is not necessary, and in practice not available. Plain
  CommonJS files load directly. Because of this, npm packages that need
  Node built-ins or a build step (for example, official API SDKs) are
  usually not usable. Use raw `fetch` against HTTP APIs instead.

## Panel start

- Put `<script src="index.js"></script>` in `<head>`. Attach
  `window.addEventListener("load", ...)`. Then use
  `document.getElementById(...)`. The document is the panel.
- A call to `entrypoints.setup({...})` is not necessary in Premiere. That
  pattern is for Photoshop and the SDK template. It does not apply here.

## DOM and CSS limits that fail with no error, or stop the panel

- **Flex `gap` is not fully supported.** Use margins between child
  elements, for example `.item + .item { margin-left: N }`.
- **CSS cannot fully replace the native `<button>` appearance.** The
  platform button shows through rounded corners and custom fills. For each
  styled interactive element (tabs, toolbar buttons), use
  `<div role="button" tabindex="0">` with click handlers.
- **A modal overlay with `position: fixed; inset: 0` can stop the panel
  with no visible window.** The overlay catches input but does not render.
  Use inline error blocks and success blocks in the panel content. Do not
  use overlay modals.
- An exception during startup (for example, a read of `.value` from an
  empty `<select>`) can stop all subsequent handlers. Put try/catch around
  init steps that are not critical. Then one defective dropdown does not
  stop tab navigation.

## Text that you write into Premiere: the GDI error

Premiere on Windows shows a modal error, **"Unable to measure string, GDI
Status: GenericError"**, when it renders marker names or comments that
contain some Unicode characters: em dashes, curly quotes, and the ellipsis
character. Text from LLMs and word processors contains these characters
frequently. Premiere creates the marker correctly. The dialog is a render
complaint, but it interrupts the user each time.

The correction: convert text to ASCII before you write it into Premiere
objects. Convert the dash to `-`, curly quotes to `'` and `"`, the
ellipsis to `...`, and the non-breaking space to a space. Emoji do not
cause the error. This is a Premiere render defect, not a plugin defect.
The same text renders correctly in other parts of the application.

## Deployment: which copy is active?

A UXP plugin can exist in three locations at the same time: your
repository, a staged folder that the UXP Developer Tool loads, and the
**installed copy** at
`C:\Program Files\Common Files\Adobe\UXP\Plugins\<user>\External\<id>_<version>\`.
The installed copy loads automatically with Premiere. A UDT reload changes
only the copy that UDT points to. Also, **when you close and open a
panel, UXP does not load the plugin JS again**. The old code stays in
memory until you reload the plugin or fully restart Premiere. If a change
does not apply, first find which copy the active panel came from.

## Plugin data is stored per Premiere major version

Verified 2026-11-09 (upgrade from 26 to 27): UXP plugin data is in
`%APPDATA%\Adobe\UXP\PluginsStorage\PPRO\<major version>\`. After the
upgrade, the plugin started with no settings. Its data from version 26
was not used. Plan a migration: on the first start, look for the folder
of the previous major version and copy the settings.
