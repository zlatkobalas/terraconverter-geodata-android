# TerraConverter geodata (Android)

Prebuilt data releases consumed by the [TerraConverter](<link na glavni TerraConverter repo>)
Android app. This repo holds **no source code** — just periodic GitHub
Releases with a binary data file, downloaded on-device the first time the
app's address search/reverse-geocoding feature is used.

## What's in a release

Each release has two assets:

- `Address.mobile.sqlite.gz` — a SQLite address index for Croatia (street,
  house number, settlement, postal code, city + coordinates), in a lean
  schema with a plain B-tree prefix index instead of FTS5.
- `Address.mobile.sqlite.gz.sha256` — its checksum, verified by the app
  before installing the download.

The release **tag** is the data version (e.g. the date of the source DGU
publication, not an app version number) — the app compares this tag
against what it already has to decide whether a newer release exists.

## Why this exists

TerraConverter's desktop/CLI address search already works offline against
a SQLite index built from the DGU (Državna geodetska uprava / Croatian
State Geodetic Administration) INSPIRE Addresses dataset, using SQLite's
FTS5 extension for full-text search (see `core/geocoding/dgu.py` in the
main repo).

Android can't reuse that index as-is: Chaquopy's bundled SQLite build does
not include FTS5 (confirmed by inspecting `libsqlite3_python.so` — no
`ENABLE_FTS5` or `fts5` strings), and the desktop index itself (~320 MB,
FTS5 duplicates the address text internally) is too large to bundle in
the app regardless. `scripts/build_android_address_index.py` (in the main
repo) reshapes the existing desktop index into the lean, FTS5-free schema
that ships from here — about 192 MB raw / 76 MB gzip for the whole
country, a one-time download instead of something bundled in the APK.

## How the app uses it

`AddressIndexManager.kt` (Android) checks this repo's `/releases/latest`
via the GitHub API, downloads the `.gz` asset with progress reporting,
verifies it against the `.sha256` sidecar, decompresses it, and installs
it atomically into the app's private storage. `core/geocoding/dgu_mobile.py`
(synced into the Android build) then reads it directly — see that module
and `TerraConverterBridge.geocodeAddress()`/`reverseGeocodeAddress()` in
the main repo for the query side.

## Updating a release (maintainer)

1. In the main TerraConverter repo, make sure the desktop DGU index is
   built and current (`external_data/dgu_addresses/Address.sqlite` — see
   that folder's own README for how to fetch/build it).
2. Run `uv run python scripts/build_android_address_index.py`. This
   produces `external_data/dgu_addresses/Address.mobile.sqlite(.gz)`.
3. Generate the checksum sidecar: `sha256sum Address.mobile.sqlite.gz >
   Address.mobile.sqlite.gz.sha256`.
4. Create a new GitHub Release here, tagged with the data version (e.g.
   the DGU publication date), and attach both files.

This only needs to happen when DGU republishes the Addresses dataset —
not on every app release.

## Data source & license

The address data originates from DGU's INSPIRE Addresses dataset
(published via geoportal.dgu.hr). DGU's reuse terms for this specific
dataset are not fully clear at the time of writing — some DGU INSPIRE
layers are covered by Croatia's open data license
(data.gov.hr/otvorena-dozvola), while the Addresses network service is
listed elsewhere as restricted to public-law bodies under a signed
protocol. Check https://geoportal.dgu.hr/#/menu/uvjeti-koristenja for the
current terms before relying on this repo for anything beyond the
TerraConverter app's own use, and attribute DGU (Državna geodetska
uprava, geoportal.dgu.hr) as the source. No separate open-source license
is asserted over the data itself in this repo.
