# Sign CEP extensions: a self-signed .zxp without PlayerDebugMode

The common belief: an unsigned CEP extension needs the
`PlayerDebugMode=1` registry key on each user machine. This is true. But
the correction is simpler than most descriptions say: **each validly
signed and timestamped `.zxp` loads with no registry changes.** A
self-signed certificate is sufficient. Adobe Marketplace approval is not
necessary for signing.

## ZXPSignCmd

ZXPSignCmd is the Adobe signing tool. Get it from
[Adobe-CEP/CEP-Resources, ZXPSignCMD/4.1.3](https://github.com/Adobe-CEP/CEP-Resources/tree/master/ZXPSignCMD).
Adobe restricts its distribution. Let your build script download it. Do
not commit it.

```
ZXPSignCmd -selfSignedCert <CC> <ST> <Org> <CN> <password> <out.p12> [-validityDays N]
ZXPSignCmd -sign <inputDir> <out.zxp> <cert.p12> <password> -tsa http://timestamp.digicert.com
ZXPSignCmd -verify <out.zxp> -certInfo -skipOnlineRevocationChecks
```

- **Add the timestamp (`-tsa`) at sign time.** A signed package without a
  timestamp is limited to the validity window of the certificate.
- On a self-signed package, `-verify` reports "Signature verified
  successfully" and "OS Trusted: false". This is the expected result. The
  package loads.
- Keep the same `.p12` for all releases. Make a backup of it. It is the
  identity that user machines know. For a paid or public product, buy a
  code-signing certificate. The other steps do not change.
- Sign Windows packages **on Windows**. A cross-platform signing problem
  from 2024 causes blank panels (see `ZXPSignCMD/KnownIssue2024.md` in
  CEP-Resources). Also make sure that the package contains no symbolic
  links.
- The signature covers **each file** in the package. Before you sign,
  prepare a folder with only the run-time files. Do not include tests or
  build scripts.

## Install the signed .zxp

A `.zxp` is a zip file that contains `META-INF/signatures.xml` and
`mimetype`. The verified install method: **extract the file** into the
per-user CEP folder:

```
%APPDATA%\Adobe\CEP\extensions\<ExtensionId>\
```

With the signature files present, CEP loads the extension **without
PlayerDebugMode**. This was verified: we removed the debug keys from
CSXS.9 through CSXS.12, restarted Premiere, and the extension loaded and
operated. No administrator rights, no UAC, and no installer framework are
necessary. A user-level script with `Expand-Archive` or
`ZipFile.ExtractToDirectory` is a complete installer.

Do not send the `.zxp` through UPIA. UPIA reports success and installs
nothing. See
[upia.md](upia.md#upia-reports-success-for-a-cep-zxp-and-installs-nothing).

## Tell users what to expect

A self-signed certificate causes a Windows SmartScreen warning on a
bundled installer `.exe`. Adobe shows "publisher not verified" text in
some windows. Write this in your documentation, or buy a certificate when
the audience is large enough.
