# TerraConverter geodata (Android)

Priređeni podatkovni releasi koje koristi [TerraConverter](<link na glavni TerraConverter repo>)
Android aplikacija. Ovaj repo **ne sadrži kod** — samo povremene GitHub
Release objave s jednom binarnom datotekom, koju aplikacija preuzme na
uređaj prvi put kad se koristi pretraga adresa/reverzno geokodiranje.

## Što je u jednom releaseu

Svaki release ima dvije datoteke:

- `Address.mobile.sqlite.gz` — SQLite indeks adresa za Hrvatsku (ulica,
  kućni broj, naselje, poštanski broj, grad + koordinate), u mršavoj shemi
  s običnim B-tree prefiksnim indeksom umjesto FTS5.
- `Address.mobile.sqlite.gz.sha256` — njegov checksum, koji app provjeri
  prije instalacije preuzete datoteke.

Release **tag** je verzija PODATAKA (npr. datum DGU objave izvora, ne
verzija aplikacije) — app uspoređuje taj tag s onim što već ima da
odluči postoji li noviji release.

## Zašto ovo postoji

TerraConverter desktop/CLI pretraga adresa već radi offline nad SQLite
indeksom izgrađenim iz DGU (Državna geodetska uprava) INSPIRE adresnog
skupa podataka, koristeći SQLite-ov FTS5 ekstenziju za pretragu teksta
(vidi `core/geocoding/dgu.py` u glavnom repou).

Android ne može iskoristiti taj indeks kakav je: Chaquopyjev ugrađeni
SQLite build nema FTS5 (provjereno pregledom `libsqlite3_python.so` — bez
`ENABLE_FTS5`/`fts5` stringova), a desktop indeks sam po sebi (~320 MB,
FTS5 interno duplicira tekst adrese) je i tako prevelik za APK.
`scripts/build_android_address_index.py` (u glavnom repou) preslaže
postojeći desktop indeks u mršaviju shemu bez FTS5 koja se distribuira
odavde — oko 192 MB raw / 76 MB gzip za cijelu državu, jednokratno
preuzimanje umjesto ugrađenog u APK.

## Kako ga app koristi

`AddressIndexManager.kt` (Android) provjerava `/releases/latest` ovog
repoa preko GitHub API-ja, preuzima `.gz` asset s prikazom napretka,
provjerava ga protiv `.sha256` sidecara, dekomprimira i atomski instalira
u privatni storage aplikacije. `core/geocoding/dgu_mobile.py`
(sinkroniziran u Android build) zatim izravno čita tu datoteku — vidi taj
modul i `TerraConverterBridge.geocodeAddress()`/`reverseGeocodeAddress()`
u glavnom repou za stranu koja pretražuje.

## Ažuriranje releasea (održavatelj)

1. U glavnom TerraConverter repou provjeri da je desktop DGU indeks
   izgrađen i ažuran (`external_data/dgu_addresses/Address.sqlite` — vidi
   README te mape za preuzimanje/izgradnju).
2. Pokreni `uv run python scripts/build_android_address_index.py`. Ovo
   proizvede `external_data/dgu_addresses/Address.mobile.sqlite(.gz)`.
3. Generiraj checksum sidecar: `sha256sum Address.mobile.sqlite.gz >
   Address.mobile.sqlite.gz.sha256`.
4. Napravi novi GitHub Release ovdje, s tagom koji predstavlja verziju
   podataka (npr. datum DGU objave), i priloži obje datoteke.

Ovo treba samo kad DGU ponovno objavi adresni skup — ne pri svakom
izdanju aplikacije.

## Izvor podataka i licenca

Adresni podaci dolaze iz DGU-ovog INSPIRE adresnog skupa (objavljen preko
geoportal.dgu.hr). DGU-ovi uvjeti ponovne upotrebe za baš ovaj skup nisu
potpuno jasni u trenutku pisanja — neki DGU INSPIRE slojevi pokriveni su
hrvatskom otvorenom dozvolom (data.gov.hr/otvorena-dozvola), dok je mrežna
usluga Adrese na drugom mjestu navedena kao ograničena na javnopravna
tijela uz potpisani protokol. Provjeri
https://geoportal.dgu.hr/#/menu/uvjeti-koristenja za trenutne uvjete prije
nego se na ovaj repo osloniš za nešto izvan vlastite upotrebe
TerraConverter aplikacije, i navedi DGU (Državna geodetska uprava,
geoportal.dgu.hr) kao izvor. Nad samim podacima u ovom repou nije
postavljena zasebna open-source licenca.
