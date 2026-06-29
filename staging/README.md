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
