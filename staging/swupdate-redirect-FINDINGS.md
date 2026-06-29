# webOS OTA Software-Update redirect — findings

Status: **held out of the live feed.** The `org.webosce.swupdate-redirect` package
correctly repoints the OMA-DM server URL, but that is **not sufficient** to drive
OTA updates to a custom server. Documenting here so we can tackle it later.

Tested on-device (HP TouchPad, webOS 3.0.5, Wi-Fi) 2026-06-29.

## What the package does (and that part works)
`UpdateDaemon` (`palm://com.palm.update/`, OMA-DM / SyncML 1.2, WBXML, HMAC-MD5)
reads the update server from an OMA-DM management tree. The URL lives, identically,
in three files (node AppAddr→SrvAddr→Addr, plus a `PortNbr`):
- `/usr/share/omadm/DmTree.xml` (template)
- `/var/lib/software/DmTree.xml` (runtime copy actually used)
- `/var/lib/software/DmTree.backup.xml`

The package's postinst backs these up (`*.webosce-orig`) and rewrites the URL
`https://ps.palmws.com/palmcsext/swupdateserver` (443) →
`http://www.webosarchive.org/swupdateserver` (80), then restarts `UpdateDaemon`.
**Verified**: all three files flip correctly. Fully reversible via prerm.

## Why it still doesn't reach the server (the real blocker)
Triggering `palm://com.palm.update/CheckForUpdate` returns `{status: Checking}`
but produces **zero packets** (tcpdump on eth0: no DNS, no SYN). `UpdateDaemon`
aborts at a **carrier gate**, before any network use:

```
[InitCarrier]: Unrecognized carrier            (x_palm_carrier = "---")
[GetPref]: pref has not been created
[HandleSenderGetLocationHostCB]: Unable to get domain
[ScheduleUpdateCheckActivityCB]: Scheduling Task Failed
[RoamingStatusCB]: ... state:noservice ... dataRegistered:false
```

### The host is NOT taken from DmTree's Addr
The decisive discovery: the server **host** is a *domain* derived from the
**`DMCARRIER` token** (system property `com.palm.properties.DMCARRIER`, **empty**
on this device) → carrier → domain, written to **`/var/lib/software/domain`**
(empty here) and injected via `OmaDm -set_domain`. Daemon strings:
`InitCarrier`, `FetchLocationHost`, `GetLocationHost`, `SetLocationHost`,
`HandleSenderGetLocationHostCB`, `RegenerateDmTree`, `"DMCARRIER token missing -
assuming HP"`, `"Unrecognized carrier %s"`, `"set domain: %s"`,
`"Unable to write domain to a file"`, `_ZN14SwUpdateSender14m_locationHostE`.

So patching DmTree's Addr changes the *path/scheme/port* but the **carrier→domain
step overrides the host** — and on a Wi-Fi-only TouchPad with an empty/`---`
carrier it fails ("Unrecognized carrier") and never gets a domain, so the whole
check is cancelled. Seeding `/var/lib/software/domain` with `www.webosarchive.org`
did **not** help — the carrier gate fails *first* (the domain file even survived
untouched, proving the daemon bailed before using it).

### Real endpoints are richer than `/swupdateserver`
From `UpdateDaemon` strings, beyond the OMA-DM `/swupdateserver`:
- `/palmcsext/services/swupdate/isOTAUpdateAvailable`
- `/palmcsext/swupdate/sustats`
- `/palmcsext/services/swupdate/setOTAUpdateResult`

A real community server has to answer these, not just the OMA-DM session.

## Path forward (for when we tackle it)
1. **Binary-patch `UpdateDaemon`** (it ships with DWARF symbols / readable
   strings) to: force the domain to `www.webosarchive.org`, and bypass the
   carrier/cellular/roaming conditions in `InitCarrier` / `FetchLocationHost` /
   `HandleSenderGetLocationHostCB` / `RoamingStatusCB`. Most reliable; folds into
   a future package alongside the DmTree redirect.
2. Or provision a recognized `DMCARRIER` so `InitCarrier` succeeds — but the
   built-in carrier→domain map points at Palm domains, so still needs the domain
   override. Brittle.
3. Cross-check the codepoet80 `webos-update-exploration` notes — the original
   test rig may have had a valid carrier or already patched this layer.

## How to reproduce the investigation
- `luna-send -n 1 palm://com.palm.update/CheckForUpdate '{}'`
- `tcpdump -i eth0 -n '(tcp[tcpflags] & tcp-syn != 0) or port 53'` during the check
- `grep -i 'UpdateDaemon\|carrier\|domain\|LocationHost' /var/log/messages`
- `cat /var/lib/software/domain` ; `luna-send ... systemProperties/Get
  '{"key":"com.palm.properties.DMCARRIER"}'`
