# CLAUDE.md

Context for this repo. Read it first when resuming. Detail that used to live here (per-release
narratives, rejected-build forensics) is in git history: `git log -p CLAUDE.md`.

## What this is

**WOSA Modernize**, a Preware feed of `.ipk` packages that modernize HP webOS devices: TouchPad
(3.0.5), webOS CE 3.1.0 TouchPads, and the 2.2.4 phones (Pre3 / Veer / Pre2). One URL for every
device:

- **Live:** `http://stacks.webosarchive.org/feeds/modernize/ipkgs` (use **http** in anything a stock
  device must reach; its TLS 1.0 stack can't do modern https, and this feed delivers the fix).
  Preware: Manage Feeds → that URL, Compressed (gzip) **on**.
- **Git:** `git@github.com:webOSArchive/preware-modernize-feed`, branch `main`.
- `ipkgs/` = ipks + `Packages`/`Packages.gz`. `ipkgs/assets/{icons,screenshots}/` are hosted for
  `Source.Icon`/`Screenshots`. `staging/` = held packages, never served.
- Current size: **74 packages / 98 stanzas** (`grep -c ^Package: ipkgs/Packages`). For live versions
  and gates, read `ipkgs/Packages`, not this file.
- No build system (the old `package.sh`/`IpkgFeedGenerator.jar` tooling is not here). We hand-build
  ipks and hand-maintain the index.

## Open / TODO

- **`"Touchpad Go"` spelling.** The 11 nizovn stanzas list only lowercase `"Touchpad Go"`. If a Go's
  real `modelNameAscii` is `"TouchPad Go"`, the whole Qt5/QupZilla/VLC stack is silently invisible on
  it. btgamepad 1.2.0 lists both spellings and installed fine on a real Go, which doesn't say which one
  matched. Next time a Go is attached, capture `modelNameAscii` and
  `cat /etc/prefs/properties/machineName`, then fix the losing side (index-only).
- **Veer (broadway) / Pre2 (roadrunner):** phone packages ship but are untested on hardware. Likely
  failure is a wrong `modelNameAscii` (invisible in Preware; *Ignore Device Compat.* diagnoses it).
  Record `machineName` per device.
- **Synergy Part 2 connectors (20)** have never been installed on any device. Test one OAuth
  (Dropbox) and one IM connector on 3.0.5, then on CE 3.1.0 with no Part 1. Unconfirmed: which Atlas
  build has the `setWindowSize` mult=1 fix the OAuth webview needs (floored at 0.9.11; ask Herrie).
- **CE 3.1.0 coverage** is a policy call: most TouchPad stanzas are `Max 3.0.9`, so a 3.1.0 device sees
  ~38 packages and gets no one-tap meta. Either widen them to `3.9.9` or declare CE out of scope.
- **Not yet run from the feed on hardware:** `media-tls13`, `com.palm.app.videos`, `webos-chat` 1.0.0
  (legacy). (`webos-chat` 3.1.0 was tested on a CE tablet.)
- **Atlas:** remove the one-time `removeHandlersForAppId` note from both Atlas `FullDescription`s
  when 0.9.13 ships.
- **Preware 1** (`~/Projects/preware`): check for uncommitted/unpushed work before building.
  `Type=static` is in 1.9.21; Preware 2 2.1.3 enforces it too.
- **`~/Projects/webos-docs`:** had uncommitted changes (two-path layout collapsed). Commit when ready.
- **staging/:** `swupdate-redirect` needs an UpdateDaemon binary patch
  (`staging/swupdate-redirect-FINDINGS.md`). `luna-update` (webOS CE 3.1.0 / LunaCE swap) bricked a
  device combined with luna-tls13; the LunaCE swap must become its own standalone-tested package.
- Report to Herrie: no Synergy script has an OS/board guard, so WOSQI/`ipkg install` bypass every
  device check.

## Building ipks (macOS)

```bash
printf '2.0\n' > debian-binary
export COPYFILE_DISABLE=1   # no macOS ._ files
( cd control && tar --uid 0 --gid 0 --uname root --gname root -czf ../control.tar.gz . )
( cd data    && tar --uid 0 --gid 0 --uname root --gname root -czf ../data.tar.gz . )
ar rc out.ipk debian-binary control.tar.gz data.tar.gz [pmPostInstall.script pmPreRemove.script]
```
- `data.tar.gz` is rooted at `./usr/palm/applications/<id>/...`. The installer extracts under the
  offline root `/media/cryptofs/apps`, so never include `media/cryptofs/apps/`.
- `control.tar.gz` = `./control` (+ `./postinst`/`./prerm`, 0755).
- Many incoming ipks are **GNU-ar** (`/`-suffixed names, `//` longname table), which BSD `ar x` chokes
  on. Use a Python ar parser that handles `#1/N` (BSD), `/N`+`//` (GNU) and plain names.
- **Repackaging recipe** (metas, control-only fixes): reuse `debian-binary`, `data.tar.gz` and any
  `pm*.script` members **verbatim**, rewrite only `control.tar.gz` (Python `tarfile`, keep each
  `TarInfo`'s order/modes/owner), `ar rc` in the original member order. Then unpack-diff old vs new:
  only the intended member may differ.
- **Repackaging upstream ipks we don't version:** ship `<upstream>.1`, `.2`…; it sorts above theirs
  and below their next release.
- `palm-package` rejects 4-part appinfo versions, so palm-packaged apps can't use `.N`.
- Before adding ANY incoming ipk, unpack it and check: `sh -n` on the scripts; no `killall`/reboot,
  no blocking `luna-send` to LunaSysMgr services, no nested `ipkg`; root:root; no macOS strays or
  stray dev files (e.g. a `CLAUDE.md`); payload rooted correctly; URLs are `http`, not `https`
  (`grep -r 'https://.*webosarchive'`); and re-check the gate you inherit, not just version/md5/size.
  Hash every payload file against the previous release, because handover notes have understated changes
  more than once.

## Maintaining the index

- **The index is the authority Preware reads, and its `Source` is the curated copy.** It is routinely
  ahead of the ipk's control (titles, icons, Feed/Category retags, lead paragraphs). Keep existing
  stanzas verbatim and add/replace only what you touch. **Never regenerate a stanza from the ipk's
  control**: take `Version`/`Depends` from a new control and keep the old `Source`. Most ipks are kept
  as-delivered with a curated index `Source` (incoming `Feed:"WebOS Internals"`/`"webOS Archive"` is
  retagged).
- Field order: `Package, Version, Depends, [Suggests], Section, Architecture, MD5Sum, Size, Filename,
  Description, Maintainer, Source`. Then `Packages.gz = gzip(Packages, mtime=0)` and
  `cmp <(gunzip -c Packages.gz) Packages`.
- **`Source` JSON** (Preware ignores std fields for display): `Type` (`Application` / `OS Application`
  / `Linux Application` / `Patch` / `Theme`), `Feed` (**must be `"WOSA Modernize"`**, since Preware
  groups by it), `Category`, `Title`, `FullDescription` (HTML), `Icon`, `Screenshots`,
  `MinWebOSVersion`, `MaxWebOSVersion`, `DeviceCompatibility`, `LastUpdated` (unix secs; missing ⇒
  "Unknown"), `Post{Install,Update,Remove}Flags` (`RestartLuna`/`RestartDevice`). Preware does
  `.replace(/\\'/g,"'")` then `JSON.parse`; validate the same way.
- **Description/gate edits are index-only and need no version bump** (re-read on Update Feeds).
- **Leading space in `Title` sorts it to the top** (raw untrimmed compare, `packages.js:789`;
  verified on device). `" TLS 1.3 Updates (TouchPad)"`, `" TLS 1.3 Updates (Phones)"`,
  `" Synergy Revival Roll-up (Touchpad)"` rely on it. **Do not tidy those spaces away.**
- **Never put `Depends` or Min/Max in an ipk's control when they belong in the index.** Preware
  installs by local file, so ipkg resolves the *control's* `Depends` from the feed (ungated). Min/Max in
  a control also filters the installed list and can hide the app. Atlas, media-tls13, videos and all
  Synergy ipks have dep-free controls on purpose.
- Periodically sweep for `Source` drift where upstream controls moved on (qt5qpaplugins once had a
  stale changelog).

**Validate before commit:** 1:1 between `ipkgs/*.ipk` and index `Filename`s; every `MD5Sum`/`Size`
matches; all `Source` JSON parses; every meta's ipk control `Version`/`Depends` matches its stanza;
**no two stanzas share name+version+arch while pointing at different files** (same file is fine, see
two-stanza pattern); and a **per-device `loadPackage` simulation** (TouchPad 3.0.5, CE 3.1.0, Go with
both spellings, Pre3 2.2.4, Veer 2.2.4/2.2.0, Pre2 2.2.4/2.1.0) at **both `ignoreDevices` settings**,
asserting every visible package's deps (incl. version floors) are visible and resolve. Expected
asserts: Atlas/media-tls13 deps = tls-updates/browser-tls13 at 3.0.5 and none at 3.1.0; btgamepad
1.1.0 on TouchPad, 1.2.0 on Go; each Synergy connector's winner is the right stanza in all four
cases; `webos-chat` 1.0.0 everywhere ≤3.0.x and 3.1.0 on CE. Known pre-existing gap: `squid` on Pre2
2.1.0 with ignoreDevices ON.

## How Preware gates (verified in `~/Projects/preware` source, 1.9.x)

- **`MinWebOSVersion` is hard** (no override, `packages.js:464`). **`MaxWebOSVersion` and
  `DeviceCompatibility` are soft**: both are skipped when the *Ignore Device Compat.* pref is on, and
  install shows only a click-through warning. Never rely on `Max` alone to keep a package off a device.
- Both bounds inclusive (`Min == Max == "3.0.5"` admits exactly 3.0.5). Absent fields = wide open
  (`1.0.0`–`99.9.9`). Filters also apply to the *installed* list.
- `DeviceCompatibility` is an exact `include()` against `modelNameAscii`. Proven on hardware:
  `TouchPad`, `Pre3`, and one of `TouchPad Go`/`Touchpad Go`. Others in the wild: `Pre`, `Pre2`,
  `Pixi`, `Veer` (no spaces in `Pre2`/`Pre3`). Preware rewrites `modelNameAscii` to `"Pre2"` for
  `roadrunner`. **When unsure of a spelling, list every candidate**; extras cost nothing.
- **postinst-level hard gate = `/etc/prefs/properties/machineName`** (what ipkgservice reads,
  `luna_methods.c:561`). Boards: TouchPad `tenderloin`/`topaz`, Go `shortloin`/`opal`, Pre3
  `mantaray`, Veer `broadway`, Pre2 `roadrunner`. This is the only guard that covers WOSQI / direct
  `ipkg install`, which ignore all feed metadata.
- `Architecture` can't separate devices (all `armv7`). Don't invent arches: unknown ones are rejected.
- **Conventions:** fence the major line (`Max 2.9.9` / `3.9.9`). Phones `Min 2.2.4 / Max 2.9.9` +
  `["Pre3","Veer","Pre2"]`. Many TouchPad stanzas are `Max 3.0.9` (see CE TODO). Exceptions:
  `com.palm.app.backup` pinned `3.0.5`/`3.0.5` (ROM-app swap tested only there). Give Min floors that
  match deps' floors (a `Min 3.0.0` package with `Min 3.0.5` deps is unresolvable on 3.0.0–3.0.4).
- **Audit finding:** the TouchPad roll-up and its members have no `DeviceCompatibility` and almost no
  board guards. On 3.0.x the version fence is the only fence.

### Two stanzas, one package name

- **Same version, same file, different gate/Depends (Atlas, media-tls13, the 20 Synergy connectors):**
  Preware keeps the **first** stanza of a name when versions are equal (`infoUpdate` Replace Type 6;
  the merge is commented out). So put the **`Min 3.1.0` / dep-free stanza FIRST**, then the
  `Min 3.0.5`/`Max 3.0.9` stanza with the 3.0.x deps. On 3.0.5 the hard Min kills the first; on 3.1.0
  the soft Max kills the second, and with ignoreDevices ON, first-wins still picks the right one. Reverse
  the order and CE gets the 3.0.x deps forced on it. Safe from ipkg's dedupe trap because both stanzas
  name the same `Filename`+`MD5Sum`.
- **Different versions (btgamepad 1.1.0/1.2.0, webos-chat 1.0.0/3.1.0):** the newer one wins (Replace
  Type 1 / Type 5 = offered as an update), so order doesn't matter. Keep the legacy version below the
  newer one.

### Updates

- **Preware compares the `Version` string only.** Any content change needs a bump; a same-version
  rebuild is never offered. **Once anything might be on the server, never re-cut at the same version.**
  Verify with `curl` against the live `Packages.gz` (and HEAD the ipk) rather than assuming. (A few
  same-version re-cuts were made deliberately by the user when nobody had downloaded the build. Treat
  them as exceptions, not precedent.)
- **Unversioned `Depends` never drag an installed dep up.** A meta that should force a new member
  version needs a floor `(>= x)` **and** its own version bump **and** its ipk control rebuilt to match.
  Adding a new member also needs the bump. Preware won't recurse into already-satisfied deps, so put
  floors as **direct** edges (e.g. `qupzilla → qt5qpaplugins (>= 1.0.4)`).
- **Update path depends on `Type`.** `Patch`, `Theme`, `OS Application`, Kernel, Font, OS Daemon
  update via `IPKGService.replace` (prerm + remove, then install). Everything else (`Application`,
  `Linux Application`) installs over the old version, with **no prerm**. Check `Type` before reasoning
  about what an update runs.
- **Restart flags:** Preware collects flags across a batch and applies the strongest **once at the
  end**. WOSQI ignores them and doesn't resolve `Depends`.
- **"No update showing":** check the live `Packages.gz` decompresses to the right version (stale `.gz`
  happens). Then the device: Update Feeds → remove/re-add feed → reboot. webOS caches hard.

## Install-script safety rules (each one has broken a device here)

1. **Never restart LunaSysMgr or reboot from a postinst/prerm.** Preware runs under LunaSysMgr; a
   restart mid-batch aborts the rest of the chain. Use the `Post*Flags` instead.
2. **Never make a blocking `luna-send` to a service LunaSysMgr provides** (`com.palm.applicationManager`,
   `com.palm.appinstaller`, listed in `/usr/share/ls2/roles/prv/com.palm.luna.json`) from a postinst
   **or prerm**. The App Installer runs these scripts synchronously inside LunaSysMgr, so it deadlocks
   with no error, and the UI freezes until a hard reboot. **This includes a plain `control.tar.gz`
   prerm**: ipkgservice copies it to `/media/cryptofs/apps/.scripts/<pkg>/pmPreRemove.script` at
   install, and `appinstaller/remove` runs that copy. (The postinst isn't promoted; it runs in
   ipkgservice's own process.) Background it: `( luna-send … & ) >/dev/null 2>&1`. Separate daemons
   (`com.palm.configurator`, `com.palm.db`, JS services) are fine blocking.
   - **Sticky:** a device stores the prerm at install time, so a bad prerm runs on the next
     remove/update even after the feed is fixed. Repair on-device first. Synergy repair:
     `sed "s|^luna-send -n 1 luna://com.palm.applicationManager/rescan .*|( & \& ) >/dev/null 2>\&1|"`
     over `/media/cryptofs/apps/.scripts/*/pm{PreRemove,PostInstall}.script`.
3. **Never `cp` over a file the running system maps or executes** (libs, fonts, running binaries):
   it causes SIGBUS/ETXTBSY. Write a same-dir temp and `mv`.
4. **Never call `ipkg` from a script.** It runs inside Preware's own `ipkg install`.
5. **Don't poison the restore point.** Identify "already ours" by **content** (e.g. `grep -q ssl11dl`,
   a marker string), not by an md5 of the current build, or a previous version's file is saved as
   "stock" and prerm restores a broken file. On upgrade, don't re-create a backup over a good one. No
   backup is safer than a wrong one. Refuse to save an empty-directory backup.
6. **`cd /` first**: the App Installer runs scripts with cwd = the app's dir, which a whole-dir
   replacement deletes.
7. **Process sweeps must skip `$$` and the installer** (`pmPostInstall|pmPreRemove|
   ApplicationInstallerUtility|/usr/lib/ipkg/|\.ipk`) or they SIGTERM their own install.
8. **Guard for WOSQI/ipkg:** put an OS-version/board check at the top of postinst, before any edit.
   The payload is inert; the wiring is what breaks things. prerm must tolerate being called twice
   (WOSQI does), so clear markers after a successful restore.
9. `/media` is on the read-only rootfs: `mkdir` there needs rw. Remount only if needed and put it back.
10. Never mount anything under `/media/internal`. Internalz Pro lists that path with mtools, which is
    blind to mounts, so it shows an empty folder and writes land on internal storage.

## Package notes (gates/versions: see `ipkgs/Packages`)

**TLS chain, TouchPad** (`org.webosinternals.*`, upstream
`github.com/webOSArchive/OpenSSL-legacyWebOS`, local `~/Projects/OpenSSL-legacyWebOS`):
- `browser-tls13` provides `/usr/lib/ssl11` → rootcertsupdate. `luna-tls13` edits
  `/etc/event.d/LunaSysMgr` to force ssl11, plus wrappers that scrub that env from children
  (`media-pipeline`, `setcpushares-pdk`, `setcpushares-task`). 1.1.4 is cosmetic on TouchPad (parity
  with phone). `mail-tls13` 1.3.2 pins TLS 1.2 + RSA cert for Gmail (ECDSA false "not trusted"/4010).
  `downloadmgr-tls13` 1.1.0 RPATHs LunaDownloadMgr onto ssl11 curl, with a 1-byte patch for a
  `curl_multi_remove_handle` SIGSEGV (1.0.0 is broken; phones don't crash, so `-phone` stays 1.0.0).
  Remove before browser-tls13. `mojomail-imap-tagfix`, `ntpdate-sync` are standalone.
- `curl-tls13` and `mail-tls13` have **empty `Depends`** (they can't name both `browser-tls13` and
  `-phone`). Their postinsts refuse without `/usr/lib/ssl11`; the metas give the order.
- ⚠️ **Never sync `curl-tls13` from upstream.** Upstream's same-version 1.0.1 has different OpenSSL
  binaries, and it replaces `/usr/bin/curl` that Synergy depends on.
- `mojomail`/`ntpdate` were re-synced at the same version on purpose (control-only change, no
  re-download for existing users).

**`org.webosarchive.tls-updates`** (TouchPad one-tap meta, `OS Application`, `RestartDevice`):
README-only payload + a postinst that defuses a still-installed retired `accountsapp`. Members: rootcerts,
browser, ntpdate (position is its only ordering), curl, luna, mail, downloadmgr, media-tls13,
mojomail-tagfix, help-redirect, enyo-findapps, usbsettings, btgamepad, `com.palm.app.accounts`,
`com.palm.app.backup`, with floors on the revved ones. Deliberately excludes QupZilla, Atlas (103MB),
atlas patches, VLC, LunaCE. Rebuild with the repackaging recipe; bump on any Depends change.

**Phones** — `*-phone` packages built by `./build-ipks.sh phone` in OpenSSL-legacyWebOS: one ipk
covers all three boards, and the postinst picks per-board content from `machineName` (fallback
`/etc/palm-build-info`) and **exits non-zero on an unknown board** (incl. TouchPad). Distinct names
are required because ipkg dedupes on name+version+arch across all feeds and Preware installs by name.
Ruled out: subdir `Filename` (ipkg cache-path bug), per-board arch, `Provides:` (Preware ignores it).
`luna-tls13-phone` 1.1.4 adds static `jailer`/`setcpushares-task` env-scrub wrappers (without them,
PDK apps and Preware/WOSQI installs died at exec from inherited `LD_BIND_NOW`).
`tls-updates-phone` 1.0.1 floors `luna-tls13-phone (>= 1.1.4)`. Build host needs GNU ar + patchelf. ⚠️
**Don't prune `ipks/{topaz,mantaray,roadrunner,broadway}/` upstream**: `phone` rebuilds reuse them.

**Atlas** (`org.webosports.app.atlas`, WPE WebKit browser, 103MB, upstream WebOS Ports):
two-stanza pattern (3.0.x stanza depends `tls-updates`). We repackage **only `control.tar.gz`** to add
`Source`; 5 ar members kept. **Never add `Depends` to its control.** Flags: `RestartDevice`
install/update, `RestartLuna` remove (set in both stanzas and the control). Upstream builds are
`PKG_TARGET=feed` (no `killall`); still grep. 0.9.9 was reverted (engine bug). A 0.9.12 build with
blocking `removeHandlersForAppId` luna-sends and a half-done Chromium backend was rejected
(`6b4478e6…`); don't resurrect it.
- `atlas-default-browser`: redirect stub (awk-swaps the `enyo.create({kind:"BrowserApp"})` line, JS
  embedded via heredoc in postinst, not a payload file) + hides the stock icon. Backup `.webosce-orig`.
- `open-in-atlas`: alternative that keeps the stock browser and adds an App Menu item. Patches
  `enyo.BrowserApp.prototype.kindComponents` (Enyo 1 deletes `.components`), un-escapes `&amp;`.
  Mutually exclusive with the redirect patch at runtime (Preware ignores `Conflicts:`). Test Enyo
  patches in node with an `enyo.kind` stub and dry-run scripts against a copy before device.

**Media/Video:** `org.webosarchive.media-tls13` replaces GStreamer `souphttpsrc` with a libcurl element
that `dlopen`s ssl11 curl. It must drop the GStreamer registry cache, and it's in the roll-up.
`com.palm.app.videos` 3.2.0 (CE-only, experimental, depends media-tls13). Only build 3
(`e3db23b5…`) is safe: builds 1–2 have the deadlocking prerm, and devices with them need
`.scripts/com.palm.app.videos/pmPreRemove.script` repaired first.

**Synergy Revival** (Herrie's account aggregation; sources in `~/Downloads/synergy_revival_0.9.3_*`;
fixes upstream in `Herrie82/webos-synergy-revival`, `~/Projects/webos-core-apps`):
- Part 1 (13 pkgs, 3.0.5 only, built into CE): `com.palm.synergy.generic` runtime + whole-dir
  replacements of stock apps/services. Part 2: 20 connectors, two-stanza pattern, all depend on
  `atlas (>= 0.9.11)` (OAuth renders through BrowserServer-atlas), plus `synergy-revival` on 3.0.x.
  `org.webosarchive.synergy-revival` = our meta (tls-updates + atlas + Part 1, floored).
- All controls bare (no Depends/Source); deps are index-only. Category `Synergy`. `RestartDevice` for
  generic/connectors/meta, `RestartLuna` for app replacements. No `DeviceCompatibility`.
- Each of Herrie's repos has its **own copy** of the shared `packaging/lib/{postinst,prerm}`. Fixes
  never propagate, so re-check every tree. Bugs found and fixed upstream: cwd deletion, prerm never
  restoring, self-SIGTERM, blocking rescan deadlock, SIGBUS from `cp` over `libWebKitLuna.so` (the
  `umount` was a red herring), and `luna-systemui` (surgical mode) prerm deleting the whole system UI
  dir. Pre-repair for that on a device with the old prerm:
  ```sh
  mkdir -p /media/cryptofs/luna-systemui-overwrite/luna-systemui
  echo /usr/lib/luna/system/luna-systemui > /media/cryptofs/luna-systemui-overwrite/luna-systemui/dest.txt
  echo app/FilePicker/AlbumGridView.js   > /media/cryptofs/luna-systemui-overwrite/luna-systemui/files.txt
  ```
- Missing core app recovery: uninstall our package, **reboot**. The boot-time `app-install` reinstalls
  stock from `/usr/palm/ipkgs/<id>/`.
- `org.webosarchive.accountsapp` is **retired** and superseded by `com.palm.app.accounts` 3.1.1. Both
  metas' postinsts adopt the genuine stock tar and re-point accountsapp's backup. **They must not delete
  that backup** (its prerm would then delete the Accounts app) and must not call ipkg.
- Part 1 + meta are hardware-verified on 3.0.5 (clean and previously-broken devices).

**Other packages:**
- `btgamepad`: 1.1.0 (all TouchPads, in roll-up) + 1.2.0 (Go-only, device list lists both spellings;
  postinst jail rewrite gated on `opal|shortloin`; replaces 1.1.0's `/dev/input` bind mount with jailer
  `mknod`). Hardware-verified on a Go. `-go` in the filename is cosmetic.
- `com.webosarchive.usbsettings` 1.1.9: OTG/high-power/storage control, self-contained,
  `RestartDevice` (new ls2 registration), `removable:false` (launcher delete skips prerm). Mounts USB
  at `/media/usb` (see rule 10).
- `com.palm.app.backup` 3.1.1: local unencrypted Backup/Restore in `/media/internal/webos-backups`. A
  higher version doesn't beat a ROM app, so the postinst moves the ROM copy aside (content-checked, 3.0.5
  only). 3.1.0 restores partially; ship only `b8f2a57c…`, since the pulled 3.1.1 `31af1c80…`/`c9e2e2e0…`
  filled storage. Known weakness: the orphan sweep is skipped if stage cleanup fails.
- `help-redirect`: points Help at `help.webosarchive.org` (content repo `~/Projects/help.palm.com`;
  rewrite absolute hosts anchored on `//host`; reboot device to clear Enyo caches when verifying).
- App Catalog: `enyo-findapps` (TouchPad, 6.2.2929; installs via `installNoVerify`/.ipk handler since
  HP signing is dead) and `findapps` (phones, 3.1.23302). Both titled "webOS App Catalog (…)". Don't
  resurrect same-version earlier builds (https URLs / Mojo `onChoice` instead of `onChoose`).
- `org.webosarchive.v8fix` 1.0.2: adds `--always_full_compiler` to `/etc/palm/browser.conf` on webOS
  2.0.x–2.1.x (V8 miscompile breaks Enyo 2). Postinst guards on version and a node probe;
  `RestartDevice`; no DeviceCompatibility. The Palm appinstaller can't remove it; Preware can. Source:
  `github.com/alan-morford/enyofix`.
- `org.webosarchive.webos-chat`: 1.0.0 (legacy ≤3.0.9) adds a marked `/etc/hosts` line pointing AOL
  login hosts at `173.230.142.249` (baked in; if the server moves, ship 1.0.1). 3.1.x (CE) installs
  libaim + account/service; prerm only spawns a detached cleanup (`start-stop-daemon -b -x <script>`,
  not `/bin/sh`). 3.1.1 also swaps in a one-line fix to the chatthreader's `newmessageassistant.js`.
  The bug: a message from a sender who isn't a linked buddy was never filed into a thread. The swap
  happens only if the installed file has md5 `37c050bc…` (Synergy chatthreader 1.1.0), and prerm
  restores the original. The same buggy file ships in our 3.0.5 `com.palm.messaging.chatthreader
  1.1.0.2`, so report it upstream.
- nizovn stack (QupZilla/Qt5/VLC/squid/dbus): hand-curated stanzas, ipks as-delivered. Floors:
  `qupzilla → qt5sdk (>= 1.0.3), qt5qpaplugins (>= 1.0.4)`, `qt5sdk → qt5 (>= 5.9.7-0)`. qt5sdk 1.0.3
  fixes upgrade-over-self and drops stale `/var/palm/jail/<id>` jails for qt5 apps.

## Preware 1 & 2 (the feed ships both)

- **Preware 1** `org.webosinternals.preware` 1.9.21 from `~/Projects/preware` (`webOSArchive/preware`).
  Its postinst writes the feed confs incl. `modernize.conf` and **deletes the App Museum feed** (never
  re-add it). CE preloads 1.9.19, so ours must be > that. Build: `./build.sh arm` (delete the copied
  `source/bin/org.webosinternals.ipkgservice` after), then repack: `control.tar.gz` = curated control
  + `postinst` + `prerm` from `source/control/`; members `debian-binary, control.tar.gz, data.tar.gz,
  pmPostInstall.script, pmPreRemove.script`. Drop build.sh's signature files.
- **Preware 2** `com.palm.app.preware2` 2.1.3 (Enyo 2, WebOS Ports). Clone `~/Projects/webos-ports-preware`,
  `origin` = user's fork `codepoet80/preware2`, work on its default branch (**no feature branches**).
  PRs go to **webOS-ports/preware**, never a contributor's fork. Build with `build-legacy.sh`.
  Nothing may depend on it; not in roll-ups.
- ⚠️ **`org.webosinternals.ipkgservice` must be byte-identical in both apps** (currently `32611fac…`).
  Preware 2 reinstalls a differing service at every boot and `killall -9`s it during its own install.
  Change both in the same release or not at all. The dbus file uses `Type=static`.

## Deploy

`git push` **and** `rsync -av --delete ipkgs/ <server>:.../feeds/modernize/ipkgs/`. Then Preware
"Update Feeds". `stacks` is Cloudflare but `Packages.gz` isn't edge-cached.

## Driving a device (novacom)

- This `novacom` eats dash-flags meant for the remote command. Pipe a script instead:
  `novacom -t open tty:// < script.sh` (end with `exit`, filter `^root@`). `novacom run file://bin/cat
  /path` works (no dashes). `novacom put file:///dev/path < local` to transfer.
- `ipkg -o /media/cryptofs/apps install x.ipk` **defers the postinst**. Run
  `sh /media/cryptofs/apps/usr/lib/ipkg/info/<pkg>.postinst` by hand. Preware/WOSQI run it.
- VMware grabs the USB device; connect it to the host. Check device health (`hostname`, version,
  `ps | grep LunaSysMgr`) before patching. Several test devices look alike.
- The user tests on hardware directly. Their "it worked" is a hardware verification.

## Bricks to remember

- **luna-update** (staging): swapped LunaSysMgr for LunaCE (built vs OpenSSL 0.9.8) while luna-tls13's
  launcher forced ssl11 → crash loop. Never bundle a LunaSysMgr binary swap with the launcher edit.
  Recovery (novacomd survives): restore `/usr/bin/LunaSysMgr.webosce-orig` and
  `/var/luna/LunaSysMgr.tls13-orig` → `/etc/event.d/LunaSysMgr`, reboot.
- A 2.2.4 phone was bricked by TouchPad TLS members that had no gate. Gate everything
  device-specific, and add postinst guards.

## OTA (held)

`com.palm.app.updates` → `/usr/bin/UpdateDaemon` (OMA-DM). The URL in `DmTree.xml` isn't enough: the
host comes from carrier → domain, which is empty on Wi-Fi TouchPads, so the check aborts. Needs an
UpdateDaemon binary patch. See `staging/swupdate-redirect-FINDINGS.md`; the user maintains
`webos-update-exploration`.

## User-facing docs

`~/Projects/webos-docs` (MkDocs, `mkdocs build --strict`; one INFO about `macos-install.md` is
expected). The dividing line is **webOS 2.2.4** (TLS Updates) vs older (HTTP only). Keep
`setup-path.md` (anchors are linked from five pages).

## Conventions

- Namespaces: `com.palm.*` stock, `org.webosinternals.*` TLS chain, `com.nizovn.*`, new packages
  `org.webosarchive.*`, `org.webosce.*` held CE line. `com.webosarchive.usbsettings` can't be renamed.
- Patches back up to `*.webosce-orig` (or a package-specific suffix) and restore on prerm.
- Commits end with the current Co-Authored-By line. Exclude `.DS_Store` and the untracked
  `.gitignore`.
