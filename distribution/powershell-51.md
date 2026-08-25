# Windows PowerShell 5.1 problems for installer scripts

An end user who selects "Run with PowerShell" gets **Windows PowerShell
5.1**, not PowerShell 7. If you develop and test with `pwsh` only, two
5.1-only failures stay hidden until a user finds them.

## 1. PS 5.1 reads UTF-8 without BOM as ANSI. One em dash stops the script.

PS 5.1 reads a file without a BOM in the **system ANSI codepage**. Each
multibyte UTF-8 character (an em dash, `E2 80 94`; a curly quote; an
ellipsis) becomes incorrect bytes. The parser stops with errors such as
`The string is missing the terminator`. The error points to a line that
looks correct in your editor. PS 7 reads UTF-8 correctly. A test under
`pwsh` does not show this defect.

**Correction:** keep distributed `.ps1` files in **pure ASCII**. Convert
the em dash and en dash to `-`, curly quotes to `'` and `"`, and the
ellipsis to `...`. Or save the file as UTF-8 **with** BOM. Do a parse
check under `powershell.exe`, not only under `pwsh`:

```
powershell -NoProfile -Command "[void][scriptblock]::Create((Get-Content -Raw script.ps1))"
```

## 2. `$PSScriptRoot` is EMPTY in `param()` default values

Under 5.1, this fails:

```powershell
param([string]$Path = (Join-Path $PSScriptRoot "file"))
# result: Cannot bind argument to parameter 'Path' because it is an empty string
```

PowerShell 5.1 does not set `$PSScriptRoot` during the binding of
parameter default values. It sets the variable only in the script
**body**. PS 7 sets the variable in both places. Again, a test under
`pwsh` hides the defect.

**Correction:** give no default value. Resolve the value in the body:

```powershell
param([string]$Path)
if (-not $Path) { $Path = Join-Path $PSScriptRoot "file" }
```

## The general lesson

The runtime that your users have is part of your test matrix. For Windows
plugin installers, that runtime is `powershell.exe` 5.1 with default
settings. Run your installer there one time before each release. Do this
also when the script looks correct under PowerShell 7.
