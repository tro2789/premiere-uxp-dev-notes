# UPIA: the Adobe Unified Plugin Installer Agent

End users install a `.ccx` (UXP plugin) with UPIA. Then the UXP Developer
Tool is not necessary:

```
C:\Program Files\Common Files\Adobe\Adobe Desktop Common\RemoteComponents\UPI\UnifiedPluginInstallerAgent\UnifiedPluginInstallerAgent.exe
```

Flags: `/install "<path>"`, `/remove "<displayName>"`, `/list <product>`.
Note: `/remove` accepts the **display name** ("MyPlugin"), not the bundle
id.

All behavior below was verified in live tests (Premiere 26.2.x, Windows
11).

## UPIA is silent, and its exit code is not reliable

UPIA writes nothing to stdout. Its exit code is not reliable. The only
reliable signal is its log:

```
%APPDATA%\Adobe\UPI\Log\Upia.log
```

Search the log for `Installation Successful` or `Removal Successful`
(success), and for `status = -NNN` or `Failed to` (failure). In an
installer script: record the line count of the log before the UPIA call.
Then examine only the new lines.

## `status = -160`: the file is on the incorrect drive

An install of a `.ccx` from a different drive than the Adobe applications
(for example, a repository on `D:\` with Adobe on `C:\`) fails with
`-160`. Spaces in the path also cause problems. The correction: copy the
`.ccx` to a path on the system drive without spaces
(`%ProgramData%\<YourApp>\install-tmp\` is safe for all account names).
Install from that path. This agrees with the Adobe helpx guidance ("run
the .ccx from the same drive as the app").

## An install of the same version does nothing, with no error

`/install` over an installed copy of the **same version** writes
`Installation Successful` to the log and **does not replace the files**.
New machines get your new build. Updated machines keep the old build, with
no error. This defect is very difficult to find in the field. The
correction: call `/remove "<displayName>"` before each `/install` (the
remove is safe when no copy is installed). For real releases, also
increase the manifest version.

## UPIA requires Creative Cloud Desktop

Each UPIA operation goes through Creative Cloud Desktop (CCD). If CCD is
not active, the install fails. Before the install, make sure that one of
these processes is active: `Creative Cloud`, `Adobe Desktop Service`,
`CCXProcess`. If not, tell the user to start Creative Cloud first.

## UPIA reports success for a CEP `.zxp` and installs nothing

`UPIA /install helper.zxp` reports `Installation Successful` (status 0)
and installs **nothing**. No folder appears under
`%APPDATA%\Adobe\CEP\extensions`. No signature appears. UPIA is an
installer for UXP/CCX. It does not process CEP, but it reports success.
(`ExManCmd` is not on standard machines. In our test, it returned `-403`
on a current CEP manifest.) Install a CEP extension by extraction of the
signed `.zxp`. See [zxp-signing.md](zxp-signing.md).

## Where UXP plugins install

```
C:\Program Files\Common Files\Adobe\UXP\Plugins\<windows-username>\External\<bundleId>_<version>\
```

Note the username segment in the path. This is important if you change the
installed copy directly. (The path
`%APPDATA%\Adobe\UXP\Plugins\External\`, shown in some references, is for
the Photoshop family of hosts, not for Premiere.)
