# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Preware feed repository for webOS devices (Pre3, TouchPad, Touchpad Go). It builds and hosts IPK packages that can be installed via Preware using the feed URL: `https://gitlab.com/nizovn/preware_feed/raw/master/ipkgs/`

## Build Commands

**Build all packages and generate feed index:**
```bash
./package.sh
```

This script:
1. Copies packages to `/srv/preware/build/nizovn_apps`
2. Runs `make` on each package with a Makefile
3. Collects built `.ipk` files into `ipkgs/`
4. Uses `IpkgFeedGenerator.jar` to generate the `Packages` index
5. Sorts and compresses to create `Packages.gz`

**Build a single package:**
```bash
make -C packages/<package-name>
```

## Architecture

### Directory Structure
- `packages/` - Source package definitions, each with a Makefile
- `ipkgs/` - Built IPK files and feed index (Packages, Packages.gz)
- `add-to-feed/` - Pre-built IPK files to add directly to the feed
- `IpkgFeedGenerator.jar` - Java tool that generates the Packages index from IPK metadata

### Package Structure
Each package in `packages/` follows the webOS-Internals build system pattern:
- `Makefile` - Defines package metadata (NAME, VERSION, APP_ID, DEPENDS, etc.) and build targets
- Packages include `../../support/package.mk` (external dependency from webos-internals/build)
- Build output goes to `build/armv7/usr/palm/applications/<APP_ID>/`

### Package Metadata Fields
Required Makefile variables: NAME, TITLE, APP_ID, VERSION, TYPE, CATEGORY, HOMEPAGE, MAINTAINER, MINWEBOSVERSION, DESCRIPTION, DEVICECOMPATIBILITY

Optional: DEPENDS, ICON, LICENSE, CHANGELOG, SCREENSHOTS

## Dependencies

- Java runtime (for IpkgFeedGenerator.jar)
- webOS-Internals build infrastructure (https://github.com/webos-internals/build) - provides `support/package.mk`
- Build environment at `/srv/preware/build/`
