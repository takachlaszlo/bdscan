# bdscan

Kezdőbarát parancssori segédprogram visszafejtett Blu-ray könyvtárak
vizsgálatához Debian 12 alatt. A `bdscan` a BDInfoCLI-ng és az FFmpeghez
tartozó `ffprobe` programokat fogja össze.

A program:

- kilistázza a Blu-ray lejátszási listáit;
- automatikusan kiválasztja a leghosszabb MPLS-t;
- BDInfo-riportot készít;
- összefoglalja a videó-, hang- és feliratsávokat;
- felismeri a PQ, HLG és az `ffprobe` által jelzett Dolby Vision adatokat;
- kérésre kiír egy szerkeszthető FFmpeg/libx264 parancsot;
- natív batch módban több release-t egymás után elemez.

> A `bdscan --ffmpeg` csak kiírja a kódolási parancsot. Nem indítja el
> automatikusan a kódolást.

## Elvárt könyvtárszerkezet

Egy release gyökerében legalább ennek kell léteznie:

```text
FILM_KONYVTARA/
└── BDMV/
    └── PLAYLIST/
        ├── 00000.mpls
        └── ...
```

A script nem fejti vissza az AACS/BD+ védelemmel ellátott lemezt.

Batch módban a megadott szülőkönyvtár közvetlen alkönyvtáraiban keres:

```text
RELEASES_ROOT/
├── Film.One/
│   └── BDMV/PLAYLIST/
├── Film.Two/
│   └── BDMV/PLAYLIST/
└── Nem.BluRay/
```

A példában a `Film.One` és `Film.Two` kerül feldolgozásra, a `Nem.BluRay`
kimarad.

## Telepítés Debian 12-re

### Függőségek

```bash
sudo apt update
sudo apt install -y git wget ca-certificates ffmpeg jq coreutils findutils
```

### .NET 8 SDK

```bash
cd "$HOME"
wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb \
  -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt update
sudo apt install -y dotnet-sdk-8.0
```

### BDInfoCLI-ng

```bash
cd "$HOME"
git clone https://github.com/Audionut/BDInfoCLI-ng.git
cd BDInfoCLI-ng

dotnet publish bdinfo-cli/bdinfo-cli.csproj \
  -c Release \
  -r linux-x64 \
  -o publish

sudo install -m 755 publish/bdinfo /usr/local/bin/bdinfo
```

Ne add hozzá a `--self-contained false` kapcsolót, mert a projekt egyfájlos,
tömörített kiadási beállításával ütközhet.

### bdscan

```bash
cd "$HOME"
git clone https://github.com/takachlaszlo/bdscan.git
cd bdscan
sudo install -m 755 ./bdscan /usr/local/bin/bdscan
```

Ellenőrzés:

```bash
command -v bdscan
bdscan --help
```

## Használat

### Egy release elemzése

```bash
bdscan "$HOME/torrents/qbittorrent/Film.Neve.1080p.BluRay"
```

Az aktuális könyvtár elemzése:

```bash
bdscan .
```

Útvonal nélkül szintén az aktuális könyvtárat használja:

```bash
bdscan
```

### Natív batch mód

A megadott könyvtár minden közvetlen alkönyvtárát feldolgozza, amelyben
`BDMV/PLAYLIST` található:

```bash
bdscan --batch "$HOME/torrents/qbittorrent"
```

Rövid kapcsolóval:

```bash
bdscan -b "$HOME/torrents/qbittorrent"
```

Másik riportkönyvtárral:

```bash
bdscan --batch \
  --output "$HOME/encode/bd-reports" \
  "$HOME/torrents/qbittorrent"
```

Batch módban:

- minden release ugyanazt az egy-release elemzést kapja;
- egy hibás release nem állítja le a többit;
- a végén sikeres/hibás összesítés készül;
- hibás release esetén a program kilépési kódja `1`;
- ha minden release sikeres, a kilépési kód `0`.

Ha a `--batch` után megadott könyvtár maga tartalmaz `BDMV/PLAYLIST`
struktúrát, azt egyetlen release-ként dolgozza fel.

### FFmpeg-parancs kiírása

Egy release-hez:

```bash
bdscan --ffmpeg .
```

Batch módban minden sikeresen elemzett release-hez:

```bash
bdscan --batch --ffmpeg "$HOME/torrents/qbittorrent"
```

A parancs csak mintaként kerül kiírásra; futtatás előtt ellenőrizni kell a
sávkiosztást, a CRF-et, a hangformátumot és a HDR-kezelést.

## Kapcsolók

```text
Usage:
  bdscan [OPTIONS] [BLU_RAY_ROOT]
  bdscan --batch [OPTIONS] [RELEASES_ROOT]

  -b, --batch        Több Blu-ray release elemzése
  -o, --output DIR   Riportkönyvtár (alapértelmezés: ~/encode)
  -f, --ffmpeg       Szerkeszthető FFmpeg/libx264 parancs kiírása
  -h, --help         Súgó megjelenítése
```

Az alapértelmezett riportkönyvtár környezeti változóval felülírható:

```bash
export BDSCAN_OUTPUT_DIR="$HOME/encode/bd-reports"
```

## Frissítés

```bash
cd "$HOME/bdscan"
git pull --ff-only
sudo install -m 755 ./bdscan /usr/local/bin/bdscan
bdscan --help
```

## Ellenőrzés és gyakori hibák

Az FFmpeg Blu-ray támogatása:

```bash
ffprobe -v error -protocols | grep -w bluray
```

Ha a `bdinfo` nem található:

```bash
command -v bdinfo
ls -l /usr/local/bin/bdinfo
```

Ha a program nem talál BDMV-struktúrát:

```bash
find "/utvonal/a/release-hez" -maxdepth 3 -type d \
  \( -name BDMV -o -name PLAYLIST \) -print
```

Az `ffprobe` olvasási hibájának gyakori okai:

- a Blu-ray még titkosított;
- hiányos a BDMV-struktúra;
- hiányzik egy hivatkozott M2TS fájl;
- nincs megfelelő olvasási jogosultság;
- az FFmpeg-build nem tartalmaz libbluray támogatást.

## Eltávolítás

```bash
sudo rm /usr/local/bin/bdscan
```
