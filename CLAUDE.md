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
- **HP Pre 3 (mantaray): VERIFIED ON HARDWARE.** `org.webosarchive.tls-updates-phone` **1.0.0** →
  the `*-phone` package family. This confirms the two values that were previously unverified guesses:
  `DeviceCompatibility: "Pre3"` matches the real `modelNameAscii` (wrong string ⇒ the packages would
  have been invisible in Preware), and the postinst board detection resolves a Pre 3 correctly
  (wrong ⇒ it would have refused to patch). See "Phone support" below.
- **HP Veer (broadway) and Palm Pre 2 (roadrunner): in the feed, NOT yet tested on hardware.** Same
  merged packages, same detection path, different `case` arm — the plausible failure is a wrong
  `modelNameAscii` (`"Veer"` / `"Pre2"`) making them invisible in Preware, which the
  Preferences → *Ignore Device Compat.* toggle diagnoses in one step.

**Atlas (modern browser) + "make Atlas default" patch, both in the feed:**
`org.webosports.app.atlas` (**0.9.7**, WPE WebKit 2.52 browser, 103MB) and
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

Latest revs (deployed): `browser-tls13 1.1.2`, `luna-tls13 1.1.3` (1.1.0 was faulty — media wedged;
1.1.1 added the media-pipeline wrapper; 1.1.2 added a `/usr/sbin/setcpushares-pdk` wrapper so
PDK apps — QupZilla, nizovn Qt5 — launch normally, incl. under LunaCE; **1.1.3** current),
`mail-tls13 1.3.2` (**1.3.2** fixes Gmail/ECDSA-cert IMAP/POP sign-in that falsely failed with
"certificate is not trusted"/error 4010 — the imap/pop/smtp launchers now pin TLS 1.2 + an RSA
server cert), `downloadmgr-tls13 1.1.0` (RPATHs the system Download Manager /
`/usr/bin/LunaDownloadMgr` onto the ssl11 curl so background downloads AND uploads reach modern
HTTPS; depends browser-tls13. **1.1.0** adds the one-byte code patch that stops the SIGSEGV in
`curl_multi_remove_handle`, and repairs a poisoned uninstall restore point — see the package
bullet), `usbsettings 1.1.0` (**new** — auto-arms the OTG controller the first
time the port goes idle, fixing a hard hang some devices hit on their first-ever OTG cable insert;
adds a live connected-device list with device-type icons via a new `usbdevmon` helper binary, and a
USB Reset button to recover a wedged connection without a full reboot),
`com.palm.app.backup 3.1.0` (**new** — see below), `tls-updates 1.0.15`
(version-floors browser `>= 1.1.2` / usbsettings `>= 1.1.8` / btgamepad `>= 1.1.0` / luna
`>= 1.1.3` / mail `>= 1.3.2` / **downloadmgr `>= 1.1.0`**; also pulls in `ntpdate-sync`,
`downloadmgr-tls13`, `accountsapp` and — **new in 1.0.15** — `com.palm.app.backup` — apps break
when the clock is wrong, TLS cert validity checks fail).

QupZilla/Qt5 chain now carries version floors so installing QupZilla drags the Qt5 stack up:
`qupzilla → qt5sdk (>= 1.0.2), qt5qpaplugins (>= 1.0.4)` (the qt5qpaplugins floor is a **direct**
edge, added because Preware won't recurse into the already-installed qt5/qt5sdk nodes to notice
qt5's own floor); `qt5sdk → qt5 (>= 5.9.7-0)`; `qt5 → qt5qpaplugins (>= 1.0.4)`. This reverses the
earlier "leave the depends tree alone" stance for this chain (index-only edit; ipks kept as-delivered).

**Open / TODO:**
- **CE 3.1.0 gate drift** — TouchPad stanzas sit at `Max 3.0.9` while the three Atlas ones sit at
  `3.9.9`, so on a `3.1.0` device Atlas is visible and its `tls-updates` dep is not. See the
  ⚠️ paragraph under the per-device visibility table.
- **`accountsapp` floor drift (pre-existing, found 2026-08-23, NOT introduced by the backup work)** —
  `org.webosarchive.accountsapp` is `Min 3.0.0` but its two deps (`browser-tls13`, `luna-tls13`) are
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
- `MinWebOSVersion`/`MaxWebOSVersion` → **hard filter, no user override.** `models/packages.js:450-461`
  drops the package inside `loadPackage`, so it never enters the model: no group, no list, no search,
  no install. Both bounds **inclusive** (`versionNewer`, `packages.js:879`, returns false on equal).
  Only applies if `platformVersion` matches `/^[0-9:.-]+$/` (true on real devices).
- `DeviceCompatibility` → **soft filter.** Same drop (`packages.js:465`) but gated on
  `!prefs.get().ignoreDevices` — Preware Preferences has an "ignore devices" toggle. With it on the
  package returns and install only shows a click-through "Incompatible Device" warning
  (`pkg-view-assistant.js:416`). So Min/Max is the lock; DeviceCompatibility is the deterrent. Set both.
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

## Package inventory (single feed = 36 packages; phone packages detailed below)

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
  `curl-tls13` (1.0.1), `luna-tls13` (**1.1.3**), `mail-tls13` (**1.3.2**), `downloadmgr-tls13`
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
    launches, so PDK apps (QupZilla, nizovn Qt5) launch normally, incl. under LunaCE → **1.1.3**
    (current). Its FullDescription is synced in BOTH the index stanza and the ipk's own control
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
- **`org.webosarchive.tls-updates`** ("TLS 1.3 Updates (TouchPad)", **1.0.15**) — payload-free **meta**
  package. Depends: rootcerts, browser-tls13, ntpdate-sync, curl/luna/mail-tls13,
  **downloadmgr-tls13 (>= 1.1.0)**, mojomail-imap-tagfix, help-redirect, enyo-findapps,
  **usbsettings (>= 1.1.8)**, **btgamepad (>= 1.1.0)**, **accountsapp** (unversioned — new in 1.0.13),
  **`com.palm.app.backup`** (unversioned — new in 1.0.15). `ntpdate-sync` sits **after** browser-tls13
  and **before** luna — added because apps break when the clock is wrong (TLS cert validity windows
  fail); left unversioned (new dep, nothing to drag up). ⚠️ The old rationale "it's browser's own dep,
  so browser installs first regardless" is **no longer true** — upstream dropped ntpdate-sync's
  `Depends: browser-tls13` (it never needed it), so its position in this list is now the *only* thing
  ordering it. Harmless either way: it just drops in an upstart job and needs no ssl11. `downloadmgr-tls13` is ordered **after** luna-tls13 (per request; it also
  hard-depends browser-tls13 so browser installs first regardless); unversioned (new dep).
  Carries **version floors** on the packages that get revved: `browser-tls13 (>= 1.1.2)`,
  `luna-tls13 (>= 1.1.3)`, `mail-tls13 (>= 1.3.2)`, `usbsettings (>= 1.1.8)`,
  `btgamepad (>= 1.1.0)`, `downloadmgr-tls13 (>= 1.1.0)` (rest unversioned).
  Bumping a floor requires bumping tls-updates' own version too (1.0.6→1.0.7 for usbsettings,
  1.0.7→1.0.8 for btgamepad, 1.0.8→1.0.9 to drag usbsettings to 1.0.8, 1.0.9→1.0.10 to drag
  usbsettings to 1.1.0, **1.0.13→1.0.14 to drag downloadmgr to 1.1.0**) AND rebuilding
  the ipk so its control Depends match the index — else on-device opkg won't pull the new deps.
  Adding a plain (unversioned) new member is the same story minus the floor: **1.0.12→1.0.13**
  added `org.webosarchive.accountsapp` to Depends and still required the version bump + control
  rebuild, since Preware only offers an update when the Version string itself changes; **1.0.14→1.0.15**
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
- **`org.webosports.app.atlas`** ("Atlas", **0.9.7**, WPE WebKit 2.52 browser, **103MB**, arch `all`)
  — the modern browser. **We repackaged the upstream WebOS Ports ipk** (index+control curated;
  `data.tar.gz` kept byte-identical — only control.tar.gz rebuilt): (1) added `Depends:
  org.webosarchive.tls-updates` to its **control** (Atlas links `/usr/lib/ssl11` OpenSSL 1.1 for
  HTTPS; the dep pulls the whole TLS stack), (2) added a Preware `Source` block (Feed "WOSA
  Modernize", Category Browser, icon `atlas.png` extracted from the app), (3) **removed the
  `killall LunaSysMgr` from BOTH postinst and prerm** (see the reboot-fix lesson below) — the reload
  is now deferred to Preware's `PostInstallFlags`/`PostRemoveFlags` (RestartLuna). Its postinst lays
  the WPE engine (runs in place on cryptofs deviceroot), copies the device Adreno driver to the
  versioned GPU sonames, symlinks `/var/atlas252 → deviceroot/wpe-252`, installs the NPAPI adapter
  (`/usr/lib/BrowserPlugins/BrowserAdapterAtlas.so`), upstart jobs (`/etc/event.d/atlas` +
  `atlas-sensord`), db8 kinds (`org.webosports.logins`/`autofill`), and ls2 roles
  (`/usr/share/ls2/roles/{prv,pub}/org.webosports.browserserver.json`). **Verified on device.**
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

- **`com.webosarchive.usbsettings`** ("USB Settings", **1.1.0**, arch `all`, TouchPad only) — Enyo app
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
  - **It is a member of the TouchPad `tls-updates` meta** (community request) even though it is not
    TLS-related — "part of making the device more modern". Added as
    `com.webosarchive.usbsettings (>= 1.1.0)` at the END of the Depends list; it is self-contained and
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

- **`com.palm.app.backup`** ("Backup and Restore", **3.1.0**, arch `all`, TouchPad only) — a working
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
  - **Member of the TouchPad `tls-updates` roll-up** (unversioned, at the END of Depends —
    self-contained and order-independent), which is what took the meta to **1.0.15**.
- **`org.webosarchive.accountsapp`** ("Accounts (Community Build)", **3.1.0**, arch `all`,
  delivered pre-built by the user as `org.webosarchive.accountsapp_3.1.0_all.ipk`) — replaces the
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
| `org.webosinternals.luna-tls13-phone` **1.1.3** | none — the webOS 2.x build is payload-free and byte-identical across all three boards |
| `org.webosinternals.mojomail-imap-tagfix-phone` **1.0.0** | none — postinst only (per-board byte offset + 2 md5s) |
| `org.webosarchive.tls-updates-phone` **1.0.0** | payload-free one-tap meta (hand-built in THIS repo, like `tls-updates`) |

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

**After ANY change to gates, names or versions, re-run both checks** (they have each caught real
bugs): (1) **no two stanzas share name+version+arch** — the ipkg dedupe trap; (2) **simulate
`loadPackage` per device** (TouchPad 3.0.5 / CE 3.1.0 / Pre3 2.2.4 / Veer 2.2.4+2.2.0 / Pre2
2.2.4+2.1.0) and assert every visible package's deps are also visible. That second check is what
found `curl-tls13`'s and `mail-tls13`'s stale deps. The visibility check also verifies **version
floors** resolve against what the feed actually ships, not just that the dep is visible. Current
result — 36 stanzas, all valid:

```
TouchPad 3.0.5    30 visible  one-tap: tls-updates        deps OK
TouchPad CE 3.1.0 15 visible  one-tap: (none)             1 pre-existing dep gap (below)
Pre3 2.2.4        20 visible  one-tap: tls-updates-phone  deps OK
Veer 2.2.4        11 visible  one-tap: tls-updates-phone  deps OK
Pre2 2.2.4        11 visible  one-tap: tls-updates-phone  deps OK
Veer 2.2.0         3 visible  one-tap: (none)             deps OK
Pre2 2.1.0         3 visible  one-tap: (none)             deps OK
```

⚠️ **The CE 3.1.0 row has regressed since this table was first written, and the cause is a gate
drift CLAUDE.md still describes the other way round.** The "use `Max 3.9.9`, not `3.0.9`" convention
recorded above is **no longer what the feed does**: every TouchPad stanza except the three Atlas ones
now carries `Max 3.0.9`, so a device reporting `3.1.0` (i.e. `staging/org.webosce.luna-update`,
"webOS CE 3.1.0") sees 15 packages instead of 29 and gets **no one-tap meta at all**. The concrete
breakage the check catches: **Atlas (Max 3.9.9) stays visible but its `Depends:
org.webosarchive.tls-updates` (Max 3.0.9) does not**, so Atlas is uninstallable there. This predates
the downloadmgr/backup work (verified against `git show HEAD:ipkgs/Packages`) and is left alone
because it's a gating-policy decision, not a packaging one: either move the TouchPad stanzas back to
`3.9.9`, or accept that CE 3.1.0 is out of scope and drop Atlas to `3.0.9` for consistency. Pick one
and fix the convention paragraph above to match.

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
