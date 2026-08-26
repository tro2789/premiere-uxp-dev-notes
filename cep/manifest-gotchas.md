# CEP manifest problems: silent rejections

CEP (the old extension platform) is still necessary in 2026. It is the
only route to ExtendScript and the QE DOM from a current plugin. Its
failure mode is worse than UXP: **the extension does not start, and no log
shows a reason.** All behavior was verified on Premiere 26.2.x with CEP
12. Several entries were found only by examination of the manifest of an
installed commercial extension.

| Element | The problem |
|---|---|
| `<RequiredRuntime Name="CSXS" Version="9.0"/>` | The version is a **minimum**. 9.0 covers CC2019 and later. No problem here. |
| `<Host Name="PPRO" Version="14.0"/>` | The cookbook shows a range notation (`Version="[14.4,15.9]"`), and the schema accepts it. But Premiere 26.2.2 **rejected a wide range** (`[14.0,99.9]`) with no error: the manifest parses, and the extension does not load. Use the single-value form. The value is a minimum. |
| Lifecycle `StartOn` event | With the cookbook's `applicationActivate` event, an invisible extension **does not start**, with no error. Use the full event name `com.adobe.csxs.events.ApplicationInitialized`. |
| `<UI>` geometry for invisible extensions | Geometry is necessary also when the extension is never shown. The cookbook's `1x1` example is **rejected with no error**. CEP has a minimum width (approximately 385 px, from the CEP log: `'310' should not be less than minimum width: '385'`). Use a real size, for example 640x320, with `<Type>Custom</Type>` and `AutoVisible=false`. |
| `<CEFCommandLine>` | Node.js is **not enabled** in CEP 11/12 by default. Add the parameters `--enable-nodejs` and `--mixed-context`. Then `require()` and the other Node globals are available in the page. This is necessary for functions such as `http.createServer`. |
| XML comments | The XML specification does not permit `--` in a comment. A comment such as `<!-- use --enable-nodejs -->` breaks the manifest. Write flag names in words. |

## Debug procedure

When a CEP extension does not appear:

1. Examine the CEP logs (`%TEMP%\csxs*.log`, `cef_debug.log`). Geometry
   violations appear there. Most other rejections do not.
2. **Find an extension on your machine that operates** (a commercial CEP
   plugin under `Program Files (x86)\Common Files\Adobe\CEP\extensions` or
   in the per-user folder). Compare your manifest with its manifest,
   element by element. The manifest of an installed extension shows what
   this Premiere version accepts. It is more reliable than the cookbook.
3. Change one element at a time. There are multiple silent failure points.
   A change of many elements at the same time gives no information.

## Install locations (Windows)

- Per user: `%APPDATA%\Adobe\CEP\extensions\<ExtensionId>\`
- System: `C:\Program Files (x86)\Common Files\Adobe\CEP\extensions`

An unsigned extension also needs `PlayerDebugMode=1` (string) under
`HKCU\Software\Adobe\CSXS.<ver>`. A signed package does not need this key.
See [distribution/zxp-signing.md](../distribution/zxp-signing.md).

## Media Encoder is not a CEP host — an AME HostList entry crashes AME

Verified 2026-08-26, Adobe Media Encoder 2026 (Windows 11), during Spec
Check development.

Media Encoder looks like a CEP host: it ships `PlugPlug.dll` and
`csxsmanager.dll` and has a Window > Extensions menu. It is not one. The
official CEP cookbook host table (CEP 11 and CEP 12) lists twelve
applications — including Prelude, Bridge, and Rush — and Media Encoder is
in neither list.

Adding `<Host Name="AME" Version="[22.0,99.9]"/>` to a panel's HostList
made the panel appear in AME's Extensions menu, and opening it crashed
AME with an access violation. The minidump shows the fault in
`dvaworkspace.dll` (Adobe's panel-docking framework) while creating the
panel view; CEP's CEF browser never started, so no panel code ran. The
crash is in the host, before your extension executes anything.

Community reports show the other failure mode on AME 24 and 25.5: the
extension never appears in the menu at all. Either way, do not declare
AME. If you need QC or automation on AME renders, run the panel in
Premiere and watch AME's output folder from there.
