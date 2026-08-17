# Seeding the ipkg status file for pre-baked packages (webOS CE 3.1 image)

Context: the community webOS 3.1 image (in development, not this repo) bakes several packages that
are also offered by this feed directly into the system image — App Catalog (`com.palm.app.enyo-
findapps`) and Preware itself (`org.webosinternals.preware`) so far. Because they were never
installed through `ipkg`, Preware has no record of them and offers them as a plain **install**, even
though an equal or newer version is already on the device. This document is for whoever builds that
image, to seed a fake-but-correct `ipkg` status record so Preware recognizes them as installed.

This is unrelated to feed-side `MinWebOSVersion`/`MaxWebOSVersion` gating (see `CLAUDE.md`), which
solves a different problem (hiding TouchPad-only packages that would be unsafe or redundant to install
on 3.1 at all, e.g. `luna-tls13`). Status-seeding is for packages that **should** stay visible to
Preware on 3.1 but are already present at a specific version, and Preware needs to know that.

## Why this happens

Preware's "is this installed" check is a pure name-match against `ipkg`'s own status database. It
never inspects the filesystem or app registry to see what's actually there.

`isInstalled` is set once, in `infoLoad()`
(`~/Projects/preware/source/app/models/package.js:174`):

```js
if (info.Status && !info.Status.include('not-installed') && !info.Status.include('deinstall'))
{
    this.isInstalled = true;
    ...
}
```

`info.Status` only ever comes from the local `ipkg` status file, fetched via
`IPKGService.getStatusFile` (`packages.js:187`), which calls straight through to native code
(`get_status_file_method`, `~/Projects/preware/source/src/luna_methods.c:1150-1154`):

```c
bool get_status_file_method(LSHandle* lshandle, LSMessage *message, void *ctx) {
  return read_file(message, "/media/cryptofs/apps/usr/lib/ipkg/status");
}
```

If a package was never installed through `ipkg` (e.g. baked directly into the system image), there is
no entry for it in that file, so `isInstalled` stays `false` forever regardless of what's really
there. The feed-sourced package record then falls through `infoUpdate()`'s merge logic
(`package.js:80-161`) to "neither package is installed" (the final `return false` at line 150-154),
and shows up in Preware's plain install list — not because Preware compared versions and decided to
offer a downgrade, but because it never got as far as comparing anything.

## The fix: seed a status record

`/media/cryptofs/apps/usr/lib/ipkg/status` uses the **same stanza format** as this feed's own
`ipkgs/Packages` file — colon-delimited `Key: Value` lines, one blank line between stanzas, parsed by
the identical `parsePackages()` (`packages.js:315-388`) that reads the feed index. Seeding it is just
appending correctly-formed stanzas.

Fields that matter, per `infoLoad()`:

- **`Package:`** — must match the feed's package name exactly (this is the join key `infoUpdate()`
  merges feed and status records on).
- **`Version:`** — the real version baked into the image, not the feed's version. This is what drives
  the "same or older" comparison once merged with the feed record.
- **`Status:`** — must not contain the substrings `not-installed` or `deinstall`. Use the real
  ipkg/opkg convention, `install ok installed`, so anything else that reads this file (`ipkg`/`opkg`
  itself, not just Preware) also parses it correctly.

Optional but worth including for a clean-looking record: `Architecture:`, `Installed-Time:` (unix
seconds), `Installed-Size:` (KB), `Depends:` (blank if none). No `Source:` field is needed — real
`ipkg`-installed packages don't carry one either, and its absence just means `minWebOSVersion`/
`maxWebOSVersion` default wide-open (`'1.0.0'`/`'99.9.9'`, `package.js` constructor defaults), so a
seeded record is never itself blocked by version gating.

### Worked example

Append to `/media/cryptofs/apps/usr/lib/ipkg/status` at image build time (blank-line-separated, same
as any other stanza in that file):

```
Package: org.webosinternals.preware
Version: 1.9.18
Depends: 
Status: install ok installed
Architecture: all
Installed-Time: 1786953600

Package: com.palm.app.enyo-findapps
Version: 6.1.2901
Depends: 
Status: install ok installed
Architecture: all
Installed-Time: 1786953600
```

Once merged (`infoUpdate()`, "Replace Type: 3" — status says installed, feed version is the same or
older), the package correctly shows as already installed with no update or downgrade offered.

## Caveats

1. **This only fixes Preware's bookkeeping.** It does not create
   `/media/cryptofs/apps/usr/lib/ipkg/info/<pkg>.{list,control,postinst,prerm}` — the side files a
   real `ipkg` install writes. If someone tries to genuinely uninstall or reinstall one of these
   packages through Preware, it will likely fail or misbehave, since there's nothing there to run or
   remove. If these components shouldn't be touchable via Preware at all, that's arguably the right
   outcome as-is.
2. **Doesn't replace feed-side gating for anything safety-sensitive.** For packages already excluded
   from webOS 3.1 via `MaxWebOSVersion` in `ipkgs/Packages` (the TLS chain, `accountsapp`,
   `usbsettings`, `btgamepad`, `help-redirect`, `enyo-findapps` — see `CLAUDE.md`), keep that gate
   regardless of whether you also seed a status record. `luna-tls13` in particular rewires
   `/etc/event.d/LunaSysMgr`; even a package showing "installed, no update" can still be explicitly
   re-triggered via Preware's "Reinstall," so the gate is the real safety backstop, not the status
   record. Status-seeding is currently only load-bearing for **Preware itself**, since that one is
   deliberately left ungated (webOS 3.1 users still need it to reach this feed at all).
