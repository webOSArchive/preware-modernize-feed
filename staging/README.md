# Held packages (not in the live feed)

These are intentionally **excluded from `ipkgs/`** (the served feed) for now,
but kept here so their builds and metadata aren't lost.

## org.webosce.luna-update ("webOS CE 3.1.0")

Proof-of-concept OS update: sets the version string to "webOS CE 3.1.0" and
installs the LunaCE launcher; depends on the full modernization chain (TLS
updates, Enyo App Catalog, mojomail fix, root certs, QupZilla).

Pulled pending community review of which packages the update should require.

### To re-add to the feed
1. `git mv staging/org.webosce.luna-update_3.1.0_armv7.ipk ipkgs/`
2. Append the contents of `org.webosce.luna-update.Packages-stanza.txt` to
   `ipkgs/Packages` (update the `Depends:` line if the required set changed;
   if the .ipk is rebuilt, refresh `MD5Sum`/`Size`).
3. Regenerate `ipkgs/Packages.gz` from `ipkgs/Packages`.

## org.webosce.swupdate-redirect (OTA server redirect)

Repoints the OMA-DM software-update client to webOS Archive. Held because the
DmTree URL change alone doesn't drive OTA — the daemon gates on a carrier/domain
layer first. Full investigation in `swupdate-redirect-FINDINGS.md`.

Note: `luna-update`'s stanza lists this as a dependency; both are held together
and should be re-added (with the carrier/domain fix) as a unit.

### To re-add to the feed
1. `git mv staging/org.webosce.swupdate-redirect_1.0.0_all.ipk ipkgs/`
2. Append `org.webosce.swupdate-redirect.Packages-stanza.txt` to `ipkgs/Packages`.
3. Regenerate `ipkgs/Packages.gz`.
