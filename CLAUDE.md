# CLAUDE.md

Guidance + hard-won context for this repo. Read this first when resuming.

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

**Editing descriptions / metadata:** safe to hand-edit `ipkgs/Packages` directly, then regenerate
`Packages.gz` only (don't re-derive). If you also want the ipk's own control to match (so it's
self-consistent), pull the edited `Source`/`Description` back into the build control and rebuild
the ipk (changes md5 → re-stanza). Editing only the index keeps ipk md5s unchanged → smaller deploy.

**Validate before commit:** 1:1 between `ipkgs/*.ipk` and index `Filename`s; every index `MD5Sum`/
`Size` matches the actual file; all `Source` JSON parses; full dependency closure resolves.

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

## Package inventory (live feed = 23 packages)

- **nizovn stack** (hand-curated stanzas): cacert, glibc, openssl, qt5*, qtwebbrowser, qupzilla,
  squid (kept for old phones, min 1.3.5), + vlcplayer, dbus.
- **TLS 1.2/1.3 chain** (`org.webosinternals.*`, armv7): `browser-tls13`→`com.palm.rootcertsupdate`;
  `curl-tls13`, `luna-tls13`, `mail-tls13` → browser-tls13; `mojomail-imap-tagfix` → mail-tls13;
  `ntpdate-sync`. (These came with `Feed:"WebOS Internals"` in their control — we retagged the
  index `Source` to `WOSA Modernize`/`Modernize` so they show in the modernize group; icon
  `openssl_icon.png`.)
- **App Catalog:** `com.palm.app.findapps` (phones, Min 2.2.4/Max 2.9.9, icon hp-appcatalog),
  `com.palm.app.enyo-findapps` (TouchPad, Min 3.0.0). These were stock Palm-packaged ipks with no
  `Source` — we injected one.
- **`org.webosarchive.help-redirect`** (built + verified this session): patches `com.palm.app.help`
  `UrlManager.js` `helpUrl` (drives all content) + `HelpApp.js` palm.com domain check + device.do
  → `http://help.webosarchive.org`. Backs up `*.webosce-orig`, restores on removal, RestartLuna.
- **`org.webosarchive.tls-updates`** ("TLS 1.3 Updates") — payload-free **meta** package. Depends:
  rootcerts, browser/curl/luna/mail-tls13, mojomail-imap-tagfix, help-redirect, enyo-findapps.
  Excludes QupZilla + LunaCE. `Type: OS Application`, no icon, **webOS 3.0.X only**
  (Min 3.0.0/Max 3.0.9, TouchPad/Touchpad Go), RestartDevice. This is the recommended one-tap install.

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

## Conventions
- Namespaces are mixed: `com.palm.*` (stock), `org.webosinternals.*` (TLS chain), `com.nizovn.*`,
  and newer webOS Archive packages under **`org.webosarchive.*`** (preferred going forward;
  help-redirect and tls-updates use it). `org.webosce.*` is the held webOS CE line.
- Patch packages back up to `*.webosce-orig` and restore on prerm; keep them reversible.
- Commit messages end with `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`. Exclude
  `.DS_Store` and the untracked `.gitignore` from commits.
- A connected TouchPad's `LastUpdated` date base we've been stamping: `1782744397` (2026-06-29).
