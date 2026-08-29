# CLAUDE.md

Guidance + hard-won context for this repo. Read this first when resuming.

## Current state / next steps (resume here)

The feed (35 pkgs) is built, live and **hardware-verified on both device families from ONE feed URL**
— TouchPad and webOS 2.2.4 phones use the same
`http://stacks.webosarchive.org/feeds/modernize/ipkgs`.

- **TouchPad (topaz):** verified with `tls-updates` **1.0.8**. Flow on a freshly-Doctored 3.0.x:
  enable Dev Mode → WOSQI-install **Preware 1.9.17** (in the feed; carries the modernize feed in its
  postinst) → Update Feeds → install **`org.webosarchive.tls-updates`** (the one-tap bundle: SSL/TLS
  stack + root certs + NTP clock sync + mail/mojomail fix + Help redirect + Enyo App Catalog + USB
  Settings + Bluetooth Gamepad Support; **no** QupZilla/Atlas/LunaCE). Help app + content
  (help.webosarchive.org) work end-to-end.
- **HP Pre 3 (mantaray): VERIFIED ON HARDWARE.** `org.webosarchive.tls-updates-phone` **1.0.1** →
  the `*-phone` package family. This confirms the two values that were previously unverified guesses:
  `DeviceCompatibility: "Pre3"` matches the real `modelNameAscii` (wrong string ⇒ the packages would
  have been invisible in Preware), and the postinst board detection resolves a Pre 3 correctly
  (wrong ⇒ it would have refused to patch). See "Phone support" below.
- **HP Veer (broadway) and Palm Pre 2 (roadrunner): in the feed, NOT yet tested on hardware.** Same
  merged packages, same detection path, different `case` arm — the plausible failure is a wrong
  `modelNameAscii` (`"Veer"` / `"Pre2"`) making them invisible in Preware, which the
  Preferences → *Ignore Device Compat.* toggle diagnoses in one step.

**Atlas (modern browser) + "make Atlas default" patch, both in the feed:**
`org.webosports.app.atlas` (**0.9.12**, WPE WebKit 2.52 browser, 103MB) and
`org.webosarchive.atlas-default-browser` (**1.0.0**, the redirect+hide patch). Dep chain:
`atlas-default-browser → org.webosports.app.atlas → org.webosarchive.tls-updates` (Atlas links the
ssl11 OpenSSL 1.1 stack for HTTPS). Installing the patch from the feed on a fresh device pulls Atlas
→ tls-updates → the whole TLS stack, then Preware prompts one restart at the end (RestartDevice from
tls-updates when it's in the batch; RestartLuna if tls-updates is already installed). **Verified on
device**: Atlas engine (BrowserServer-atlas) runs, NPAPI adapter + ls2 roles + db8 kinds + upstart
registered, redirect applied, no mid-install reboot. See the two subsections under Package inventory.

**`org.webosarchive.open-in-atlas` (1.0.0), the gentler alternative:** some users
found the all-or-nothing redirect too much, so this patch leaves the stock browser fully working and
just adds an **"Open in Atlas"** App Menu item that hands the current page to Atlas via
`palm://com.palm.applicationManager/open {id, params:{target}}`. Same `Depends:
org.webosports.app.atlas`, same index.html backup/restore pattern, **mutually exclusive** with
atlas-default-browser (it detects that patch and bails). **Verified on device** — menu item appears,
hands the current page to Atlas, stock browser otherwise unaffected. (Pre-device checks that paid off
and are worth repeating for any Enyo-app patch: run the injected JS in node against the app's real
kind file with an Enyo-1-faithful `enyo.kind` stub, and dry-run postinst/prerm against a copy of the
target file with `BROWSER_DIR` re-pointed.)

Latest revs (deployed): `browser-tls13 1.1.2`, `luna-tls13 1.1.4` (1.1.0 was faulty — media wedged;
1.1.1 added the media-pipeline wrapper; 1.1.2 added a `/usr/sbin/setcpushares-pdk` wrapper so
PDK apps — QupZilla, nizovn Qt5 — launch normally, incl. under LunaCE; **1.1.4** current —
⚠️ note **1.1.4 is cosmetic-only on the TouchPad**: four reworded postinst/prerm echo strings, payload
byte-identical apart from the version. It was bumped for symmetry with the phone 1.1.4, which is where
the real fixes are. See the luna-tls13 history bullet),
`mail-tls13 1.3.2` (**1.3.2** fixes Gmail/ECDSA-cert IMAP/POP sign-in that falsely failed with
"certificate is not trusted"/error 4010 — the imap/pop/smtp launchers now pin TLS 1.2 + an RSA
server cert), `downloadmgr-tls13 1.1.0` (RPATHs the system Download Manager /
`/usr/bin/LunaDownloadMgr` onto the ssl11 curl so background downloads AND uploads reach modern
HTTPS; depends browser-tls13. **1.1.0** adds the one-byte code patch that stops the SIGSEGV in
`curl_multi_remove_handle`, and repairs a poisoned uninstall restore point — see the package
bullet), `usbsettings 1.1.9` (**new** — a mounted USB drive now lands on
`/media/usb` instead of a folder inside `/media/internal`, so it stops showing as an empty folder in
Internalz Pro and stops swallowing files onto internal storage; see the package bullet),
`com.palm.app.backup 3.1.1` (**3.1.1** fixes a restore that silently restored only part of a backup
and still reported success; see the package bullet), `tls-updates 1.0.19`
(version-floors browser `>= 1.1.2` / usbsettings `>= 1.1.9` / btgamepad `>= 1.1.0` / luna
`>= 1.1.4` / mail `>= 1.3.2` / downloadmgr `>= 1.1.0` / **`com.palm.app.accounts >= 3.1.1`** /
**`com.palm.app.backup >= 3.1.1`**; also
pulls in `ntpdate-sync`, `downloadmgr-tls13` and `com.palm.app.backup` — apps break
when the clock is wrong, TLS cert validity checks fail).

**Synergy Revival is DEPLOYED and hardware-verified.** ~~Branch `synergy-connectors` is NOT MERGED~~
— **merged as of 2026-08-29**: `synergy-connectors` is now a fully-merged ancestor of `main`, and `main`
is ahead of it (the connector re-cuts and the SIGBUS fix landed there). Build from `main`.
Four upstream installer bugs were found and fixed along the way — see "Four upstream installer
bugs" below before touching any of these packages.
34 new packages (Part 1 runtime + core-app updates, 20 pick-and-choose connectors, and our
`org.webosarchive.synergy-revival 1.0.0` roll-up) take the feed to **69 packages / 90 stanzas**. The same
work retired `org.webosarchive.accountsapp` in favour of `com.palm.app.accounts` 3.1.1 and took
`tls-updates` to **1.0.18**. Index checks all pass; **nothing has been run on hardware yet.** Read the
"Synergy Revival" section before touching any of it.

QupZilla/Qt5 chain now carries version floors so installing QupZilla drags the Qt5 stack up:
`qupzilla → qt5sdk (>= 1.0.2), qt5qpaplugins (>= 1.0.4)` (the qt5qpaplugins floor is a **direct**
edge, added because Preware won't recurse into the already-installed qt5/qt5sdk nodes to notice
qt5's own floor); `qt5sdk → qt5 (>= 5.9.7-0)`; `qt5 → qt5qpaplugins (>= 1.0.4)`. This reverses the
earlier "leave the depends tree alone" stance for this chain (index-only edit; ipks kept as-delivered).

**Open / TODO:**
- **Synergy Revival: hardware test, then merge + deploy.** Branch `synergy-connectors`. Test order in the
  Synergy section (3.0.5 TouchPad first: meta, then one OAuth connector and one IM connector; then a CE
  3.1.0 device where the connector must install with no Part 1). Two things to report to Herrie while
  testing: the shared prerm never restores (reads a `dest.txt` its postinst deleted), and no script has
  an OS/board guard, so WOSQI/`ipkg install` bypass every device check. Also unconfirmed: which Atlas
  build carries the `setWindowSize` mult=1 viewport fix the OAuth webview needs (we floored at 0.9.11).
- **CE 3.1.0 gate drift — the Atlas half is FIXED (2026-08-23), the rest stands.** Atlas no longer
  has an unresolvable dep at 3.1.0: it now ships two stanzas and the 3.1.0 one carries no `Depends`
  at all (see "Atlas's two stanzas"). What remains is a *coverage* question, not a broken dep — a
  device reporting `3.1.0` sees 35 of 69 packages and gets **no one-tap meta**, because every
  TouchPad stanza except Atlas's and the Synergy connectors' sits at `Max 3.0.9`. (Synergy made CE 3.1.0
  a first-class target for the first time: its 20 connectors ship a dep-free 3.1.0 stanza each.) Still a gating-policy call: either move them back
  to `3.9.9`, or accept CE 3.1.0 as out of scope. The two `atlas-*` patch packages are `Max 3.9.9` and
  depend only on Atlas, so they are fine either way.
- **~~`accountsapp` floor drift~~ — MOOT since 2026-08-24: the package is retired and out of the feed.**
  Kept for the reasoning, which still applies to any `Min 3.0.0` package whose deps are `Min 3.0.5`:
  `org.webosarchive.accountsapp` was `Min 3.0.0` but its two deps (`browser-tls13`, `luna-tls13`) are
  `Min 3.0.5`, so on a TouchPad reporting 3.0.0 / 3.0.2 / 3.0.4 the app is visible and neither dep is
  → unresolvable, two dep gaps. Verified identical in `git show HEAD:ipkgs/Packages`, so it predates
  this session. Fix is one index edit — raise accountsapp to `Min 3.0.5`, matching the rest of the TLS
  chain — but it's a gating-policy call, so left alone. Only shows up if you sweep the whole 3.0.x
  range; the standard check only simulates 3.0.5.
- **Veer (broadway) + Pre 2 (roadrunner)** — shipped and gated but never run on hardware. Test as for
  the Pre 3. Worth capturing while a phone is attached:
  `cat /etc/prefs/properties/machineName` per device, so the board-detection values are recorded
  rather than inferred (only `roadrunner` has independent corroboration, from Preware's own source).
- **`~/Projects/preware`** (Preware 1.9.17 source: version bump, http modernize feed, injected
  control.tar.gz postinst) is still **uncommitted** by the user's instruction — commit/push when ready.
- **`~/Projects/webos-docs`** (the user-facing MkDocs site, `github.com/webosarchive/webos-docs`) —
  11 files **uncommitted**: the two-path tablet-vs-phone layout was collapsed into one path now that
  phones have native TLS (see "User-facing docs" below). Commit/push when ready.
- **OTA / `swupdate-redirect`** held in `staging/` — needs the UpdateDaemon carrier/domain binary
  patch (see `staging/swupdate-redirect-FINDINGS.md`) before the OTA-in-a-feed goal is real.
- **`webOS CE 3.1.0` / `luna-update`** held in `staging/` — the LunaCE LunaSysMgr swap must become
  its own opt-in, tested-standalone package (it bricked a device combined with luna-tls13). The
  tls-updates bundle is the safe stand-in for now.

## What this is

A **Preware feed** ("WOSA Modernize" / `modernize`) of `.ipk` packages that modernize
HP webOS 3.0.x TouchPads (and some Pre3). Served at:

- **Live:** `http://stacks.webosarchive.org/feeds/modernize/ipkgs/` (http AND https; use
  **http** in anything a stock device must reach — its TLS 1.0 stack can't do modern https,
  and this feed is what *delivers* the TLS fix).
- **Git:** `git@github.com:webOSArchive/preware-modernize-feed` (branch `main`).
- Add in Preware: Manage Feeds → URL `http://stacks.webosarchive.org/feeds/modernize/ipkgs`,
  Compressed (gzip) **on**.

`ipkgs/` holds the `.ipk` files + the feed index (`Packages`, `Packages.gz`).
`ipkgs/assets/icons/` + `.../screenshots/` are referenced by `Source.Icon`/`Screenshots` URLs.
`staging/` holds packages pulled from the live feed but kept for later (see below).

> The original upstream CLAUDE.md described a `package.sh` + `IpkgFeedGenerator.jar` + `packages/`
> Makefile build system. **That tooling is NOT in this checkout and we do not use it.** We
> hand-build ipks and hand-maintain the index (below). Old tooling notes kept only for archaeology.

## How we actually build & maintain the feed (macOS)

We build ipks and regenerate the index with `ar`/`tar`/Python. Reusable patterns:

**Build an ipk** (debian-binary + control.tar.gz + data.tar.gz):
```bash
printf '2.0\n' > debian-binary
export COPYFILE_DISABLE=1   # no macOS ._ files
( cd control && tar --uid 0 --gid 0 --uname root --gname root -czf ../control.tar.gz . )
( cd data    && tar --uid 0 --gid 0 --uname root --gname root -czf ../data.tar.gz . )
ar rc out.ipk debian-binary control.tar.gz data.tar.gz   # this member order
```
- macOS BSD `ar`/`tar` are what we have (no gnu). bsdtar rejects `--no-mac-metadata`/`--no-xattrs`
  on older macOS — rely on `COPYFILE_DISABLE=1`. webOS opkg accepts BSD-ar output (proven).
- **Offline-root gotcha:** the on-device installer extracts `data.tar.gz` under offline root
  `/media/cryptofs/apps`. So `data.tar.gz` must be rooted at `./usr/palm/applications/<id>/...`
  (NOT include `media/cryptofs/apps/`), else it doubles and postinst can't find its payload.
- **control.tar.gz** contains `./control` (+ `./postinst`/`./prerm` at mode 0755 for patch pkgs).
  postinst/prerm self-run as root via ipkg/Preware.
- **BSD long ar names:** members >16 chars (e.g. Preware's `pmPostInstall.script`) are encoded
  `#1/N` by BSD ar. Any ar parser must handle `#1/N` (BSD), `/N`+`//` table (GNU), and plain names.
  (See the Python `ar_read()` we reuse in the index scripts in this repo's git history.)

**Regenerate the index:** the index is the authority Preware reads. We KEEP existing stanzas
verbatim and only add/replace the one we touch (the original nizovn stanzas are hand-curated and
differ from their ipk controls — do NOT blanket-regenerate from controls). Stanza field order:
`Package, Version, Depends, [Suggests], Section, Architecture, MD5Sum, Size, Filename,
Description, Maintainer, Source`. Then `Packages.gz = gzip(Packages, mtime=0)`; always
`cmp <(gunzip -c Packages.gz) Packages`.

**Preware metadata lives in the `Source:` field** as a JSON blob (Preware ignores std opkg fields
for display). Keys that matter: `Type` (`Application`/`OS Application`/`Linux Application`/
`Patch`/`Theme`), `Feed` (**Preware groups by this string, NOT the ipkg feed name** — must be
`"WOSA Modernize"` or the package hides under another group), `Category`, `Title`,
`FullDescription` (HTML, `<br>`), `Icon`/`Screenshots` (hosted URLs), `MinWebOSVersion`/
`MaxWebOSVersion`, `DeviceCompatibility` (e.g. `["TouchPad","Touchpad Go"]`), `LastUpdated`
(unix secs; absence → "Unknown" date header in Preware), `PostInstallFlags`
(`RestartLuna`/`RestartDevice`). Preware parses `Source` with `.replace(/\\'/g,"'")` then
`JSON.parse`, so `\'` is allowed; validate JSON in Python with the same replace.

**⚠️ Wrong-device protection — how Preware actually gates (verified in `~/Projects/preware` source):**
A user bricked a **webOS 2.2.4 phone** by installing the TouchPad TLS patches, because the
`tls-updates` meta was gated but **every member package was wide open**. The gate is real and free —
use it on anything device-specific.
- ⚠️ **`Min` is hard; `Max` and `DeviceCompatibility` are SOFT.** An earlier version of this file said
  "Min/Max → hard filter, no user override" — **that is wrong for `Max`** in the Preware 1.9.17 source
  we ship, and the difference decides feed-layout questions (see the Atlas two-stanza trick below), so
  re-read `loadPackage` rather than trusting a summary:
  - **`MinWebOSVersion` → hard, no override.** `models/packages.js:464-469`, `if (versionNewer(platformVersion, minWebOSVersion)) return;` — NOT gated on any pref. Only applies
    if `platformVersion` matches `/^[0-9:.-]+$/` (true on real devices).
  - **`MaxWebOSVersion` → soft.** `packages.js:473` calls `newPkg.versionIncompatible()` but the whole
    test is behind `if (!prefs.get().ignoreDevices ...)`. Same toggle as the device list.
  - **`DeviceCompatibility` → soft.** `packages.js:479`, also behind `!prefs.get().ignoreDevices`.
  - With the toggle on, the package returns and install shows only a click-through "Incompatible
    Device" warning (`pkg-view-assistant.js:416`). So **`Min` is the only real lock**; `Max` and the
    device list are deterrents. Set all three, but never rely on `Max` alone to keep a package off a
    device that must never see it.
  - Both bounds **inclusive**: `versionNewer(one, two)` returns true iff `one < two` (false on equal),
    and the call sites are `versionNewer(platform, min)` / `versionNewer(max, platform)`, so
    `Min == Max == "3.0.5"` admits exactly 3.0.5.
- **Absent fields = wide open** (defaults `minWebOSVersion '1.0.0'`, `maxWebOSVersion '99.9.9'`,
  `package.js:52-53`). This is the trap — silence is not a safe default.
- The filter also applies to the **installed** list (`getStatusFile` → `parsePackages` → `loadPackage`),
  so gating retroactively **hides an already-installed wrong-device package from Preware** — victims
  need WOSQI/novacom to remove it. (Moot for a real brick: no UI, so no Preware anyway.)
- **Gating is index-only and needs NO version bump** — `Source` is re-read on Update Feeds; only ipk
  contents/deps need a bump. Cheapest fix in the repo.
- **WOSQI and direct `ipkg install` ignore all of this** (no feed metadata). The only cover there is a
  version guard at the TOP of `postinst`, before any launcher edit — the payload alone is inert, the
  wiring is what breaks things. Not yet implemented; `mojomail-imap-tagfix`'s md5-guard is the
  in-repo precedent for the idea.
- Precedent for the two-sided fence: `com.palm.app.findapps` (Min 2.2.4 / **Max 2.9.9**) vs
  `com.palm.app.enyo-findapps` (Min 3.0.0). Give the 2.x line `Max 2.9.9` so the mistake can't run
  the other direction.
- **Use `Max 3.9.9`, NOT `3.0.9`.** Community convention (from 122 curated stanzas in the old
  `ipkg.preware.net/webos-internals` feed) is to fence the top of the major line: `1.9.9` / `2.9.9` /
  `3.9.9`. A `3.0.9` max would **hide the package from any device reporting 3.1.0** — exactly what
  `staging/org.webosce.luna-update` ("webOS CE 3.1.0") sets the version string to. All TouchPad-gated
  stanzas here were moved 3.0.9 → 3.9.9 for this reason. **One deliberate exception:**
  `com.palm.app.backup` is pinned `Min 3.0.5` / `Max 3.0.5` because it swaps a ROM app and is only
  safe on the build it was tested against — see its package bullet.

**Hardware (not OS-version) detection — two layers, both verified:**
- **Feed level = `DeviceCompatibility`**, matched as an **exact string** against
  `Mojo.Environment.DeviceInfo.modelNameAscii` (`packages.js:466`, `devices.include(...)`). The only
  values used in the wild (mined from the old webos-internals feed) are:
  **`Pre`, `Pre2`, `Pre3`, `Pixi`, `Veer`, `TouchPad`** — note `Pre2`/`Pre3` have **no space**.
  Preware's own code corroborates (`'Veer'`, `'Pixi'`, `'Pre2'`, and `indexOf('TouchPad') == 0`), and
  **`Pre3` + `TouchPad` are now confirmed on hardware** — packages appeared and installed on a real
  Pre 3 and TouchPad, and a wrong string here makes a package silently invisible in Preware.
  `Veer` / `Pre2` are still unconfirmed on hardware.
  - Preware **rewrites** `modelNameAscii` to `"Pre2"` when the machine name is `roadrunner`
    (`pkg-load-assistant.js:138`) — a Pre2 does not report `"Pre2"` natively. The fixup happens
    inside Preware, which is where the filter runs, so `"Pre2"` is the right stanza value; just don't
    assume it's an OS-level truth.
  - ⚠️ **`"Touchpad Go"` (lowercase p), used in ~12 stanzas here, matches nothing** — the compare is
    exact, Preware tests `indexOf('TouchPad') == 0`, and the string appears nowhere in the old feed.
    It's inert rather than harmful (real TouchPads still match on `"TouchPad"`). Unverified guess for
    a real Go/opal is `"TouchPad Go"`; needs one to test before "fixing" it.
- **postinst level = `/etc/prefs/properties/machineName`** — a plain file, `cat`-able as root. This is
  exactly what Preware's own ipkgservice does for `getMachineName`
  (`~/Projects/preware/source/src/luna_methods.c:561`: `/bin/cat /etc/prefs/properties/machineName`).
  This is the **hard** gate and the only one that covers WOSQI / direct `ipkg install`.
  Board names: TouchPad = `tenderloin`/`topaz`, TouchPad Go = `shortloin`/`opal`, Pre3 = `mantaray`,
  Veer = `broadway`, Pre2 = `roadrunner`. `roadrunner` is confirmed from Preware's source, and
  **`mantaray` is confirmed by a working install on a real Pre 3** (though that does not distinguish
  which source fired — machineName or the palm-build-info fallback). Still worth running
  `cat /etc/prefs/properties/machineName` on a Veer and a Pre 2 to record their values.
- **CPU/`Architecture` cannot separate these devices** — all four families are `armv7` (Pre2
  OMAP3630/Cortex-A8; Pre3 + Veer MSM8x55, same CPU as each other; all TouchPads APQ8060), which is
  why every upstream ipk is `_armv7`. The old feed's `all/armv6/armv7/i686` split separates the
  original Pre/Pixi era, not this one. Stock OS versions: Pre3 2.2.4, Veer 2.2.0 (upgradable to
  2.2.4), Pre2 2.1.0 (community 2.2.4) — so a `Min 2.2.4` gate would wrongly exclude un-upgraded
  Veers and Pre2s. For the phone line, keep the version fence loose (2.x) and separate by hardware.

**TouchPad-only vs device-common in the TLS chain** (per upstream
`github.com/webOSArchive/OpenSSL-legacyWebOS/tree/main/ipks` — **moved from `codepoet80/` to the
`webOSArchive` org**, same content, `~/Projects/OpenSSL-legacyWebOS` already points at the new
remote — which splits ipks into `topaz/` TouchPad, `mantaray/` Pre3, `roadrunner/` Pre2,
`broadway/` Veer, plus `phone/` for the merged multi-board builds):
- **Device-specific, built per device → gated 3.0.0–3.0.9 + DeviceCompatibility + `(TouchPad)` title
  suffix + a bold do-not-install-on-a-phone lead in FullDescription:** `browser-tls13`,
  `luna-tls13`, `downloadmgr-tls13`, `mojomail-imap-tagfix` (+ the `tls-updates` meta, already gated;
  title retitled for when the phone bundle lands).
- **Common to all devices → deliberately left ungated** so the 2.2.4 line reuses them as-is:
  `curl-tls13`, `mail-tls13`, `ntpdate-sync`, `com.palm.rootcertsupdate`.
- Those three commons all `Depends: org.webosinternals.browser-tls13`, which is now 3.0.x-gated —
  so until the phone builds land in the feed a phone user picking `mail-tls13` hits an unresolvable
  dep (a safe failure). It resolves itself once the phone `browser-tls13` builds are added.
- **When adding the phone builds:** upstream reuses the **same package names** per device. Same-name
  stanzas separated only by `DeviceCompatibility` do work on the happy path (each device's
  `loadPackage` drops the others), BUT — Min/Max can't tell Pre3 from Veer (all 2.2.4), so the split
  would rest entirely on the *bypassable* soft filter, and `package.js:452-465` **merges** the device
  lists of same-name stanzas that both survive. Prefer **distinct package names per device**.

**Editing descriptions / metadata:** safe to hand-edit `ipkgs/Packages` directly, then regenerate
`Packages.gz` only (don't re-derive). If you also want the ipk's own control to match (so it's
self-consistent), pull the edited `Source`/`Description` back into the build control and rebuild
the ipk (changes md5 → re-stanza). Editing only the index keeps ipk md5s unchanged → smaller deploy.

**Sorting a package to the TOP of a Preware list — a leading space in `Source.Title`.** Verified in
source and then **on device** (2026-08-24): Preware's default list sort is a raw
`a.title.toLowerCase()` string compare with **no trim** (`models/packages.js:789`), and `title` is
assigned straight from `Source.Title` untrimmed (`models/package.js:233`). A leading space (0x20)
therefore sorts ahead of every letter and digit, which is how
`" Synergy Revival Roll-up (Touchpad)"` sits above its 33 connectors when you search "Synergy".
Search is unaffected — it is a substring `include()` on the lowercased title
(`pkg-list-assistant.js:510`) — and the space does not render in the list row. ⚠️ **Do not "tidy" that
leading space away**; it is load-bearing, and it is the only lever we have over list order, since
Preware exposes no weight/priority field. Index-only, so it needs no version bump.

⚠️ **Never regenerate a stanza from the ipk's own control.** The index `Source` is the *curated*
copy and is routinely ahead of the control — titles, icons, Category/Feed retags, lead paragraphs.
Rebuilding a stanza from `control_of(ipk)` silently reverts every one of those (hit twice on
`synergy-revival` 2026-08-24: the roll-up lost its leading-space title, its icon and its opening line
when the meta was re-cut at 1.0.1). When a meta is rebuilt, take `Version` and `Depends` from the new
control and **keep the existing stanza's `Source` verbatim**, editing only what changed.

**Offering updates (Preware update mechanics — cost us a session):**
- **Preware compares the `Version` STRING only.** To ship an update you MUST bump the version number.
  Rebuilding an ipk *in place at the same version* (new md5, new content) will **never** show as an
  update — the device sees `1.0.1 == 1.0.1` and ignores it. This bit us on `tls-updates`: we changed
  its deps but kept it `1.0.1`; fix was to bump to `1.0.2`.
- **Replacing one package by name** (e.g. luna `1.1.0`→`1.1.1`, single stanza, higher version, old ipk
  removed) is all you need: Preware shows it as an *update* to anyone on the old version and a fresh
  *install* to anyone with neither. There's no feed way to offer an update "only if already installed".
- **Version floors** (`Depends: foo (>= 1.1.2)`) are the ONLY way to make a meta package drag an
  already-installed dep up: unversioned depends are "is any version installed?" → satisfied → no
  upgrade. So a meta that should force new deps needs BOTH the floor AND its own version bumped
  (see `tls-updates` 1.0.2). `(>= x)` syntax is standard opkg/Preware; validated in-repo but worth an
  on-device sanity check since nothing else here uses it.
- **"No update showing" debugging:** the feed is usually fine — verify first with
  `curl http://stacks.../Packages.gz | gunzip` that the version is really > installed AND that the
  `.gz` decompresses to `Packages` (a stale rsync'd `.gz` = Preware reads old versions). `stacks` is
  Cloudflare but serves `Packages.gz` as `DYNAMIC` (not edge-cached). If the server is right, the
  stale spot is the **device's** Preware feed cache → Update Feeds, else remove+re-add the feed, else
  reboot (webOS caches hard).

**Validate before commit:** 1:1 between `ipkgs/*.ipk` and index `Filename`s; every index `MD5Sum`/
`Size` matches the actual file; all `Source` JSON parses; full dependency closure (incl. version
floors) resolves.

## Deploy

`git push` (git) **and** sync `ipkgs/` to the server (e.g. `rsync -av --delete .../ipkgs/
<server>:.../feeds/modernize/ipkgs/`) — `--delete` removes stale ipks (renamed/held packages).
`staging/` is outside `ipkgs/` so it is never served. After deploy, Preware "Update Feeds".

## Driving the device over novacom (TouchPad)

- This machine's `novacom` eats dash-flags meant for the remote command (its own getopt). So
  `novacom run file://bin/grep -rl ...` fails (`-r`/`-l` consumed). Reliable pattern: write a
  script locally and pipe it: `novacom -t open tty:// < script.sh` with `exit` as the last line;
  filter the prompt with `grep -vE '^root@'`. `novacom run file://bin/cat /path` works only because
  it has no dash args. `novacom put file:///dev/path < local` for transfers.
- Install for testing: `ipkg -o /media/cryptofs/apps install /media/internal/x.ipk` — BUT this is
  **offline-root mode and DEFERS the postinst** ("not running ...postinst"). Run it manually:
  `sh /media/cryptofs/apps/usr/lib/ipkg/info/<pkg>.postinst`. Real Preware/WOSQI installs DO run it.
- VMware on the host keeps grabbing the USB device → flaky `novacom -l`. Disconnect it from the VM
  (connect to host). The device is `topaz-linux`; NDUIDs seen: `e516…` (healthy test device),
  `c931…` (the bricked one). No hostnames set, so they're hard to tell apart — check health first
  (`hostname`, version, `ps | grep LunaSysMgr`, installed pkgs) before patching.

## Package inventory (single feed = 69 packages / 90 stanzas — Atlas and each of the 20 Synergy
connectors have two; phones detailed below)

- **nizovn stack** (hand-curated stanzas): cacert, glibc, openssl, qt5* (`qt5qpaplugins` **1.0.4**,
  qt5 5.9.7-0, qt5sdk 1.0.2), qtwebbrowser, `qupzilla` (**2.3.1**), squid (kept for old phones,
  min 1.3.5), + vlcplayer, dbus. Dep chain: qupzilla→qt5sdk→qt5→qt5qpaplugins. Originally all
  **unversioned**, which meant bumping qupzilla did NOT drag qt5qpaplugins up (Preware stops at the
  first already-installed dep and won't recurse into satisfied qt5/qt5sdk to check qt5's own floor).
  **Now version-floored** (user asked, reversing the earlier leave-the-tree-alone stance):
  `qupzilla → qt5sdk (>= 1.0.2), qt5qpaplugins (>= 1.0.4)` — the qt5qpaplugins floor is a **direct**
  edge on qupzilla (the reliable trick; the transitive `qt5 → qt5qpaplugins (>= 1.0.4)` floor alone
  won't fire on existing installs); plus `qt5sdk → qt5 (>= 5.9.7-0)`. All **index-only** edits.
  qupzilla/qt5qpaplugins ipks are stock-packaged/GNU-ar; we curate the index Source (retag
  Feed/Category, host Icon) and keep the ipk as-delivered.
- **TLS 1.2/1.3 chain** (`org.webosinternals.*`, armv7): `browser-tls13` (**1.1.2**)→`com.palm.rootcertsupdate`;
  `curl-tls13` (1.0.1), `luna-tls13` (**1.1.4**), `mail-tls13` (**1.3.2**), `downloadmgr-tls13`
  (**1.0.0**) → browser-tls13; `mojomail-imap-tagfix` → mail-tls13; `ntpdate-sync`. (These came
  with `Feed:"WebOS Internals"` in
  their control — we retagged the index `Source` to `WOSA Modernize`/`Modernize` so they show in the
  modernize group; icon `openssl_icon.png`.) **These incoming ipks are GNU-ar** (member names end
  `/`, `//` longname table) — macOS BSD `ar x` chokes ("File exists"); extract them with the Python
  ar parser (`ar_members`, handles `#1/N` + `//` + plain) we reuse for the index scripts.
  - **`luna-tls13` history:** 1.0.0 (launcher edit only) → 1.1.0 (**faulty:** HTML5 media wedged
    after one track) → **1.1.1** adds a `/usr/bin/media-pipeline` wrapper that keeps the ssl11 stack
    out of the media worker so streaming/local media play → **1.1.2** adds a `/usr/sbin/setcpushares-pdk`
    wrapper that keeps the ssl11 launcher env (LD_BIND_NOW, compat-shim preload) out of PDK app
    launches, so PDK apps (QupZilla, nizovn Qt5) launch normally, incl. under LunaCE → 1.1.3 →
    **1.1.4** (current) — **cosmetic only on the TouchPad.** Verified by unpack-diff: the three wrap
    binaries and every other payload file are byte-identical, `appinfo.json` differs only in the version
    string, and postinst/prerm differ only in four reworded `echo` messages (dropping "(and to activate
    the media fix)" and naming Preware/WOSQI in the setcpushares-task line). It exists so both families
    sit at 1.1.4; the functional work in 1.1.4 is the **phone** build's two new env-scrub wrappers.
    This is the one place we deliberately departed from the mojomail/ntpdate precedent of NOT bumping a
    no-op — the cost is a 677KB re-download plus the meta's `RestartDevice` reboot for every TouchPad
    user, in exchange for version parity across the two families. Its FullDescription is synced in BOTH the index stanza and the ipk's own control
    (incoming ipk ships `Feed:"WebOS Internals"`/`Category:"System"`; we retag those in the index
    Source to `WOSA Modernize`/`Modernize` + add icon/LastUpdated — ipk kept as-delivered, index curated).
  - **`mail-tls13` history:** 1.3.1 → **1.3.2** fixes Gmail (and any ECDSA-leaf-cert) IMAP/POP
    sign-in that previously failed with a false "certificate is not trusted" (error 4010) — the stock
    libpalmsocket mis-verifies ECDSA certs, so the imap/pop/smtp launchers now pin TLS 1.2 + an RSA
    server cert (full validation preserved; Gmail needs a Google App Password for IMAP). Same retag
    pattern; the v1.3.2 note is folded into the curated index FullDescription.
  - **Upstream re-sync (`mojomail-imap-tagfix`, `ntpdate-sync`):** both were re-pulled from upstream
    at the **same version** (1.0.0 / 2.0.1). Byte-diffing proved there is **no functional change for
    the TouchPad** — every data file is identical, ntpdate's postinst/prerm are identical, and
    mojomail's postinst only hoists the hardcoded `seek=991784` into an `IMAP_OFF=` variable (a
    refactor so each board can carry its own patch offset). The real change is in the **controls**:
    upstream dropped two bogus `Depends` — `mojomail-imap-tagfix → mail-tls13` and
    `ntpdate-sync → browser-tls13` — neither of which was ever needed (both are standalone). Those
    drops are now reflected in our index too. **Deliberately NOT version-bumped:** a bump would push
    a no-op re-download to every existing user; at the same version, new installs get the new file and
    existing installs are left alone (their installed control keeps the harmless old dep). This is the
    one case where CLAUDE.md's "same version never shows as an update" gotcha is the *desired* behavior.
  - ⚠️ **`curl-tls13` is NOT re-synced and upstream's differs.** Upstream shipped a genuinely different
    build at the same version `1.0.1`: the bundled OpenSSL binaries differ (`libcrypto.so.1.1` ours
    2720826B vs upstream 2739092B; `libssl.so.1.1` 576868B vs 566844B), plus the same `Depends:` drop.
    Left alone on purpose — this package replaces `/usr/bin/curl`, which Synergy depends on, so
    swapping it is a tested change, not a metadata one. `mail-tls13` by contrast is **byte-identical**
    to upstream.
  - **`downloadmgr-tls13` 1.0.0 → 1.1.0:** RPATHs `/usr/bin/LunaDownloadMgr`
    (com.palm.downloadmanager) onto the ssl11 curl 7.61.1 (OpenSSL 1.1.1w) + a baked CA bundle, so
    background downloads/uploads negotiate TLS 1.2/1.3. `Depends: browser-tls13` (provides
    `/usr/lib/ssl11`); **remove BEFORE browser-tls13**. Incoming ipk ships
    `Feed:"WebOS Internals"`/`Category:"System"` — retagged in the index Source to
    `WOSA Modernize`/`Modernize` + icon `openssl_icon.png`/LastUpdated (ipk kept as-delivered).
    No reboot needed.
    - **1.1.0 is a real bug-fix and 1.0.0 should be considered broken.** 1.0.0's claim of "no binary
      code patch" turned out to be the bug: the daemon tears down and recreates its curl multi handle
      every time the transfer list empties, which the stock libcurl 7.21.7 survives but which SIGSEGVs
      in `curl_multi_remove_handle` on any modern libcurl. 1.1.0 adds a **one-byte** code patch
      disabling that path — 28 crashes per 60 download cycles before, 0 after, with no change in
      throughput, memory or fds. Payload is otherwise identical (same 541,910B binary, 1 byte apart).
    - **It also repairs a poisoned uninstall restore point**, and the reasoning generalises to every
      binary-swap package here: 1.0.0 guarded the "is this already ours?" test with an **md5 of the
      current build**, so a *previous* version's RPATH'd binary was mistaken for stock and saved as
      `/var/luna/LunaDownloadMgr.tls13-orig`. prerm would then restore that RPATH'd binary AND delete
      `/usr/lib/ssl11dl` — leaving the daemon unable to load its libcurl at all, i.e. downloads dead.
      1.1.0 identifies our builds by **content, not md5** (`grep -q ssl11dl` — every one of our builds
      carries `DT_RPATH /usr/lib/ssl11dl:/usr/lib/ssl11`, no stock one mentions it), which covers every
      earlier version and every board with no build-time bookkeeping. postinst discards a poisoned
      restore point on sight; prerm refuses to restore one. Note **no backup is safer than a wrong
      one**: with no backup, prerm correctly keeps ssl11dl in place.
    - **`downloadmgr-tls13-phone` stays at 1.0.0 and that is correct** — per the user, the
      `curl_multi_remove_handle` crash **does not occur on the phones**, so there is nothing for a
      `phone` rebuild to fix and `tls-updates-phone` needs no floor. 1.1.0 is a topaz-only release.
- **App Catalog:** `com.palm.app.findapps` (phones, Min 2.2.4/Max 2.9.9, icon hp-appcatalog),
  `com.palm.app.enyo-findapps` (TouchPad, Min 3.0.0). These were stock Palm-packaged ipks with no
  `Source` — we injected one.
- **`org.webosarchive.help-redirect`** (built + verified this session): patches `com.palm.app.help`
  `UrlManager.js` `helpUrl` (drives all content) + `HelpApp.js` palm.com domain check + device.do
  → `http://help.webosarchive.org`. Backs up `*.webosce-orig`, restores on removal, RestartLuna.
- **`org.webosarchive.tls-updates`** (`" TLS 1.3 Updates (TouchPad)"` — ⚠️ deliberate leading
  space, see the sorting note above; **1.0.19**) — **meta** package
  (README-only payload; it gained a `postinst` in 1.0.18, see below)
  package. Depends: rootcerts, browser-tls13, ntpdate-sync, curl/luna/mail-tls13,
  **downloadmgr-tls13 (>= 1.1.0)**, mojomail-imap-tagfix, help-redirect, enyo-findapps,
  **usbsettings (>= 1.1.9)**, **btgamepad (>= 1.1.0)**, **`com.palm.app.accounts (>= 3.1.1)`**
  (**1.0.18** — replaced `org.webosarchive.accountsapp`, which was a member from 1.0.13 and is now
  retired; see the Synergy Revival section),
  **`com.palm.app.backup (>= 3.1.1)`** (added unversioned in 1.0.15; floored in **1.0.19** to drag
  existing installs onto the restore fix). `ntpdate-sync` sits **after** browser-tls13
  and **before** luna — added because apps break when the clock is wrong (TLS cert validity windows
  fail); left unversioned (new dep, nothing to drag up). ⚠️ The old rationale "it's browser's own dep,
  so browser installs first regardless" is **no longer true** — upstream dropped ntpdate-sync's
  `Depends: browser-tls13` (it never needed it), so its position in this list is now the *only* thing
  ordering it. Harmless either way: it just drops in an upstart job and needs no ssl11. `downloadmgr-tls13` is ordered **after** luna-tls13 (per request; it also
  hard-depends browser-tls13 so browser installs first regardless); unversioned (new dep).
  Carries **version floors** on the packages that get revved: `browser-tls13 (>= 1.1.2)`,
  **`luna-tls13 (>= 1.1.4)`**, `mail-tls13 (>= 1.3.2)`, **`usbsettings (>= 1.1.9)`**,
  `btgamepad (>= 1.1.0)`, `downloadmgr-tls13 (>= 1.1.0)` (rest unversioned).
  Bumping a floor requires bumping tls-updates' own version too (1.0.6→1.0.7 for usbsettings,
  1.0.7→1.0.8 for btgamepad, 1.0.8→1.0.9 to drag usbsettings to 1.0.8, 1.0.9→1.0.10 to drag
  usbsettings to 1.1.0, **1.0.13→1.0.14 to drag downloadmgr to 1.1.0**) AND rebuilding
  the ipk so its control Depends match the index — else on-device opkg won't pull the new deps.
  Adding a plain (unversioned) new member is the same story minus the floor: **1.0.12→1.0.13**
  added `org.webosarchive.accountsapp` to Depends and still required the version bump + control
  rebuild, since Preware only offers an update when the Version string itself changes; **1.0.15→1.0.16**
  moved the luna floor to `>= 1.1.4`; **1.0.14→1.0.15**
  added `com.palm.app.backup` the same way — and 1.0.14 was *already on the server* (verified by
  `curl`-ing the live `Packages.gz` before touching anything, rather than assuming), so re-cutting it
  in place was never an option.
  (The 1.0.14 rebuild also brought the ipk control's
  `MaxWebOSVersion` back in line with the index at `3.0.9` — the control had drifted to `3.9.9`.
  Cosmetic only: Preware reads the index, not the ipk's control.)
  **Rebuild recipe that works:** reuse the old ipk's `debian-binary` and `data.tar.gz` members
  **verbatim** (the payload is just a README and must not churn), rewrite only `./control`, repack
  `control.tar.gz`, then `ar rc` in member order. Verify afterwards by unpack-diffing old vs new —
  the ONLY difference should be `C:control`.
  Excludes QupZilla, Atlas and LunaCE. **Several members are deliberately not TLS fixes** —
  `com.webosarchive.usbsettings`, `org.webosarchive.btgamepad`, `org.webosarchive.accountsapp` and
  `com.palm.app.backup`, all added by community request as
  "part of making the device more modern". They are the wired and wireless halves of the same
  capability (USB Settings covers a DualShock 4 over USB via its high-power toggle; btgamepad pairs
  classic Bluetooth HID pads), so shipping only one was the odd outcome. Both are TouchPad-only,
  dependency-free and order-independent, so they sit at the END of the Depends list and their position
  is cosmetic. Still deliberately OUTSIDE the roll-up: `atlas-default-browser` / `open-in-atlas` (they
  are mutually exclusive with each other and Atlas is 103MB) and `vlcplayer` — the meta is not
  "everything in the feed".
  `Type: OS Application`,
  no icon, **webOS 3.0.X only** (Min 3.0.0/Max 3.0.9, TouchPad/Touchpad Go), RestartDevice. This is
  the recommended one-tap install.
- **`org.webosports.app.atlas`** ("Atlas", **0.9.12**, WPE WebKit 2.52 browser, **103MB**, arch `all`)
  — the modern browser, shipped as **TWO index stanzas over ONE ipk** (see below). **We repackage the
  upstream WebOS Ports ipk on every bump**, rebuilding **only `control.tar.gz`** and keeping every
  other ar member byte-identical. What we change in the control: add the Preware `Source` block (Feed
  "WOSA Modernize", Category Browser, icon `atlas.png`). That is *all* we change.
  - ⚠️⚠️ **NEVER add `Depends:` to Atlas's control.** Upstream deliberately ships **no `Depends` at
    all**, and that is load-bearing, not an oversight — see the two-stanza section. A previous pass
    "helpfully" re-added `Depends: org.webosarchive.tls-updates` to the control and broke the design.
    Preware installs by **local file** (`IPKGService.install(cb, filename, location)` →
    `luna_methods.c:1692`, `ipkg -o /media/cryptofs/apps -force-overwrite install <path>`), so ipkg
    reads **the ipk's own control** and would resolve *its* Depends from the feed — dragging the whole
    TLS stack onto a 3.1.0 device that already has it. The dependency belongs in the **index only**,
    where it can be gated per OS version.
  - ⚠️ Do not put `MinWebOSVersion`/`MaxWebOSVersion` in the control either. The same filters run over
    the **installed** list (`getStatusFile` → `parsePackages` → `loadPackage`), so a Min/Max in the
    control can hide the already-installed app from Preware. Gating is the index's job.
  - The old step "remove the `killall LunaSysMgr` from postinst and prerm" (mid-batch restart lesson
    below) **is now handled upstream**: the build script rewrites a `PKG_TARGET=` line and the ipks
    delivered for this feed are built `PKG_TARGET=feed`, which skips the restart in postinst and never
    restarts from prerm. Still `grep -n killall` any new build to confirm.
  - ⚠️ Newer upstream ipks carry **five** ar members — the usual three plus `pmPostInstall.script` and
    `pmPreRemove.script` for the Palm installer. Keep them: repack as `ar rc out.ipk debian-binary
    control.tar.gz data.tar.gz pmPostInstall.script pmPreRemove.script`, then verify by unpack-diffing
    against the delivered ipk — **the only member that may differ is `control.tar.gz`**.
  - **Restart flags: `RestartDevice`** for `PostInstallFlags`/`PostUpdateFlags` (per the user,
    2026-08-23 — a Luna restart alone does not bring the new engine up); `PostRemoveFlags` stays
    `RestartLuna`. Supersedes the earlier all-`RestartLuna` setting. Set them in **both** stanzas and
    in the control (the control's copy is what the *installed* entry carries, so it is what governs
    the flag on removal).
  - **Version history in the feed:** 0.9.7 → 0.9.8 → (0.9.9 shipped and was **reverted** to 0.9.8 —
    a WPE engine regression upstream, packaging verified sound, see the `atlas-0-9-9-known-engine-bug`
    memory) → 0.9.10 → 0.9.11 → **0.9.12** (current). 0.9.10 dropped the self-triggering engine watchdog 0.9.8
    added (it could not tell a hung engine from an idle one and was reloading cards mid-session),
    replaced it with a user-triggered **Restart Browser Engine** menu item, and folded in 0.9.9's
    fresh-install GPU fix (`libEGL.so` bundled as a fallback); its WPE engine binary was
    byte-identical to 0.9.8/0.9.9, i.e. app + installer only.
  - **0.9.11 is the mirror image of 0.9.10: engine-only, app untouched.** It fixes the freeze
    upstream had been calling the "GPU wedge" at its source — not the GPU at all, but three defects in
    the engine's own plumbing (the frame channel closing a socket WebKit already owned; frame
    acknowledgements accumulating in a buffer nothing drained, blocking mid-frame after a few hundred
    frames; and a leftover debug hook sitting on the signal JavaScriptCore uses to coordinate GC,
    deadlocking the JS heap). Upstream's before/after on one device: froze after 8 page loads vs 64
    heavy page loads + a 12-minute animation soak clean. Restart Browser Engine stays as a backstop.
    **Verified by hashing every payload file in both ipks: exactly 3 of 1678 differ** —
    `appinfo.json` (version + changelog), `deviceroot/wpe-252/BrowserServer-atlas`, and
    `deviceroot/wpe-252/lib/libWPEBackend-atlas.so`. postinst/prerm and the two `pm*.script` members
    are byte-identical to 0.9.10, and the upstream build is again `PKG_TARGET=feed` (checked, not
    assumed). So the repackage was the standard one — control.tar.gz only — with no new surprises.
    Never run on hardware as 0.9.11 itself, but **its engine is now hardware-verified**: 0.9.12 ships it
    byte-for-byte and is deployed and working.
  - **0.9.12 is a one-line fix — and the FIRST build handed over was not.** DEPLOYED and confirmed
    working on hardware 2026-08-29. `appinfo.json` drops exactly one `mimeTypes` entry,
    `{"urlPattern": "^file:"}`, which Atlas had been using to claim every local `file:` link — so a
    photo, video or song opened from a file manager came to the browser instead of Photos, Video or
    Music. A non-scheme redirect handler outranks every mime and extension handler, so that one line
    beat every media app on the device. Payload diff vs 0.9.11: **1 of 1588 files**, engine binaries
    hash-identical; both `pm*.script` members byte-identical to 0.9.11.
    - ⚠️ **The first `0.9.12` ipk delivered (md5 `6b4478e6…`, 103,664,622B) was rejected — do not
      resurrect it.** It carried two problems the "just an appinfo.json change" handover did not
      mention, both found by the standard unpack-and-hash-every-file pass:
      1. **13 payload files changed, not 1.** Six added (`source/engine/{AtlasHost,ShellPalmSystem,
         ChromiumWebView,ChromiumOverlay,TabLayer}.js`, `css/chromium-tabs.css` — ~1,800 lines of an
         in-progress **LuneOS `browser_shell`/Chromium** second host backend) plus its wiring in
         `index.html`, `depends.js`, `Browser.js`, `AtlasEngineOverride.js`, and an unrelated
         find-bar match counter in `FindBar.js`/`browser.css`. Guarded behind `window.__atlasChromium`
         (capability detection, defaults to `wpe`) so *mostly* inert on a TouchPad — but the FindBar
         control is added to the component tree unconditionally.
      2. ⚠️ **A blocking `luna-send` to LunaSysMgr in BOTH postinst and prerm** —
         `luna-send -n 1 palm://com.palm.applicationManager/removeHandlersForAppId …`. Same deadlock
         class as Synergy issues #4/#6: `com.palm.applicationManager` is served **by LunaSysMgr**, the
         scripts run synchronously inside its own request handling (the prerm's own comment says so),
         and `-n 1` waits for a reply the blocked service can never send. `2>/dev/null` does not help.
         The replacement build dropped both calls; the scripts are now byte-identical to 0.9.11's.
    - **The trade-off that leaves, and the one-time note in the description.** Those calls existed to
      clear a real cache: webOS persists the *resolved* handler table to
      `/var/usr/palm/command-resource-handlers-active.json` and reloads it at startup in preference to
      rebuilding it, so an older Atlas's claim survives both the install and the restart — i.e. the
      `appinfo.json` fix alone lands for fresh installs but not for people upgrading from 0.9.11.
      Rather than ship the deadlock, **both Atlas stanzas carry a one-time instruction at the top of
      `FullDescription`** (user's call, 2026-08-29) telling upgraders to run the same call by hand and
      reboot:
      `luna-send -n 1 -f palm://com.palm.applicationManager/removeHandlersForAppId '{"appId":"org.webosports.app.atlas"}'`
      Index-only, so it needed no version bump. **Remove it when 0.9.13 ships** — it is a note for this
      upgrade, not a permanent part of the description.
    - **Lesson worth generalising: never `luna-send -n 1` to a service LunaSysMgr itself provides from
      a postinst/prerm** (`com.palm.applicationManager`, `com.palm.appinstaller` — both in its
      `allowedNames` in `/usr/share/ls2/roles/prv/com.palm.luna.json`). Background it:
      `( luna-send … & ) >/dev/null 2>&1`. `com.palm.configurator` is a *separate* daemon and is safe
      to call blocking, which is why the two `configurator/run` calls in Atlas's postinst are fine.
      And **grep every incoming ipk for it** — this is the third package family to ship the bug.
  - Its postinst lays the WPE engine (runs in place on cryptofs deviceroot), stages the device Adreno
    driver to the versioned GPU sonames (falling back to the bundled copy), symlinks
    `/var/atlas252 → deviceroot/wpe-252`, installs the NPAPI adapter
    (`/usr/lib/BrowserPlugins/BrowserAdapterAtlas.so`), upstart jobs (`/etc/event.d/atlas` +
    `atlas-sensord`), db8 kinds (`org.webosports.logins`/`autofill`), and ls2 roles
    (`/usr/share/ls2/roles/{prv,pub}/org.webosports.browserserver.json`). **Verified on device.**

### Atlas's two stanzas — a per-OS-version `Depends`, and why the ORDER matters

**The requirement:** on webOS 3.0.5 Atlas needs `org.webosarchive.tls-updates` (it links
`/usr/lib/ssl11` OpenSSL 1.1 for HTTPS). On **webOS CE 3.1.0 it must have NO depends** — that build
already carries the stack, and pulling `tls-updates` there is what made the install fail outright
(empty app folder, no launcher icon). One `Depends:` line cannot say both.

**The fix:** two stanzas, same `Package`/`Version`/`Architecture`, **same `Filename` and `MD5Sum`**,
differing only in gate + `Depends` + description:

| order | gate | `Depends` |
|---|---|---|
| **FIRST** | `Min 3.1.0` / `Max 3.9.9` | *(none)* |
| **SECOND** | `Min 3.0.5` / `Max 3.0.9` | `org.webosarchive.tls-updates` |

**The 3.1.0 stanza MUST come first.** Preware keeps the **first** stanza of a given name and discards
the rest: `loadPackage` (`packages.js:507`) finds the existing entry, calls `infoUpdate`, which for
"neither installed, same version" hits *Replace Type 6* and `return false` — and the
`this.infoLoadFromPkg(newPackage)` merge on that path is **commented out**, as is the depends-union
branch in `infoLoadFromPkg` (`package.js:523-541`). The loser is inert; `loadPackage`'s return value
is discarded by both call sites. So order alone decides. Combined with hard-`Min` / soft-`Max`:

| device | pref | what happens |
|---|---|---|
| 3.0.5 | either | 3.1.0 stanza killed by the **hard Min** filter → 3.0.x stanza wins → **dep present** ✓ |
| 3.1.0 | default | 3.0.x stanza killed by the Max filter → 3.1.0 stanza wins → **no dep** ✓ |
| 3.1.0 | *ignore devices* ON | both survive → **first wins** → 3.1.0 stanza → **no dep** ✓ |

Put the 3.0.x stanza first and that last row inverts, force-installing the TLS stack onto CE — the
exact breakage 0.9.10 fixed. **Only `Min` is un-bypassable, so the stanza that must win on the device
whose bound is a `Max` has to be first.**

**Why this is safe against the ipkg dedupe trap** (name+version+arch is ipkg's dedupe key and it takes
the "just overwrite the old one" branch): the trap is only dangerous when the colliding stanzas resolve
to **different files** — the per-board `browser-tls13` problem that forced the `*-phone` rename. Here
both stanzas name the **same `Filename` with the same `MD5Sum`**, so whichever ipkg keeps, it fetches
the identical ipk. And because that ipk's control carries **no `Depends`**, ipkg never consults the
feed entry for Atlas at all. Preware pre-resolves deps from its own model and installs the file
directly. **So the duplicate-stanza check must test "same name+version+arch AND different file",
not the name triple alone** — the check in this repo was updated accordingly.

- **`org.webosarchive.atlas-default-browser`** ("Make Atlas the default browser", **1.0.0**) — the
  new-style redirect+hide patch. `Depends: org.webosports.app.atlas`. **Self-contained (no AUSMT)**,
  same backup/restore pattern as help-redirect (the legacy Advanced-Browser/Isis patches this
  replaces were AUSMT — needing ausmt/patch/lsdiff, which this feed doesn't carry). postinst backs up
  `com.palm.app.browser/{index.html,appinfo.json}` to `*.webosce-orig`; prerm restores. **(1) Redirect:**
  an `awk` swaps ONLY the stock launch statement (`enyo.create({kind:"BrowserApp"})…`) for a redirect
  stub — the JS is **embedded in the postinst via a single-quoted heredoc** (NOT a payload file: a
  packaged payload would land under `/media/cryptofs/apps/…`, not `/usr/palm/…`, so referencing it by
  `/usr/palm` path is the bug that broke the first build). Guarded on the marker so it no-ops instead
  of clobbering. Fallback = "Load Stock Browser" only (renders `BrowserApp` in-place in the already
  running card — works even with the icon hidden). **(2) Hide:** sets `visible:false` **and**
  `removable:true` in appinfo.json. Caveat: `visible:false` governs a *fresh* app scan (never-placed
  apps stay hidden), but an **already-placed** grid icon persists in the saved launcher layout
  (`/var/luna/preferences/launcher3/page_ReorderablePage_APPS_{guid}` + the `quicklaunch_fixed.qlsave`
  dock) — a Luna restart re-saves it; even a rescan doesn't purge it. Same limitation the legacy
  stock-browser patches had; deemed acceptable (redirect still fires from the leftover icon). `Type:
  Patch`, Category Browser, RestartLuna.
- **`org.webosarchive.open-in-atlas`** ("Open in Atlas (stock browser menu)", **1.0.0**) — the
  **alternative** to atlas-default-browser for people who want to keep the stock browser. Same
  `Depends: org.webosports.app.atlas`, same self-contained embedded-heredoc + awk swap of the single
  `enyo.create({kind:"BrowserApp"})…` line, but it *prepends* an extension block and then **re-prints
  the original launch statement**, so the stock browser still runs — it only gains an App Menu item.
  Backup is `index.html.openinatlas-orig` (deliberately a different suffix from the redirect patch's
  `.webosce-orig`). The injected JS runs **before** instantiation and:
  - unshifts `{caption:"Open in Atlas", onclick:"openInAtlasClick"}` onto the AppMenu entry of
    **`enyo.BrowserApp.prototype.kindComponents`** — NOT `.components`: Enyo 1.0's
    `enyo.Component.subclass` renames a kind's `components` block to `kindComponents` and *deletes*
    `prototype.components` (framework/source/kernel/Component.js:440). Getting this wrong is a silent
    no-op. Because it happens pre-instantiation, Enyo builds/renders the item normally — no
    `createComponent`+`render()` on an unrendered popup.
  - appends its own `PalmService` component (`palm://com.palm.applicationManager/`, method `open`) so a
    missing Atlas produces a banner instead of the browser's generic "Cannot open MIME type" popup;
    `openInAtlasClick` calls `{id:"org.webosports.app.atlas", params:{target:url}}` — the same call
    stock uses in `openResourceWithApp`/`helpClick`. Atlas reads `p.target || p.url` in `rendered()`
    and, when a card already exists, navigates it from `applicationRelaunchHandler`.
  - **un-escapes the URL**: `Browser.pageTitleChanged()` stores the live URL through
    `enyo.string.escapeHtml` (`& < >` only), so `&amp;` would corrupt every query string handed over.
  - wraps `toggleAppMenuItems` to grey the item out unless the browser view is showing and a real
    (scheme-prefixed) URL is loaded — mirrors the stock Print item.
  Mutual exclusion is enforced at runtime, not by `Conflicts:` (Preware ignores it): postinst exits
  early if index.html contains `Redirecting to Atlas` or `index.html.webosce-orig` exists, and prerm
  refuses to restore over an index.html that no longer carries the `OPEN-IN-ATLAS` marker. `Type:
  Patch`, Category Browser, icon atlas.png, RestartLuna.

- **`org.webosarchive.btgamepad`** ("Bluetooth Gamepad Support", **1.1.0**, armv7) — delivered
  pre-built by the user in `add-to-feed/` together with a hand-written `Packages-stanza.txt`; added
  index-only (ipk kept as-delivered, md5/size in the stanza already matched). Interposes the stock
  unfinished Bluetooth HID path via `libpmbtgamepad.so` + a udev rule under
  `/usr/palm/applications/org.webosarchive.btgamepad/files/` (no binary patch); DS4 / classic BR/EDR
  pads pair under **Other** in Bluetooth settings. No `Depends`. `Type: Application`, Category
  Modernize, Min 3.0.0 / **Max 3.9.9**, RestartDevice. **Member of the TouchPad `tls-updates`
  roll-up** as `btgamepad (>= 1.1.0)`. The only thing we added to the delivered stanza was
  `"Icon": .../assets/icons/bt-gamepad.png` (the ipk's own control Source has no Icon — index is the
  curated copy, per the pattern above).

- **`com.webosarchive.usbsettings`** ("USB Settings", **1.1.9**, arch `all`, TouchPad only) — Enyo app
  + JS bridge service + a root upstart daemon (`usbctl-watchd`) controlling the USB port: **USB host
  mode (OTG)** for keyboards / game controllers / flash drives, **high-power devices** (e.g. a
  DualShock 4 over USB — enable before plugging in), and **USB storage** mount/unmount with live
  status and banner notifications on detect/mount/remove. Delivered as a palm-packaged ipk with a bare
  control (no `Source`, generic "This is a webOS application.") — we inject a curated `Source`, same
  as the findapps packages; the ipk is kept as-delivered. `Type: Application`, Category Modernize,
  Min 3.0.0 / **Max 3.9.9**, `["TouchPad","Touchpad Go"]`, icon `usbsettings-icon.png` + screenshot
  `usbsettings_screenshot.png`, License GPL. **`PostInstallFlags: RestartDevice`** — the postinst
  registers the JS service under `/var/palm/ls2/{roles,services}/{pub,prv}` and the LS2 hub only reads
  that map at startup, so the app cannot reach its service until a reboot (its UI has a "Helper not
  running… reboot" row for exactly that state). prerm removes the daemon, upstart job and registration
  and leaves mounted media alone.
  - **1.0.3 → 1.0.6 made it genuinely self-contained.** 1.0.3 required
    `/var/usr/bin/run-homebrew-js-service` from an external feed package; 1.0.6 ships its own
    `usbctl-jsservice` shim (a thin wrapper over the *stock* `/bin/node` +
    `/usr/palm/services/jsservicelauncher/bootstrap-node.js`), so every path it touches exists on a
    bare 3.0.5 device. Hence **no `Depends` at all** — and none is possible anyway, since the thing it
    used to need was never a package in this feed.
  - **1.0.6 → 1.0.7 fixes exactly one thing:** `"removable": false` in appinfo.json, which hides the
    launcher's Delete button. A launcher Delete skips `prerm`, which would leave the `usbctl-watchd`
    daemon running and the LS2 registration in place — so it must be uninstalled through
    Preware/WOSQI. Everything else in the diff is version strings (verified: 22 files, 3 changed,
    all three just version bumps apart from that one line). Note `removable:false` governs the
    *launcher* UI only; Preware/ipkg removal still runs prerm normally — worth confirming on device.
  - **1.0.7 → 1.0.8 is a one-line `usbctl-watchd.sh` fix** (verified by unpack-diff: 4 files differ,
    3 of them version strings in appinfo/packageinfo/control; postinst and prerm **unchanged**).
    `power_apply()` used to force config 1 only on devices whose `bConfigurationValue` reads `"0"`
    (a DS4, which asks for 500mA against the root hub's ~390mA budget). Some pads — a DragonRise was
    the report — get rejected outright and read **EMPTY** instead, so they were skipped. Now it
    matches `"0"` OR empty, and still skips anything already configured (`"1"`+). Shipping it needed
    the meta bumped **1.0.8 → 1.0.9** with the floor moved to `usbsettings (>= 1.0.8)`.
  - **1.0.8 → 1.1.0 is a real feature bump**, delivered pre-built in `add-to-feed/` (verified by
    unpack-diff against 1.0.8 before adding to the feed): (1) `usbctl-watchd.sh` now **auto-arms the
    OTG controller** the first time the port is idle — writing `host` once to the OTG mode-select node
    is a persistent, one-shot flip; a device that had **never** been armed would instead hard-hang its
    USB port on the very first OTG cable insert until reboot, so this is a safety fix, not just
    convenience (guarded off while a PC/charger gadget session is active, so it can't kill novacom/
    charging mid-use). (2) Live **OTG state is now probed from `/sys/bus/usb/devices/usb1`** every
    status read instead of trusted from a saved flag — the controller flips modes on its own on cable/
    ID-pin events, so the flag could go stale. (3) A new **USB Reset** button cycles OTG
    peripheral→host to recover a device that enumerated but stopped delivering input (an
    interrupt-endpoint wedge from suspend/power churn); refused both in the shell script and the UI
    while storage is mounted, so it can't yank a drive mid-use. (4) A new **`usbdevmon`** helper binary
    is spawned only while the app panel is open (gated on a keepalive file the app refreshes, `send
    watch-on`) and writes a live connected-device list (with per-type icons: keyboard/mouse/gamepad/
    storage/generic) that `GetStatusAssistant.js` reads back if fresh (<6s old) — kept off otherwise so
    it never contends for input nodes while, e.g., a game is running. Shipping it needed the meta
    bumped **1.0.9 → 1.0.10** with the floor moved to `usbsettings (>= 1.1.0)`.
  - **1.1.0 → 1.1.9** covers three releases not written up here individually (see the meta's own
    changelog in its `Source` for 1.1.4 and 1.1.8: an idle-daemon fork storm that caused frame-rate
    stutter, and a device-monitor bug that opened controllers *exclusively* so a pad verified in this
    app was then dead in any game launched afterwards).
  - **1.1.8 → 1.1.9 changes exactly one thing:** the USB storage mountpoint moves from
    `/media/internal/usbdrive` to **`/media/usb`**. Verified by unpack-diff — 3 files differ, two of
    them version strings (`appinfo.json`, `packageinfo.json`), the third `usbctl-watchd.sh`; postinst
    and prerm are **byte-identical**. The bug it fixes is a good one to remember, because it will bite
    anything else that mounts under the media partition: **Internalz Pro's backend (FileMgr-Service)
    lists any path under `/media/internal` with mtools** (`mdir` against mtools.conf's
    `drive A: file="/dev/mapper/store-media"`) whenever the caller asks to hide hidden files — which
    is that app's default. mtools parses the internal partition's FAT directly and is therefore
    **blind to the kernel mount table**, so a stick mounted at a subdirectory of that volume listed as
    an **empty folder**, and writes into it silently landed on internal storage *underneath* the
    mountpoint. The check is a bare `startsWith("/media/internal")` with no trailing slash, so a name
    like `/media/internal-usb` would not have helped either. ⚠️ The catch the old code was right
    about: **`/media` is on the read-only rootfs**, so `mkdir -p /media/usb` only succeeds once rootfs
    has been opened for writing (which Preware / webOS Internals users have generally done). The
    script falls back to the old in-partition path when the mkdir fails, so a locked-down device still
    gets a working — if Internalz-blind — mount. Shipping it needed the meta bumped
    **1.0.16 → 1.0.17** with the floor moved to `usbsettings (>= 1.1.9)`.
  - **It is a member of the TouchPad `tls-updates` meta** (community request) even though it is not
    TLS-related — "part of making the device more modern". Added as
    `com.webosarchive.usbsettings (>= 1.1.9)` at the END of the Depends list; it is self-contained and
    order-independent, so position is cosmetic. Adding it required bumping the meta **1.0.6 → 1.0.7**
    AND rebuilding the meta ipk so its control `Depends` match the index — verified by an automated
    check that now compares every meta's control `Version`/`Depends` against its index stanza.
    ⚠️ **Re-cutting a meta in place at the same version bit us.** `tls-updates 1.0.7` was re-cut twice
    at the same version number (moving the usbsettings floor 1.0.6→1.0.7, then adding btgamepad) on the
    assumption it was still unpublished — but it had already been pushed to the server, because pushing
    is the only way to test on a device. Preware compares the Version STRING only, so every device that
    took an earlier 1.0.7 would have been stuck with it forever: same version, no update offered, new
    members silently missing. Fixed by bumping to **1.0.8**, which supersedes every 1.0.7 variant and
    whose floors (`usbsettings >= 1.0.7`, `btgamepad >= 1.1.0`) drag the members up too.
    **Rule: once anything might be on the server, never re-cut at the same version — always bump.**
    "It is not published yet" is an assumption to verify, not assume.)
  - Note the id is `com.webosarchive.*`, not the preferred `org.webosarchive.*` — it is baked into
    appinfo.json/serviceId, so renaming is not a feed-side edit.

- **`com.palm.app.backup`** ("Backup and Restore", **3.1.1**, arch `all`, TouchPad only) — a working
  local Backup/Restore, bringing back what died when Palm's cloud went dark in 2013. Backups live on
  the device in `/media/internal/webos-backups` as plain, content-addressed files you can copy off
  over USB; deliberately **not encrypted** (stock encrypted with a device key that survives neither a
  Doctor nor a move to another device — i.e. useless in exactly the two cases you'd want a backup).
  Covers accounts/contacts/calendar, settings, browser data, app data, third-party registered data,
  and the apps themselves (reinstalled on restore); media is opt-in per category.
  - **This is the corrected rebuild of the package that was held back.** The earlier attempt was
    `com.woce.backup` 1.0.0, staged and then pulled before release because the app id was wrong. The
    id is now `com.palm.app.backup` and the version is **3.1.0 on purpose**: it must out-rank the
    built-in `com.palm.app.backup` 2.0.0. The `fresh-ipks/` copy of the old build is gone.
  - ⚠️ **Version alone does NOT beat a ROM app**, which is the interesting finding here and cost the
    author a build. Installing to `/media/cryptofs` with a higher declared version, after a reboot,
    still left the launcher tile, `getAppInfo` and the running card resolving to
    `/usr/palm/applications/com.palm.app.backup` — the ROM copy simply wins. So the postinst
    **moves the ROM app aside** (remount rw → `cp -a` to `/var/luna/com.palm.app.backup.woce-orig` →
    `rm -rf`), same shape as the TLS packages' binary swaps, and prerm puts it back. Guards: only
    moves a copy that is **not ours** (checked by content — `grep "webOS Community Edition"` in
    appinfo.json, not by version, so it holds on a webOS CE image where the ROM copy *is* this app),
    only when `/etc/palm-build-info` reports the targeted `3.0.5`, and never overwrites an existing
    restore point.
  - **`PostInstallFlags`/`PostUpdateFlags` = `RestartDevice`** (per the user; the ROM-app swap needs
    it — this reverses the "RestartLuna is enough" note from the first pass, which predated the swap).
    `PostRemoveFlags` = `RestartLuna`. Its postinst contains **no `killall LunaSysMgr` and no reboot**,
    which is what makes it safe inside the meta's dependency batch (see the mid-batch restart lesson).
  - **Gated to exactly one OS version: `Min 3.0.5` / `Max 3.0.5`** (per the user). This is the only
    stanza in the feed pinned to a single version, and it deliberately departs from the `Max 3.9.9` /
    `Max 3.0.9` convention — the justification is the ROM-app swap: the postinst is only willing to
    move the built-in app aside on the build it was tested against, so there is no point offering the
    package anywhere else. It lines the feed gate up with the postinst's own `EXPECT_OSVER=3.0.5`.
    An exact pin is safe to express because **both bounds are inclusive**: `versionNewer` returns
    false on equal (`packages.js:920`), and the two call sites are `versionNewer(platform, min)` for
    the floor and `versionNewer(max, platform)` for the ceiling, so `min == max == "3.0.5"` admits
    exactly 3.0.5 and nothing else. No `DeviceCompatibility` needed — 3.0.5 is TouchPad-exclusive, and
    Min/Max is the *hard* filter with no user override, unlike the bypassable device list.
    ⚠️ The flip side of a hard exact gate: if a device reported anything but a bare `3.0.5`, the
    package would be invisible with **no way for the user to opt in**. Real TouchPads report `3.0.5`
    (corroborated by `tls-updates` itself being `Min 3.0.5` and verified on hardware), so this is
    sound — but it is the assumption the gate rests on.
  - ⚠️ **The exact pin opens a theoretical hole in the roll-up:** the meta is `Min 3.0.5` / `Max 3.0.9`,
    so on a device reporting 3.0.6–3.0.9 the meta would be visible while this member is not, and the
    one-tap install would fail on an unresolvable dep. **No such webOS build exists** (the 3.x line is
    3.0.0 / 3.0.2 / 3.0.4 / 3.0.5, then CE 3.1.0), so it is inert — but if the meta's `Max` is ever
    relaxed, or a device turns up reporting 3.0.6+, either pin the meta to `Max 3.0.5` too or widen
    this one. The per-device check below catches it if it ever becomes real.
  - Unlike the earlier build, this ipk's control **already carries a curated `Source`** (the palm-packager
    bare-control problem is gone). We still curate the index copy — the ipk's `Feed` is `"webOS Archive"`,
    which is **not** the string Preware groups by, so the index sets `Feed:"WOSA Modernize"` /
    `Category:"Modernize"` and overrides Section→`System`, Maintainer→`webOS Archive`. ipk kept
    as-delivered. Icon `backup-icon.png` extracted from `usr/palm/applications/com.palm.app.backup/`.
  - Root helper unchanged in shape from the first review: `/usr/bin/woce-backupd.js` +
    `/etc/event.d/woce-backupd` outside the app bundle (`/media/cryptofs` is `nosuid`), `stop` →
    `pkill` → `start` on upgrade (a bare `start` on a running job is a no-op). A non-root
    `palm-install` skips postinst entirely and the app says so in a "limited mode" banner — so
    Preware/WOSQI is the supported route. prerm restores `/etc/palm/mojodb.conf` from saved bytes and
    keeps `/media/internal/webos-backups`.
  - **Member of the TouchPad `tls-updates` roll-up**, at the END of Depends (self-contained and
    order-independent). Added unversioned in meta 1.0.15; **floored `(>= 3.1.1)` in meta 1.0.19**, since
    an unversioned depend is only "is any version installed?" and would never drag an existing 3.1.0
    install onto the restore fix.
  - **3.1.0 → 3.1.1 fixes a silently-partial restore, and 3.1.0 should be considered unsafe to restore
    from.** Payload diff: 5 files modified, 1 added. Three of the five are pure version strings
    (`appinfo.json`, `packageinfo.json`, and the `VERSION`/`SERVICE_BUILD` build stamps in
    `device/woce-backupd.js` and `assistants/service-assistant.js`); `postinst` and `prerm` are
    **byte-identical**, so the ROM-app swap and its guards are unchanged. The one real change is
    `util/backup-service.js` → `syncManifests()`.
    - The bug: manifest names are `NNNNNN-<nduId>` and the counter **restarts whenever the backup store
      is cleared**, while `/media/internal` survives a Doctor. So a stale local `000001-19Q` can name a
      completely different backup than the target's `000001-19Q`. The old code reconciled by **name
      only** — fetch what is missing, drop what the target lacks — which left any name present on both
      sides untouched, so the stale local copy shadowed the real one. Measured once upstream: the
      device read a 6-file manifest, restored 6 files out of a **115-package** backup, and reported
      success.
    - The fix re-fetches **every** manifest the target lists, and a fetch that fails deletes the local
      copy rather than trusting it. Upstream's comment explains why there is no cheaper content test:
      manifests carry no Etag (only the content-addressed `files/` store does), `get()` does not
      preserve mtime, sizes collide because manifests are lists of fixed-width checksums, and the
      `nduId` inside is identical between the colliding copies since they come from the same device.
    - Minor, not blocking: the "newest manifest" the function returns is still picked from the target's
      full list, so if the newest one is the one that failed to fetch, the name is returned while the
      local copy has been dropped. It logs `UNFETCHABLE:` when that happens.
  - ⚠️ **3.1.1 was RE-CUT AT THE SAME VERSION on 2026-08-29 — there are two different `3.1.1` ipks.**
    The first was published briefly, then **pulled off the server by the user before anyone downloaded
    it**, because it shipped a bug that filled `/media/internal`. The replacement keeps the version
    string `3.1.1` (user's call, reaffirmed after the version risk was raised). This is a deliberate
    exception to "never re-cut at the same version once anything is on the server", and it rests
    entirely on nobody having installed the first one — Preware compares the version STRING, so any
    device that *did* take the first 3.1.1 will never be offered the replacement.
    - shipped (current): md5 `b8f2a57cb32114c6600b64f3ec429890`, 177,346B, 54 payload files
    - pulled: md5 `31af1c8089b730f19beff05c3cc535ab`, 182,148B, 55 files (and our stray-stripped re-cut
      of it, `c9e2e2e04ae89441d5f693850ec00ef5`) — **do not resurrect either**
  - **What the replacement fixes: a failed backup used to leak its partial output.** One file changed,
    `handlers/backup.js` `handleError()` (the other two diffs are build stamps). A backup stores each
    file as it goes and writes the manifest **last**, so a run that dies partway leaves stored files
    that no manifest references — and only the success path purged. Every failure leaked at full size,
    which made the next run likelier to fail the same way. Measured upstream on a daily driver: two
    failed runs in one day left two unreferenced copies of a 295MB app archive, `/media/internal` hit
    100%, and every subsequent backup failed with ENOSPC. The failure path now also calls
    `backup.purge(target, 100000)`.
    - The `100000` is not a magic number, it is "sweep orphans but trim **no** manifests":
      `purge(target, keep)` computes `excess = max(0, manifests.length - keep)`, so a huge `keep` means
      nothing is trimmed and only unreferenced files are deleted. `list-backups.js` uses the identical
      idiom for the same reason. Checked and correct — a failed backup must not delete the user's older
      good backups.
    - Safe against the stale-cache hazard: `syncManifests()` runs early in `doBackup` (backup.js:709),
      well before anything is stored, and `getReferencedFiles(false)` counts **all** local manifests
      including other devices', so the sweep is conservative. A failure before the sync has stored
      nothing to sweep.
    - ⚠️ **Known weakness, reported not blocking:** the new `f.nest(purge…)` sits *after* a
      `var result = f.result;` in the same handler, and in Palm Foundations reading `.result` on a
      future carrying an exception throws — this codebase's own idiom is to test `f.exception` first
      (`list-backups.js:121`). `cleanup` is `fileUtil.rmFiles(stageDir, true)`, so **if stage cleanup
      itself fails the orphan sweep is skipped** — and that is exactly the ENOSPC case the fix exists
      for. Moving the `f.nest()` above the dead `var result` read closes it. The dead read predates
      this change, and error reporting is unaffected (`f.exception = err` is set in the next handler).
  - **The stray `CLAUDE.md` is gone from the build** — the delivered ipk is now 54 files and needs no
    stripping, so it is back to being kept **as-delivered**. (The previous build shipped the
    developer's Enyo-gotchas agent notes inside the app payload; that was stripped here by hand. If a
    future release brings it back, the TarInfo-preserving method is in this file's git history.)
    Note the incoming control's `Source.Changelog` still has **3.1.0** as its newest entry — the 3.1.1
    notes exist only in our curated index stanza.

- **`org.webosarchive.accountsapp`** — ⚠️ **RETIRED 2026-08-24, no longer in the feed.** Superseded by
  `com.palm.app.accounts` **3.1.1** (Synergy Revival Part 1), which is a *strict superset*: byte-identical
  files apart from `appinfo.json` version strings and `AccountManager.js`, plus 9 localization files.
  Its stanza and ipk were removed and `tls-updates` 1.0.18 depends on the new package instead. Both metas
  carry a postinst that defuses a still-installed copy — see "Retiring `accountsapp`" below. The rest of
  this bullet is kept for archaeology.
  Original description: "Accounts (Community Build)", **3.1.0**, arch `all`,
  delivered pre-built by the user as `org.webosarchive.accountsapp_3.1.0_all.ipk` — replaces the
  stock Accounts app (`com.palm.app.accounts`) with the webOS-ports/LG open-source Enyo 1.0 build,
  re-enabling System account management (sign in, name/email/password, public username, devices on
  the account, sign out from Settings > Accounts) against the **community account endpoint**. Needs
  the companion webOS Account app to actually host the account — without it the Accounts row simply
  stays greyed out, same as stock. Added index-only; ipk kept as-delivered (its own control has no
  `Source` block, generic `Description: Accounts (webOS Archive build)` — same pattern as
  `com.palm.app.findapps`/`usbsettings`: index carries the curated `Source`, ipk untouched).
  - **`Depends: org.webosinternals.browser-tls13, org.webosinternals.luna-tls13`** (both
    unversioned) — per the user's instruction. The app runs as an Enyo app under the LunaSysMgr
    WebKit host and talks to the community endpoint over HTTPS, which only works once luna-tls13
    (and its own dependency browser-tls13) are installed.
  - **postinst is TouchPad-only by its own hard guard**, independent of feed gating: it reads
    `/etc/palm-build-info` and refuses (`exit 1`) unless the version starts with `3.0` — so a webOS
    2.x phone *or* webOS CE 3.1.0 gets a clean refusal rather than a broken Accounts app, even under
    WOSQI/`ipkg install` where feed metadata is not consulted. Backs the stock app up to
    `/media/cryptofs/webosarchive-accounts/stock-app.tar` (skipped if a stock backup already exists
    or the app already in place is our own build — detected via a marker file,
    `source/palmID/UsernameDialog.js`, unique to this build) and restores it on `prerm`, reading the
    install path from a state marker rather than the (by-then-deleted) staged payload — the bug this
    avoids: a prerm that reads `dest.txt` from the staged payload finds nothing once postinst has
    cleaned it up, so it silently skips the restore. Guards against a second uninstall call
    (confirmed to happen under WOSQI) turning a completed restore into a missing app directory.
  - **Feed-level gate:** `Min 3.0.0` / `Max 3.9.9`, matching the same convention as
    `com.palm.app.enyo-findapps`/`btgamepad`/`usbsettings` (no `DeviceCompatibility` needed — the
    Min/Max hard filter alone is TouchPad-exclusive since no phone reports 3.x). This is
    deliberately looser than the postinst's own `3.0.x`-only check, which is fine: feed visibility
    just needs to keep phones out, the postinst is the second, stricter layer that also catches CE.
  - `PostInstallFlags`/`PostUpdateFlags`/`PostRemoveFlags` all `RestartLuna` (postinst/prerm both log
    "restart the UI to pick up the new app JS" rather than reloading it themselves — same
    single-end-restart pattern as every other patch in this feed, see the mid-batch restart lesson).
  - **Added as a new (unversioned) member of `org.webosarchive.tls-updates`**, bumping the meta
    **1.0.12 → 1.0.13** — see the TLS 1.2/1.3 chain bullet above.

## Synergy Revival (experimental — added 2026-08-24, branch `synergy-connectors`)

**What it is.** Herman van Hazendonk's revival of webOS account aggregation: modern IM services
(WhatsApp, Telegram, Signal, Teams, Discord, Google Chat, Facebook), cloud/photo storage (Dropbox,
Google Drive, OneDrive, Box, MEGA, pCloud, Koofr, kDrive, HiDrive, Yandex Disk, S3, Flickr) and
CalDAV/CardDAV, as real Synergy accounts under Settings > Accounts. **34 packages, ~71MB**, in two
parts plus a roll-up we build here:

- **Part 1 (13 pkgs, 24MB)** — required on 3.0.5, **built into webOS CE 3.1.0**. `com.palm.synergy.generic`
  (13MB: imlibpurpletransport + libpurple + a frozen `synergy-glibc` + cloudcore + the `com.palm.app.cloud-auth`
  OAuth app) plus whole-directory replacements of stock apps/frameworks/services: `com.palm.app.{accounts,
  contacts,messaging,phone}`, `com.palm.service.{accounts,contacts.linker}`, `com.palm.messaging.chatthreader`,
  `enyo-accounts`, `enyo-contactsui`, `messaging.library`, `contacts.plugin.messaging`, `luna-systemui`.
- **Part 2 (20 pkgs, 47MB)** — the connectors themselves, pick-and-choose, **both** device classes.
- **`org.webosarchive.synergy-revival` 1.0.0** — our hand-built meta (like `tls-updates`): pulls
  `tls-updates`, `atlas (>= 0.9.11)` and all 13 Part 1 packages, each floored at its shipped version.

**Sources:** `~/Downloads/synergy_revival_0.9.3_part1_v4/` and `..._part2_v3/`. ⚠️ For
`com.palm.app.accounts` use the **`~/Desktop/com.palm.app.accounts_3.1.1_all.ipk`** copy, not the one in
part1_v4 — the Downloads build is broken (its postinst is the old accountsapp script with
`REL=media/cryptofs/webosarchive-accounts-overwrite`, while the payload ships at
`core-apps-overwrite/com.palm.app.accounts/`, so it finds nothing: `exit 1` on a fresh device, or a silent
"already installed" no-op where accountsapp was present). The Desktop build uses the shared core-apps
postinst, which locates the payload by PKGID, and differs in content only in `AccountManager.js` (SYNERGY
ACCOUNTS becomes a real `RowGroup`). It is what CE 3.1.0 RC ships.

**Packaging facts (all ipks kept as-delivered):** every control is bare — **no `Depends`, no `Source`** —
so every dependency here is index-only (safe: Preware installs by local file and ipkg reads the ipk's own
control, the same reason Atlas must not carry `Depends`). Controls declare `Architecture: armv7` even
though filenames say `_all`; the stanzas follow the control. All 33 are 5-member ars (`pmPostInstall.script`
/ `pmPreRemove.script` alongside the usual three). **No script restarts LunaSysMgr or reboots** — they only
log "restart the UI" — so they are safe inside a Preware dependency batch (`generic` does restart bluetooth
and mediaserver, which is fine). `generic` declares `Replaces:`/`Conflicts: imlibpurpleservice`; Preware
ignores both, and nothing in this feed ships that package.

### Index layout — the Atlas two-stanza trick, reused per connector

| stanza | gate | `Depends` |
|---|---|---|
| Part 1 (13) + the meta | `Min 3.0.5` / `Max 3.0.9` | none / (meta: tls-updates, atlas, the 13) |
| connector — **FIRST** | `Min 3.1.0` / `Max 3.9.9` | `org.webosports.app.atlas (>= 0.9.11)` |
| connector — SECOND | `Min 3.0.5` / `Max 3.0.9` | `org.webosarchive.synergy-revival, org.webosports.app.atlas (>= 0.9.11)` |

Same reasoning as Atlas's own two stanzas (read that section first): Preware keeps the **first** stanza of a
name, `Min` is the only un-bypassable filter, so the stanza that must win where the *other* one is excluded
by a soft `Max` has to come first. Both stanzas name the same `Filename`+`MD5Sum`, so the ipkg dedupe trap
does not apply. **`Min 3.0.5`, not 3.0.0**, everywhere here: the meta depends on `tls-updates`, which is
`Min 3.0.5`, and a looser floor would recreate the `accountsapp` dep-gap drift. **No `DeviceCompatibility`**
— 3.0.x is TouchPad-exclusive and `Min`/`Max` is the hard filter (the `accountsapp` precedent); this also
avoids propagating the known-inert `"Touchpad Go"` string into 54 new stanzas.

**Why Atlas is a dependency of *every* Part 2 package** (user's call, 2026-08-24): the OAuth sign-in flow
lives in `com.palm.app.cloud-auth` (inside `generic`) and `cloudAuth.js` swaps `enyo.BasicWebView`'s plugin
mime to `application/x-atlas-browser`, i.e. it renders the sign-in page through **BrowserServer-atlas** —
the stock webview cannot complete a modern TLS handshake. Floored at `>= 0.9.11` because `cloudAuth.js`
notes it needs "the BrowserServer-atlas fix that makes setWindowSize's mult=1 viewport" and 0.9.11 is the
first release since 0.9.8 whose engine binary changed; **which build actually carries that fix is unconfirmed
— ask Herrie.** Note a few connectors never open a browser (`mega` email+password, `kdrive` token, `s3`
SigV4, `cdav` user/password, and the IM ones), so for them the dep is a uniformity choice, not a requirement.

**Why the meta depends on `tls-updates`:** `_cloudcore/httpcurl.js` shells every cloud request out to
`/usr/bin/curl` ("the device node runtime is OpenSSL 0.9.8k … all TLS lives in curl"), which is exactly what
`curl-tls13` replaces.

### Retiring `accountsapp` — and the migration postinst both metas now carry

`com.palm.app.accounts` 3.1.1 supersedes `org.webosarchive.accountsapp` 3.1.0 (superset; see that bullet).
`accountsapp` was removed from the feed and from `tls-updates`, which went **1.0.17 → 1.0.18** with
`com.palm.app.accounts (>= 3.1.1)` in its place. Rebuild was the standard meta recipe — `debian-binary` and
`data.tar.gz` reused verbatim, only `control.tar.gz` rewritten — verified by unpack-diff.

The hazard is that both packages replace `/usr/palm/applications/com.palm.app.accounts` and **each keeps its
own backup of what it found there**, under different names:

- accountsapp: `/media/cryptofs/webosarchive-accounts/stock-app.tar` (+ an `installed` marker)
- core-apps: `/media/cryptofs/core-apps-backup_usr_palm_applications_com.palm.app.accounts.tar` — note
  `BAK_ROOT` has **no trailing slash** and `$DST` starts with `/`, so this is a file sitting *beside* the
  empty `core-apps-backup` directory, not inside it.

With accountsapp still installed when 3.1.1 goes on, the core-apps postinst captures *accountsapp's build*
as "stock", and accountsapp's own prerm later restores its build over 3.1.1. So **both metas got a
`postinst`** that (1) adopts the genuine stock tar as our restore point and (2) re-points accountsapp's
backup at whatever is installed now, making its prerm a no-op. Idempotent, temp-file + `mv`, exits 0 when
accountsapp was never installed. Tested in a sandboxed fake root: adopt/re-point/idempotent all correct.

⚠️ **Two things that must not be "simplified" later:**
- **It must NOT delete accountsapp's backup.** That prerm treats "no backup + our marker file
  (`source/palmID/UsernameDialog.js`) present at the target" as *remove the app* — and 3.1.1 carries that
  same marker, being a superset. Deleting the backup converts "restores an older app" into "deletes the
  Accounts app outright with nothing put back".
- **It must NOT call `ipkg`.** The postinst runs *inside* Preware's own
  `ipkg -o /media/cryptofs/apps -force-overwrite install` (`luna_methods.c:1692`); a nested ipkg blocks on
  the lock or races the outer process's `status` write. (`tls-updates` 1.0.17 also *declared* accountsapp as
  a dep, so removing it behind Preware's back would leave an installed meta with an unmet dependency.)

### Four upstream installer bugs — found, fixed and hardware-verified (2026-08-25)

All four are in the **shared `packaging/lib/{postinst,prerm}`** that every one of Herrie's trees
copy-pastes (core-apps, app-services, chatthreader, enyo, luna-systemui). Fixed in
`~/Projects/webos-core-apps` on branch **`herrie/milestone2-servers-tab`** (PR to Herrie); the feed
ships repackaged ipks carrying the same fixes for the trees not yet forked. Issues #2/#3/#4 on
`Herrie82/webos-synergy-revival`; the deadlock is a fifth, separate issue.

1. **postinst deletes its own cwd → no Contacts/Messaging app.** The App Installer runs
   `pmPostInstall.script` with cwd = `/media/cryptofs/apps/usr/palm/applications/<pkg-id>`. On a
   TouchPad, Contacts and Messaging are ipkg-managed apps living at exactly that path, so `dest.txt`
   names it and `rm -rf "$DST"` removes the script's own cwd; GNU tar then fails `getcwd()` before it
   ever honours `-C` (`tar: Cannot save working directory`), exits 2, and the rollback restores an
   empty backup. **Fix: `cd /` before anything is deleted.** Only these two packages are affected —
   every other `dest.txt` is a rootfs path, not the installer's cwd.
2. **prerm never restores.** It locates its work through `$OV/dest.txt`, but postinst `rm -rf "$OV"`s
   as its last act, so on a real uninstall the lookup finds nothing and the restore is skipped
   silently. **Fix: fall back to `.last-installed/<pkg-id>.dest.txt`** (postinst already writes it for
   the WOSQI double-invocation guard), require an absolute path before any `rm -rf`, and clear the
   marker on success — otherwise a second uninstall call deletes the stock app the first one restored.
3. **postinst SIGTERMs its own installer.** The cmdline sweep kills any process matching a fixed
   string; `ipkg`, `ApplicationInstallerUtility` and the script itself all carry the package name or
   the `.ipk` path. When the swept string is a substring of the package's own id the installer dies:
   the package installs fine, webOS reports `FAILED_IPKG_INSTALL`, Preware aborts the batch, and
   retrying replays it forever (Preware only refreshes its installed list on Update Feeds — four
   identical retries observed). Hits `com.palm.service.contacts.linker` ("contacts.linker") and
   `com.palm.messaging.chatthreader` ("chatthreader"). **Fix: `nudge_kill()` skips `$$` and anything
   matching `pmPostInstall|pmPreRemove|ApplicationInstallerUtility|/usr/lib/ipkg/|\.ipk`.**
4. **⚠️ Blocking `luna-send` deadlocks the installer AND the UI.** Both scripts ended with
   `luna-send -n 1 luna://com.palm.applicationManager/rescan '{}'`, which waits for a reply — but the
   scripts run *inside* LunaSysMgr's own request handling, so the service that owes the reply is the
   one waiting for the script to exit. Nothing times out. Symptom: Preware stuck on
   "Downloading/Updating — [core-apps] removal complete", tablet unresponsive, **no error anywhere**;
   the only evidence is a blocked `luna-send` in `ps -ef`. Killing that one process releases the whole
   chain. **Fix: fire-and-forget —
   `( luna-send … & ) >/dev/null 2>&1`.** Leave the `putKind`/`putPermissions` calls blocking: they
   parse their replies and talk to `com.palm.db`, not to the caller.

⚠️ **The deadlock fix cannot apply to its own first upgrade.**
`/media/cryptofs/apps/.scripts/<pkg-id>/pmPreRemove.script` is written at *install* time and is what
runs at the next removal, so a device carrying a pre-fix build deadlocks once during the remove half
before the fixed package can land. Only affects devices that installed during the broken window (a
plain `ipkg` install/downgrade does NOT refresh those stored copies). Device-local repair, no feed
change: rewrite the line in place, syntax-check each file, then update normally —
`sed "s|^luna-send -n 1 luna://com.palm.applicationManager/rescan .*|( & \& ) >/dev/null 2>\&1|"`
over `/media/cryptofs/apps/.scripts/*/pm{PreRemove,PostInstall}.script` (25 files on the affected
device; verified byte-identical apart from that line).

**Recovery for a missing core app: uninstall, then REBOOT.** Once our package is unregistered,
webOS's own boot-time `app-install` service reinstalls the stock ipk from `/usr/palm/ipkgs/<id>/`.
That is also why a half-broken device *stays* broken — while our higher version is registered the
service logs "already installed. skipping...". No special backup machinery is needed or wanted; the
postinst now simply refuses to save an empty-directory backup (a restore point that restores nothing)
and says so.

### Upstream merged those fixes — and finished them (2026-08-25). Feed re-cut in place.

Herrie merged the core-apps PR and **ported it to the other four packaging trees**
(`enyo-1.0` 57ba6b2, `luna-systemui` d74e8f5, `com.palm.messaging.chatthreader` 53e80fd,
`app-services` 46aefaa). Verified by unpacking every feed ipk and diffing its scripts against the
merged files:

- **core-apps family — already current.** `com.palm.app.{accounts,contacts,messaging,phone}`,
  `messaging.library` and `contacts.plugin.messaging` are **byte-identical** to the merged tree
  (that PR is our own work). Nothing to do.
- **The other six were behind.** Herrie did not just transplant the four fixes, he finished them.
  Three additions our repackaged builds lacked: `bak_is_usable` (an empty-directory tar is a restore
  point that restores nothing — rebuild it instead of trusting it forever); **the per-mode manifests
  persisted beside the marker** (`.last-installed/<pkg-id>.{preserve,files}.txt`) with `PLIST`/`FLIST`
  fallbacks in prerm; and **clearing the markers after a successful restore**, without which a second
  prerm call (WOSQI makes one) re-resolves `DST` and `rm -rf`s the stock component the first call just
  restored — whose backup tar that first call had already deleted.

⚠️ **The `luna-systemui` case was a device-breaking bug that WE introduced.** It is the only package
here in **surgical mode** (`files.txt` = `app/FilePicker/AlbumGridView.js`), and its prerm picks the
branch with `[ -f "$OV/files.txt" ]`. On a normal uninstall `$OV` is gone — exactly the case our
`LASTDEST` fallback was added to handle — so `DST` now resolves (before our change it did not, and the
whole block was skipped), `files.txt` does not, and control falls into the **whole-dir** branch:
`rm -rf /usr/lib/luna/system/luna-systemui`, then no whole-dir backup tar exists (postinst made a
per-file one), so it logs "was never on stock" and stops. **The system UI directory is deleted with
nothing put back.** Upstream's `FLIST` fallback is what closes it.

**And it is sticky, like the deadlock:** the stored
`/media/cryptofs/apps/.scripts/luna-systemui/pmPreRemove.script` is what runs at the next removal, and
Preware updates by remove-then-install — so pushing the fixed build to a device already carrying the
old one *is* the trigger. Device-local pre-repair (no script editing — just give the old prerm back
the markers it looks for, after which it takes the surgical branch correctly and the new postinst
cleans the directory up as usual):

```sh
mkdir -p /media/cryptofs/luna-systemui-overwrite/luna-systemui
echo /usr/lib/luna/system/luna-systemui > /media/cryptofs/luna-systemui-overwrite/luna-systemui/dest.txt
echo app/FilePicker/AlbumGridView.js   > /media/cryptofs/luna-systemui-overwrite/luna-systemui/files.txt
```

**How the six were shipped: re-cut IN PLACE at the same version** — `enyo-accounts 1.1.1.1`,
`enyo-contactsui 1.0.1.1`, `luna-systemui 3.1.1.1`, `com.palm.messaging.chatthreader 1.1.0.2`,
`com.palm.service.accounts 1.1.0.1`, `com.palm.service.contacts.linker 1.1.0.2`. ⚠️ This is a
**deliberate exception** to the "once anything might be on the server, never re-cut at the same
version" rule, made by the user on the grounds that almost nobody has installed them yet (the two test
tablets were repaired by hand). The consequence is the usual one: **no device already carrying these
versions will ever be offered the fix** — it must be uninstalled/reinstalled or repaired by hand. Do
not treat this as precedent.

Upstream touched only `packaging/lib/*`, so the rebuild was the standard repackage: `debian-binary`
and `data.tar.gz` reused **verbatim**, only `control.tar.gz` re-rolled (same member order/modes/mtimes,
only `./postinst` and `./prerm` replaced) plus the `pmPostInstall.script` / `pmPreRemove.script` ar
members. Verified by unpack-diff — `./control` and every payload byte unchanged — then index `MD5Sum`/
`Size` updated for those six stanzas only (no `Source`, `Version` or `Depends` edits), `Packages.gz`
regenerated, and the full per-device sweep re-run: 90 stanzas / 69 ipks, deps OK at both
`ignoreDevices` settings, only the known pre-existing `squid` gap.

### The connectors carried it too — now on upstream's own fix (2026-08-25)

Herrie's five ports covered core-apps, enyo-1.0, luna-systemui, chatthreader and app-services. The
connectors live in **`webos-synergy-revival`**, which was not in that set, so all 20 still shipped the
blocking `luna-send -n 1 …/rescan` in both postinst and prerm (issue #6).

The feed was first patched here with a minimal wrap of that one call; **`Herrie82/webos-synergy-revival`
e71c8e4 then landed the canonical fix and the feed was re-cut from his scripts instead.** His version
is a superset, and the extras are worth knowing:

- **`ls-control scan-services`** after install (cloud + messaging + generic). Not a deadlock fix at
  all: ls-hubd only scans `/usr/share/dbus-1/system-services/*.service` at **its own** startup, so a
  freshly installed connector's service file sits on disk unseen until a reboot — reported live as
  "Service not listed in service files: com.palm.service.mega". Our stanzas set `RestartDevice`, which
  masks it, but the fix belongs in the script.
- **`nudge_kill()` in `generic/postinst`**, replacing the unguarded `contacts.linker` `/proc` sweep
  (issue #3) that our own `0.9.3.1` repackage had left in place.
- **`cd /`** in all eight scripts (issue #4, defensive here), and **host-runnable tests** under
  `tests/` — a static sweep for un-backgrounded `luna-send`, a `nudge_kill` matrix against a synthetic
  `/proc`, and a sandboxed install→uninstall restore test.
- He also notes this repo was **never** vulnerable to issue #2: `apply_rootfs_overwrite()` plus a
  per-PKGID applied-paths manifest never depended on the `$OV/dest.txt` that postinst deletes.

Verified on the CE 3.1.0 tablet that the appinstaller calls are the same deadlock class:
`/usr/share/ls2/roles/prv/com.palm.luna.json` lists **both `com.palm.applicationManager` and
`com.palm.appinstaller`** in LunaSysMgr's `allowedNames`, which is why `carddav/postinst`'s two
`notifyAppInstalled` calls are wrapped too. Left blocking on purpose:
`com.palm.service.accounts/listAccountTemplates` (a separate JS service) and the already-backgrounded
`checkStatus` ping.

**Before swapping wholesale, check the scripts are not per-connector.** They are not: a diff of every
shipped script against its family file found exactly **one** shipped-only code line in all 21 packages
(generic's old sweep). So each ipk takes its family's scripts verbatim — `carddav` (cdav), `cloud`
(12), `messaging` (7), `generic` (1). Classify by similarity if you need to re-derive the mapping, but
use a real `ratio()`: `difflib`'s `quick_ratio()` is a bag-of-characters estimate and mis-sorted every
cloud connector into carddav.

Re-cut **in place at the same versions** again — payload and `./control` byte-identical, scripts
byte-equal to upstream's, `sh -n` clean, index md5/size refreshed on **41 stanzas** (20 connectors × 2
+ generic). Same standing caveat as the six above: a device already carrying a connector runs its
**stored** `pmPreRemove.script` at the next removal, so it needs the device-local `sed` in
`~/Desktop/synergy-revival-connector-deadlock.md` (the write-up for Herrie, now superseded upstream but
still the reference for the repair).

### The SIGBUS: fixed upstream, and our diagnosis was WRONG (2026-08-25)

Filed as issue #8; `Herrie82/webos-synergy-revival` **7db2aa6** fixed it the same day, and the feed's
`com.palm.synergy.generic 0.9.3.1` was re-cut from his scripts. **The `umount` was a bystander.** Do
not repeat the reasoning that got this wrong: we saw `prerm` unmount two live fuse bind mounts, saw
`WebAppMgr`/`LunaSysMgr`/`BrowserServer` die with `received 7`, and joined them. But a **non-lazy
`umount` of a busy mount returns `EBUSY`** — it detaches nothing and signals nobody. Proximity in the
script is not causation.

The real cause was three lines further up, and the crash trio identifies it exactly: those three
processes are precisely the set that **mmap `/usr/lib/libWebKitLuna.so`**, and prerm's restore was

```sh
restore() { [ -f "$1" ] && cp "$1" "$2" && log "  restored $2"; }
restore /media/internal/libWebKitLuna.so.prewebm /usr/lib/libWebKitLuna.so
```

A plain `cp` **truncates and rewrites the live inode in place**; any mapper faulting a page during
that window — or past the truncation point — takes SIGBUS. `generic`'s own postinst already knew this
and used temp+rename for the same library in its webkit-webm-mime patch; only prerm didn't.

Upstream's fix, now in the feed:
- `restore()` copies to a same-directory temp and **atomic `mv`** (rename leaves current mappers on
  the old inode). Also fixes a silent `ETXTBSY` when restoring the running `PmBtEngine`, and an
  `[ -e "$2" ]` guard stops it conjuring a spare 12.8MB `libWebKitLuna.so` onto the 559MB root.
- **`atomic_cp()`** in postinst for the same bug class on the *upgrade* path — the Thai font
  (`HeiT_nb.ttf`, mapped by every FreeType user) and the gst-0.10 plugins (mapped by a mid-playback
  media-pipeline). It `cmp`-skips identical files, so a routine reinstall doesn't churn the inode.
- **`umount -l`** on both bind mounts — not the crash cause, but a plain `umount` strands the
  mountpoint (`EBUSY` → failed `rmdir`) if a straggler still holds it.
- `tests/test-prerm-atomic-restore.sh` holds an open fd across a sandboxed prerm run: 6 failures
  pre-fix, 12/12 pass post-fix.

**The general rule this leaves behind:** never `cp` over a file the running system has mapped or is
executing — write beside it and `mv`. That covers every binary-swap package in this feed, not just
Synergy.


### Repackaging convention for upstream ipks: append `.N`

When we patch an upstream ipk whose version we do not control, ship it as `<upstream>.1` (then `.2`,
…): it sorts **above** upstream's version and **below** their next release, so their fix supersedes
ours automatically with no same-name-same-version collision in the index. Used for
`contacts 3.0.6701.2`, `messaging 3.0.6607.2`, `contacts.linker 1.1.0.2`, `chatthreader 1.1.0.2`,
`generic 0.9.3.1`, `service.accounts 1.1.0.1`, `enyo-accounts 1.1.1.1`, `enyo-contactsui 1.0.1.1`,
`messaging.library 1.3.1.1`, `contacts.plugin.messaging 12.2.0.1`, `luna-systemui 3.1.1.1`,
`accounts 3.1.1.1`, `phone 2.0.1.1` — with the roll-up at **1.0.3** floored to all of them. ⚠️ Never
re-cut at the same version once anything is on the server: Preware compares the version STRING only.

### Hardware status (2026-08-25)

**Both paths verified on 3.0.5 TouchPads, unassisted, through Preware from the feed:**
- **Clean/stock device** — fresh install of all 16 (28-package chain incl. Atlas): 0 blocked rescans,
  0 `FAILED_IPKG`, 0 `runScriptCwd` errors, all apps complete on disk, and — the point — all 12
  stored `pmPreRemove.script` copies are the *fixed* ones, so that device can never hit the deadlock.
- **Previously-broken device** (survived two failed attempts) — after the stale-script repair above:
  full chain including eleven remove-then-install replacements, 0 crashes, 0 blocked, 0 failures.

⚠️ **The 20 Part 2 connectors have never been installed on any device.** Part 1 and the roll-up are
well tested; the connectors are not. Cheapest first test: one cloud connector (Dropbox) and one IM
connector, both of which pull the meta + Atlas.

### Icons, and the rest of the metadata

Icons were extracted from the packages' own payloads into `ipkgs/assets/icons/synergy/synergy-*.png`
(24 files, script in this session's scratchpad: pick `icon-256x256.png` > `icon.png` > `-64` > `-48`,
allowing non-square since several vendor logos are e.g. 48x45). `flickr`'s own icons are 97–176-byte
placeholders, so it uses `synergy-generic.png` (the cloud-auth icon, also used by the meta and the
framework/service packages). `cdav` had to be forced to `caldav-48.png` — a naive "largest `_64`" pick
grabbed `icloud_64.png`. `Feed: "WOSA Modernize"` with **`Category: "Synergy"`** (user's call — same feed
group, new category). `Type`: `OS Application` for Part 1 + meta, `Application` for connectors.
`PostInstallFlags`/`PostUpdateFlags`: **`RestartDevice`** for the meta, `generic` and every connector (new
ls2 roles / dbus services, which the hub only reads at startup), `RestartLuna` for the Part 1 app and
framework replacements; `PostRemoveFlags: RestartLuna` throughout.

**Titles and icons (user's calls, 2026-08-24, index-only — no ipk was rebuilt for any of this).** The
12 Part 1 app/framework/service packages are titled `<Name> (For Synergy Revival)`; the connectors are
`<Service> (Synergy)`; the runtime is `Synergy Shared Runtime` and deliberately carries **no `Icon`**,
so Preware shows its default box (same as `tls-updates`). The meta is
**`" Synergy Revival Roll-up (Touchpad)"`** — note the deliberate leading space (see the sorting note
above) and the user's spelling of "Touchpad" here, which differs from the `(TouchPad)` used elsewhere
in the feed. Its description opens with "This roll-up installation prepares your 3.0.5 Touchpad for
Synergy Revival." and it uses `assets/icons/synergy/synergy-rollup.png` (supplied by the user).
`com.palm.app.accounts`'s index `Maintainer` is **`webOS Archive`**, not Herrie: that app started life
as our `accountsapp` build and was given to him. The remaining Part 1 packages correctly credit him.
`synergy-generic.png` (the cloud-auth icon) is now used only by `Flickr` — whose own icons are
97–176-byte placeholders — and `Luna System UI`.

**Not yet run on any hardware.** First test should be a 3.0.5 TouchPad: install `synergy-revival` from the
feed (expect ~130MB before connectors, one reboot at the end), then one OAuth connector (Dropbox is the
cheapest) and one IM connector. Then a CE 3.1.0 device, where the same connector must install with **no**
Part 1 and no meta.

## Phone support (webOS 2.2.4: Pre3 / Veer / Pre2) — LIVE + **Pre 3 hardware-verified**

**One feed URL for every device.** Phones and TouchPad both use
`http://stacks.webosarchive.org/feeds/modernize/ipkgs`. No per-device feeds, no second URL.

### Why the packaging had to change first (the thing that blocks every feed-layout fix)

Upstream `OpenSSL-legacyWebOS` builds the four device-specific packages once per board but gives
every build the **same `Package` + `Version` + `Architecture`** (`browser-tls13` / `1.1.2` / `armv7`,
four times, different contents). ipkg 0.99.163's dedupe key is exactly that triple
(`pkg_vec.c`, `pkg_vec_insert_merge`) and for feed parsing it takes the
`/* just overwrite the old one */` branch — across **all** configured feeds, because they share one
hash table. Preware then installs **by name** and lets ipkg choose the file. So per-board ipks that
share a name can never coexist in any layout: a Pre 3 could be handed the topaz build, and
`luna-tls13` (the package that edits `/etc/event.d/LunaSysMgr`) had no md5 guard to catch it.
Dead ends also checked and ruled out: a **subdir in `Filename`** (`ipkg_download.c:118` builds the
URL right, `:124` builds the local cache path wrong — ipkg's own source flags the bug), a **per-board
`Architecture:`** (unknown arch is rejected unless every device's `ipkg.conf` declares it first — a
bootstrapping problem), and **`Provides:`** (ipkg supports it; Preware's JS has zero references, so
Preware reports an unmet dep).

### The fix: one package per name, board chosen at install time

`./build-ipks.sh phone` (a `phone` pseudo-device added to the upstream build script) emits
**one ipk per package covering all three phones**, into `ipks/phone/`, named `*-phone`:

| package | per-board content bundled |
|---|---|
| `org.webosinternals.browser-tls13-phone` **1.1.2** | 3 × `BrowserServer.rpath.<board>` (~250KB each); the 3.9MB ssl11 OpenSSL/curl payload is shared |
| `org.webosinternals.downloadmgr-tls13-phone` **1.0.0** | 3 × `LunaDownloadMgr.rpath.<board>`; shared mail libcurl |
| `org.webosinternals.luna-tls13-phone` **1.1.4** | none per-board — but **1.1.4 added a 440KB payload** (see below); still byte-identical across all three boards |
| `org.webosinternals.mojomail-imap-tagfix-phone` **1.0.0** | none — postinst only (per-board byte offset + 2 md5s) |
| `org.webosarchive.tls-updates-phone` **1.0.1** | payload-free one-tap meta (hand-built in THIS repo, like `tls-updates`) |

#### `luna-tls13-phone` 1.1.4 — two env-scrub wrappers (the real content of this release)

1.1.3 was payload-free (5,590B); **1.1.4 is 454,092B** because it now ships two static ARM wrapper
binaries under `files/`. Both fix the same root cause: the launcher patch exports `LD_BIND_NOW=1` and
the ssl11 compat preload into LunaSysMgr's environment, and **child processes inherit it**. Eager
binding then hits `libpvrtc.so`'s lazily-unresolved `NApp_*` symbols and the child dies at exec,
before `main()`.

| wrapper | installed as | what was broken without it |
|---|---|---|
| `jailer.wrap` | `/usr/bin/jailer` (stock → `.real`) | webOS 2.x has no `setcpushares-pdk`; LunaSysMgr execs `jailer -t pdk …` directly, so **every PDK ("Linux binary") app silently failed to start** — `applicationManager/launch` still returned a processId, the death was in the child |
| `setcpushares-task.wrap` | `/usr/sbin/setcpushares-task` (stock → `.real`) | the App-Manager install path runs `setcpushares-task /usr/bin/ApplicationInstallerUtility …`; `/bin/sh` died at exec (127 / status 32512) before the installer ran, so **Preware (installSvc/replaceSvc) and WOSQI wedged on a stuck IPKG lock** |

Both wrappers are the **same static binary** (md5 `b87bcc11845aeda28eca13e5d3b7ae2f`, also identical to
the TouchPad's `setcpushares-task.wrap`) — it strips only the tls13 additions and execs the `.real`
beside it. Static on purpose: a shell-based scrub cannot run when the env is what kills `/bin/sh`.

Two details worth keeping:
- **Both blocks run BEFORE the launcher "already patched" short-circuit**, so an existing 1.0.0–1.1.3
  install picks the wrappers up on upgrade rather than hitting `exit 0` and skipping them.
- **Stray-wrapper detection differs per wrapper**, because you cannot grep a binary portably: the
  script wrapper keys on a `#!` shebang, the jailer on **size** (stock ~124KB vs wrapper ~450KB, so
  `>300000` with no `.real` beside it ⇒ refuse to wrap). `prerm` restores from `.real`, falling back to
  `/var/luna/{jailer,setcpushares-task}.tls13-orig`.

The TouchPad already had the `setcpushares-task` wrapper (since 1.1.2, alongside `setcpushares-pdk`
and `media-pipeline`) — this release is the phone line catching up, plus a `jailer` wrapper the
TouchPad does not need.

Total **2.73MB** for all three phones (vs 6.7MB for the three separate per-board feeds this replaced).
The postinst resolves the board from **`/etc/prefs/properties/machineName`** (what Preware's own
ipkgservice reads — `luna_methods.c:561`), falling back to matching known board names in
`/etc/palm-build-info`, then `case`s to that board's binary/offset/md5s.

**This gives these packages the hard wrong-device guard they never had:** an unrecognised board —
including a TouchPad — exits non-zero *before touching anything*. Feed metadata can't do that, since
`DeviceCompatibility` is bypassable by a Preware pref. Verified 8/8 in a harness: each board selects
its own correct values; topaz / bogus / missing all refuse; trailing-newline and the build-info
fallback work. **Then verified end-to-end on a real HP Pre 3** — installing `tls-updates-phone`
resolved, dispatched and patched correctly, which retires the two previously-unverified assumptions
(`modelNameAscii == "Pre3"`, and a Pre 3 resolving to `mantaray`). Not yet exercised on
broadway/roadrunner.

**Single-board targets are untouched.** `tgt_suffix`/`tgt_boards` return `""`/`"$dev"` for a real
board, so `ipks/topaz/` keeps the historical flat payload filenames. A rebuild-and-compare of all
four topaz ipks showed only the intended `BS_FILE=`/`DL_FILE=` indirection line — the deployed
TouchPad packages are unchanged.

### The `Depends` consequence — commons can't name a device-specific provider

`/usr/lib/ssl11` now comes from a differently *named* package per family (`browser-tls13` vs
`browser-tls13-phone`), so the device-INDEPENDENT packages cannot declare it: one `Depends` line
can't name both, and declaring either makes the package uninstallable on the other family. So
`mail-tls13` **and** `curl-tls13` now have empty `Depends` (upstream had already done this for
`ntpdate-sync`; `mojomail-imap-tagfix`'s bogus `mail-tls13` dep is gone too). Both postinsts already
refuse (`exit 1`) unless `/usr/lib/ssl11` is present, and the two `tls-updates*` metas supply the
install ORDER. `mail-tls13` was rebuilt upstream for this (payload identical, one control line);
`curl-tls13` had **only its control.tar.gz repacked** in this repo — deliberately NOT rebuilt from
upstream, because upstream's same-version `1.0.1` carries different bundled OpenSSL binaries and this
package replaces `/usr/bin/curl` that Synergy depends on.

### Gating + the check to re-run

Phone stanzas: `Min 2.2.4` / `Max 2.9.9` + `DeviceCompatibility ["Pre3","Veer","Pre2"]`, `(Phones)`
title suffix, bold not-for-TouchPad lead. `Min 2.2.4` deliberately excludes an un-upgraded Veer
(2.2.0) or Pre 2 (2.1.0) — their stock binaries differ from what the patches were built against.

**`tls-updates-phone`** is titled `" TLS 1.3 Updates (Phones)"` — ⚠️ deliberate leading space, same
as the TouchPad meta, see the sorting note above.

**`tls-updates-phone` 1.0.1 carries the phone meta's first version floor**
(`luna-tls13-phone (>= 1.1.4)`); every other dep of that meta is still unversioned. It shipped at
1.0.0 with no floors at all, so an existing phone install would never have been dragged up to a new
member version — the floor plus the meta's own `1.0.0 → 1.0.1` bump is what makes the 1.1.4 fixes
actually reach a device that already ran the bundle. Same rule as the TouchPad meta: a new floor needs
the meta's own version bumped **and** its ipk control rebuilt to match the index.

**After ANY change to gates, names or versions, re-run both checks** (they have each caught real
bugs): (1) **no two stanzas share name+version+arch *while pointing at different files*** — the ipkg
dedupe trap. Note the qualifier: Atlas deliberately has two stanzas on one file, which is safe (see
"Atlas's two stanzas"); a bare name-triple check would false-positive on it; (2) **simulate
`loadPackage` per device** (TouchPad 3.0.5 / CE 3.1.0 / Pre3 2.2.4 / Veer 2.2.4+2.2.0 / Pre2
2.2.4+2.1.0) and assert every visible package's deps are also visible. That second check is what
found `curl-tls13`'s and `mail-tls13`'s stale deps. The visibility check also verifies **version
floors** resolve against what the feed actually ships, not just that the dep is visible.

Current result — **90 stanzas** (69 packages; Atlas and each of the 20 Synergy connectors have two),
all valid, with the sweep run at **both settings of the `ignoreDevices` pref** (added 2026-08-23, since
`Max`/`DeviceCompatibility` are soft and only `Min` survives that toggle):

```
ignoreDevices = OFF (default)
TouchPad 3.0.5    63 visible  synergy 21  atlas deps: tls-updates  metas: tls-updates, synergy-revival  deps OK
TouchPad CE 3.1.0 35 visible  synergy 20  atlas deps: (none)       metas: (none)                        deps OK
Pre3 2.2.4        20 visible  synergy  0  atlas: not visible       metas: tls-updates-phone             deps OK
Veer 2.2.4        11 visible  synergy  0  atlas: not visible       metas: tls-updates-phone             deps OK
Pre2 2.2.4        11 visible  synergy  0  atlas: not visible       metas: tls-updates-phone             deps OK
Veer 2.2.0         3 visible  synergy  0  atlas: not visible       metas: (none)                        deps OK
Pre2 2.1.0         3 visible  synergy  0  atlas: not visible       metas: (none)                        deps OK

ignoreDevices = ON
TouchPad 3.0.5    69 visible  synergy 21  atlas deps: tls-updates                                       deps OK
TouchPad CE 3.1.0 69 visible  synergy 21  atlas deps: (none)                                            deps OK  <- order works
Pre2 2.1.0         5 visible  squid -> glibc/openssl not visible (PRE-EXISTING, Min-gated)
```

Two more assertions, added with Synergy and worth re-running: the **connector stanza that wins** must be
the dep-free-of-Part-1 one at 3.1.0 and the `synergy-revival`-carrying one at 3.0.5, in **both** pref
states (all 20 connectors checked, all four cases correct); and `synergy 21` vs `synergy 20` is the
Part 1 runtime (`com.palm.synergy.generic`) correctly disappearing at CE 3.1.0.

The Atlas `deps:` column is the assertion that matters: **`tls-updates` at 3.0.5, nothing at 3.1.0, in
both pref states.** The lone `squid` gap is pre-existing (verified against `git show HEAD~1:` at the
time), appears only under `ignoreDevices` on a 2.1.0 Pre2, and is a `Min`-gate artifact of the nizovn
stack, unrelated to anything here.

⚠️ **CE 3.1.0 coverage.** The convention paragraph above says "use `Max 3.9.9`, not `3.0.9`", but
every TouchPad stanza except Atlas's now carries `Max 3.0.9`, so a `3.1.0` device (i.e.
`staging/org.webosce.luna-update`) sees 15 packages instead of 31 and gets **no one-tap meta**. The
one thing this used to *break* — Atlas visible at 3.1.0 with its `tls-updates` dep invisible — was
fixed on 2026-08-23 by giving Atlas a separate, dep-free 3.1.0 stanza. What is left is a policy
choice, not a bug: move the TouchPad stanzas back to `3.9.9`, or declare CE 3.1.0 out of scope. Pick
one and fix the convention paragraph to match.

### Feed vs `webOSArchive/OpenSSL-legacyWebOS` — verified in sync (2026-07-28)

Both repos clean and fully pushed (`origin/main...main` = 0/0). Full unpack-and-compare of every
file inside every shared ipk. **All four `phone/` packages are byte-identical to the feed**, as are
`mail-tls13` 1.3.2 and `ntpdate-sync` 2.0.1. `tls-updates-phone` is correctly absent from that repo
(hand-built here, like `tls-updates`). Three files differ, none of them a problem:
- **`topaz/browser-tls13`** — payload identical; only the control's `Description`/`Source` text
  (repo carries the newer `(topaz)`-suffixed titles).
- **`topaz/luna-tls13`** — payload identical; postinst differs by one **comment reword**, prerm by
  one **blank line**. Repo's control declares `Depends: browser-tls13` where the feed ipk's is empty
  — harmless, Preware resolves from the index, which has the dep.
- **`curl-tls13`** — bundled `libcrypto.so.1.1` / `libssl.so.1.1` differ. **Deliberate, leave it**
  (see the ⚠️ note in the package inventory: upstream shipped different binaries at the same version
  `1.0.1`, and this package replaces the `/usr/bin/curl` Synergy depends on).

If the two topaz controls ever get re-synced, do it **without a version bump** — payloads are
identical so it's cosmetic, and same-version means existing installs are untouched while new ones
get the tidier control (the mojomail/ntpdate precedent). Never sync `curl-tls13`.

**Two previously-noted upstream drifts are now RESOLVED** (both were "worth fixing upstream sometime";
the 2026-07-28 comparison shows they're gone): `topaz/downloadmgr-tls13` now matches byte-for-byte,
and `topaz/luna-tls13` now ships the PalmPDK-built `setcpushares-pdk.wrap` (the 457,924B vs 382,496B
mismatch against the committed `setcpushares-pdk-wrap.bin` no longer appears in the payload).

**Build host** needs GNU `ar` (`brew install binutils`) and `patchelf`. The stock Palm binaries under
`devices/<board>/` are **not committed** (and no `devices/` dir exists in a fresh clone), so
`prebuilt_rpath()` (`build-ipks.sh:197`) is what makes the repo self-sufficient: it reuses the
already-RPATH'd binary from the committed per-board ipk — verified bit-identical to all six shipped
binaries. ⚠️ **Do not prune `ipks/{topaz,mantaray,roadrunner,broadway}/`** — rebuilding `phone`
depends on them.

## Held in `staging/` (NOT in the live feed)

- **`org.webosce.luna-update`** = "webOS CE 3.1.0". Sets the OS version string AND swaps
  `/usr/bin/LunaSysMgr` with the **LunaCE 5.0.0** binary + tweaks; depends on the whole chain
  (incl. QupZilla, and swupdate-redirect). **It semi-bricked a TouchPad.**
- **`org.webosce.swupdate-redirect`** — OTA server redirect; held because the URL change isn't
  sufficient (see findings). luna-update lists it as a dep; re-add them together.
- Each has its `.Packages-stanza.txt` preserved + restore steps in `staging/README.md`.

## ⚠️ The brick lesson (why luna-update is held)

`luna-update` swapped the LunaSysMgr **binary** (LunaCE) while `luna-tls13` (a dependency) had
**already edited `/etc/event.d/LunaSysMgr`** to force the app host through OpenSSL 1.1.1
(`/usr/lib/ssl11`). On reboot the modified launcher ran the LunaCE binary with 1.1.1 forced onto
its library path → ABI mismatch on `libssl`/`libcrypto` (LunaCE was built vs 0.9.8) → LunaSysMgr
crash loop → semi-brick. `luna-tls13` ALONE is fine (the healthy test device runs it, UI up).
**Lessons:** never bundle the LunaSysMgr binary swap with luna-tls13's launcher edit; keep the
LunaCE swap its own opt-in package, tested standalone; keep the harmless version-string change
separate from launcher/binary changes. Recovery if it happens again (novacomd survives a UI
crash): restore `/usr/bin/LunaSysMgr.webosce-orig` and `/var/luna/LunaSysMgr.tls13-orig`→
`/etc/event.d/LunaSysMgr`, reboot.

## ⚠️ The mid-batch restart lesson (Atlas repackage)

**Never `killall LunaSysMgr` / reboot inside a postinst (or prerm) of a package that installs as part
of a Preware dependency batch.** Preware itself runs *under* LunaSysMgr, so restarting it mid-batch
kills Preware and **aborts every remaining package in the chain**. Atlas's upstream postinst ended
with `killall LunaSysMgr` (to load its NPAPI plugin) — since `atlas-default-browser → atlas →
tls-updates`, that fired an "immediate reboot" the moment Atlas installed, before the patch (and
anything after) could install. **Fix:** remove the restart from the script and declare it as the
package's Preware `PostInstallFlags`/`PostUpdateFlags`/`PostRemoveFlags` instead — Preware collects
the flags across the whole batch and applies the **strongest one ONCE at the end** (RestartDevice >
RestartLuna). So the engine/plugin reload still happens, just cleanly after the chain completes.
- **WOSQI does NOT honor Preware `Source` flags** (it's not a feed client) and does **not** resolve
  `Depends:`. So a WOSQI install of these gives no auto-deps and **no** end restart — you must install
  each ipk yourself and reboot manually (and WOSQI's postinst behavior has been inconsistent on our
  device; safest is to run postinsts via novacom). The dep-chain + single-end-restart design is a
  **Preware-from-the-feed** thing; test it that way.
- **Same-version won't reinstall:** Preware compares the Version STRING, so a device already carrying
  `atlas-default-browser 1.0.0` (e.g. from a manual novacom test) will NOT pick up a rebuilt-but-still-
  1.0.0 feed copy — its stale control (empty Depends) lingers. Bump the version to force it. Likewise,
  installing the patch when Atlas is **already installed** won't re-pull tls-updates (Preware doesn't
  recurse into an already-satisfied dep node) — remove Atlas first (or test on a fresh Doctor) to see
  the full chain.

## OTA / "System Updates" — see `staging/swupdate-redirect-FINDINGS.md`

Short version: `com.palm.app.updates` is just UI → `palm://com.palm.update/` = `/usr/bin/UpdateDaemon`
(OMA-DM/SyncML). Server URL is in `/usr/share/omadm/DmTree.xml` + runtime `/var/lib/software/DmTree.xml`
(+ backup). BUT the host is actually derived from a `DMCARRIER` token → carrier → domain
(`/var/lib/software/domain`, injected via `OmaDm -set_domain`); on a Wi-Fi TouchPad the carrier is
empty → "Unrecognized carrier" → the check aborts before any network. So the DmTree URL redirect is
necessary-but-insufficient; the real fix is a binary patch of UpdateDaemon (force domain, skip
carrier/roaming gating). Codepoet80 (the user) maintains the `webos-update-exploration` repo.

## Preware self-bootstrap (separate repo `~/Projects/preware`, NOT yet committed/pushed)

We bumped Preware 1.9.16→**1.9.17**, added the modernize feed (http) to its default feeds, and
**injected `./postinst`/`./prerm` into `control.tar.gz`** (palm-package only emits the Palm-installer
`pmPostInstall.script`; ipkg/feed installs need control.tar.gz/postinst, else the feed setup never
runs). Build keyless via `./build.sh arm` (signing keys absent + unreproducible; fine for
ipkg/WOSQI installs which don't check the Palm signature). The Preware 1.9.17 ipk is in this feed.
On a fresh Doctor: enable Dev Mode → WOSQI-install Preware 1.9.17 → its postinst writes
`modernize.conf` → Update Feeds → patches appear. **Verify** `/media/cryptofs/apps/etc/ipkg/
modernize.conf` exists after install (the one spot that depends on the install hook running).
The `~/Projects/preware` source edits are still uncommitted per the user's instruction.

## User-facing docs — separate repo `~/Projects/webos-docs` (MkDocs, `readthedocs` theme)

The public guide (activate → install apps → get online → use the device). Build/validate with
`mkdocs build --strict` from the repo root; `site/` is untracked, and one INFO about
`macos-install.md` not being in the nav is pre-existing and expected.

**2026-07-28: the two-path layout was collapsed.** The docs used to fork into "Setup · TouchPad
(webOS 3.x)" vs "Setup · Other Devices" (certs + proxy), which stopped being true the moment the
phone packages shipped. The real line is **webOS 2.2.4**, not tablet-vs-phone: at 2.2.4+ you run the
TLS Updates; below it you're limited to HTTP (and *can't* run a proxy either — that API arrived in
2.2.4, 2.2.0 on Pre3), so it isn't a second path, just a smaller set of options.
- `setup-path.md` **kept but retitled "Getting Ready"** — deliberately not deleted, because five
  pages link to `setup-path.md#set-the-date-time-first`. The fork became a capability table.
- `modern-tls.md` now covers both bundles and notes only one is ever visible per device.
- `online.md` / `proxysetup.md` moved to an "Older Devices (before webOS 2.2.4)" nav group and were
  reframed as the legacy/optional route.
- Super Doctor advice points at **`doctor.md`** (archived, prebuilt Doctors) rather than telling
  people to build one with meta-doctor — meta-doctor survives only as a carrier-mismatch fallback.
- Also refreshed: `timesync.md` (both bundles ship `ntpdate-sync`, so the manual XTerm fix-ntp script
  is now the pre-TLS-Updates fallback and the "no permanent fix on phones" claim is gone),
  `email.md` Gmail (App Password + the error-4010 / ECDSA fix from `mail-tls13` 1.3.2), and
  `browsers.md` (new Atlas section — TouchPad-only, the two mutually-exclusive patches).

## Help content server (`help.webosarchive.org`) — separate repo `~/Projects/help.palm.com`

`org.webosarchive.help-redirect` only points the Help app at `help.webosarchive.org`; the
*content* is mirrored palm.com help, served from that repo (committed + deployed by the user;
on Cloudflare). The mirror had absolute palm.com/dev URLs baked in. We rewrote them to the
serving host, **anchored to `//host` so subdomains aren't corrupted** (BSD macOS, batch xargs to
avoid arg limits):
```
grep -rIl --exclude-dir=.git '//<host>' . | tr '\n' '\0' \
  | xargs -0 -n 300 sed -i '' 's#//<host>#//help.webosarchive.org#g'
```
Rewrote (all fetched-resource hosts → `help.webosarchive.org`): `help.palm.com` (images/TOC/
articles), `ws-dev.help.palm.com` (article CSS, `/css/*`), `downloads.help.webosarchive.org`
(videos `/devicehelp/*.mp4`). **Preserved** (not re-hosted, not content): `stage-help.palm.com`
(chat), `dev-help.palm.com`, `kb.palm.com`, `www.palm.com`, `developer.palm.com`, and the bare
Omniture `s.prop30="help.palm.com"` analytics labels. The app fetches `.json` catalogs AND
article `.html`; `index.html`→`index.json` is auto-rewritten by the app, so mirror `.json` must
exist. Videos need byte-range support — Cloudflare returns `206`, good.

**Cache gotcha (cost us an hour):** after deploying a content fix, the Enyo Help app kept replaying
the OLD cached URLs (videos 404'd against stale hosts) even though the server was correct. webOS
caches Enyo apps + their fetched content hard. **A full reboot cleared it.** When verifying any
content change on-device, reboot (or at least relaunch→Luna restart) before concluding it's broken.
Consider short `Cache-Control` on the help `*.json` at Cloudflare while iterating.

## Conventions
- Namespaces are mixed: `com.palm.*` (stock), `org.webosinternals.*` (TLS chain), `com.nizovn.*`,
  and newer webOS Archive packages under **`org.webosarchive.*`** (preferred going forward;
  help-redirect and tls-updates use it). `org.webosce.*` is the held webOS CE line.
- Patch packages back up to `*.webosce-orig` and restore on prerm; keep them reversible.
- Commit messages end with `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`. Exclude
  `.DS_Store` and the untracked `.gitignore` from commits.
- A connected TouchPad's `LastUpdated` date base we've been stamping: `1782744397` (2026-06-29).
