<img src="https://raw.githubusercontent.com/webos-internals/preware/refs/heads/master/icon.png">

Packaging is done by Claude, old dead tools were too hard to use...

# WOSA Modernize feed

A Preware feed that brings legacy webOS devices back onto the modern internet: TLS 1.2/1.3 for the
browser, apps, mail, curl and the download manager, plus current root certificates, working clock
sync, a live Help app, the App Catalog, a modern browser, and some hardware extras.

## How to install

Add a feed in Preware using url: http://stacks.webosarchive.org/feeds/modernize/ipkgs/

Set **Compressed (gzip)** to on, then **Update Feeds**.

That is the same one URL for every supported device — Preware only shows you the packages built for
the device you are holding.

## Then install the one-tap bundle for your device

| Device | Install |
|---|---|
| HP TouchPad (webOS 3.0.x) | **TLS 1.3 Updates (TouchPad)** |
| HP Pre 3 / HP Veer / Palm Pre 2 (webOS 2.2.4) | **TLS 1.3 Updates (Phones)** |

Either one pulls in everything it needs, in the right order, and reboots the device once at the end.
Only one of the two will be visible to you, so there is nothing to choose between.

Supported and tested: **HP TouchPad** and **HP Pre 3**. The **HP Veer** and **Palm Pre 2** packages
are built and published but have not been tested on hardware yet — reports welcome.

webOS 2.2.4 is required on the phones. An un-upgraded Veer (2.2.0) or Pre 2 (2.1.0) is not supported,
and the packages are hidden on those so they cannot be installed by mistake.

## Also in the feed (install separately if you want them)

- **Atlas** — a modern WPE WebKit browser, plus a patch to make it the default, or a gentler
  "Open in Atlas" item in the stock browser's menu. Pick one of the two patches, not both.
- **Synergy Revival (experimental)** — brings account-based messaging and cloud storage back to the
  tablet: WhatsApp, Telegram, Signal, Teams, Discord, Google Chat, Facebook, Dropbox, Google Drive,
  OneDrive, Box, MEGA, pCloud, Koofr, kDrive, HiDrive, Yandex Disk, S3, Flickr and CalDAV/CardDAV, as
  real accounts under Settings > Accounts. Install **Synergy Revival** first, then pick the connectors
  you want. It is a large download (roughly 130 MB before any connectors, since it pulls TLS 1.3
  Updates and Atlas), and it replaces the Accounts, Contacts, Messaging and Phone apps with updated
  builds. webOS CE 3.1.0 already includes the runtime, so CE users install connectors directly.
  If you already have **Accounts (Community Build)** installed, uninstall it first — it has been
  replaced by the newer Accounts app that comes with this.
- **QupZilla** and the nizovn **Qt5** stack, **VLC Player**.
- **USB Settings** (USB host/OTG, high-power devices, USB storage) and **Bluetooth Gamepad Support** —
  both already included in the TouchPad bundle above.

## Notes

- Install and uninstall through **Preware** or **WebOS Quick Install**. Deleting an app from the
  launcher skips its cleanup script and can leave a background daemon behind.
- Use the **http** URL, not https. A stock device's 2011 TLS stack cannot reach a modern https
  server — and this feed is what fixes that.
