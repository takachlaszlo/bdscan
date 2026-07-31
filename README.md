# bdscan

Kezdőbarát parancssori segédprogram Blu-ray könyvtárak vizsgálatához Debian 12
alatt. A `bdscan` a
[BDInfoCLI-ng](https://github.com/Audionut/BDInfoCLI-ng) és az FFmpeghez tartozó
`ffprobe` programokat fogja össze egyetlen paranccsá.

A program:

- kilistázza a Blu-ray lejátszási listáit (MPLS fájlok);
- automatikusan kiválasztja a leghosszabb lejátszási listát;
- BDInfo riportot készít;
- összefoglalja a videó-, hang- és feliratsávokat;
- megjeleníti a kodeket, felbontást, csatornaszámot, nyelvet és az elérhető
  bitrátaadatokat;
- felismeri a PQ és HLG HDR-jelölést, valamint az `ffprobe` által átadott
  Dolby Vision-információt;
- kérésre kiír egy biztonságosan idézőjelezett, szerkeszthető FFmpeg/libx264
  parancsot.

> [!IMPORTANT]
> A `bdscan --ffmpeg` csak kiírja a kódolási parancsot. Nem indítja el
> automatikusan a kódolást. Futtatás előtt mindig ellenőrizd a sávkiosztást,
> a minőségi beállításokat és HDR-forrás esetén a kívánt HDR/SDR munkafolyamatot.

## Mit vár bemenetként?

A bemenet egy már visszafejtett, könyvtárba másolt Blu-ray struktúra legyen.
A megadott gyökérkönyvtárban legalább ennek kell léteznie:

```text
FILM_KONYVTARA/
└── BDMV/
    └── PLAYLIST/
        ├── 00000.mpls
        └── ...
```

AACS/BD+ védelemmel ellátott, közvetlenül olvasott lemezt a script nem fejt
vissza.

## Teljes telepítés Debian 12-re

Az alábbi lépéseket sorrendben hajtsd végre. A `$HOME` a saját felhasználói
könyvtáradat jelenti, például `/home/felhasznalo`.

### 1. Alapcsomagok telepítése

```bash
sudo apt update
sudo apt install -y git wget ca-certificates ffmpeg jq coreutils
```

Ezek szerepe:

- `git`: projektek letöltése GitHubról;
- `wget` és `ca-certificates`: a Microsoft csomagtár biztonságos letöltése;
- `ffmpeg`: tartalmazza az elemzéshez használt `ffprobe` programot;
- `jq`: az `ffprobe` JSON-kimenetének feldolgozása;
- `coreutils`: többek között a `realpath` és `mktemp` parancsokat biztosítja.

### 2. A .NET 8 SDK telepítése

A BDInfoCLI-ng lefordításához .NET SDK szükséges. Add hozzá a Microsoft
hivatalos Debian 12 csomagtárát:

```bash
cd "$HOME"

wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb \
  -O packages-microsoft-prod.deb

sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

sudo apt update
sudo apt install -y dotnet-sdk-8.0
```

Ellenőrzés:

```bash
dotnet --list-sdks
```

A kimenetben szerepelnie kell egy `8.0.x` verziónak.

### 3. A BDInfoCLI-ng fordítása és telepítése

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

> [!WARNING]
> Ne add hozzá a `--self-contained false` kapcsolót. A projekt egyfájlos,
> tömörített kiadási beállítása ezzel ütközik, és `NETSDK1176` fordítási hibát
> okozhat.

Ellenőrzés:

```bash
command -v bdinfo
bdinfo --version
```

Az első parancs várható kimenete:

```text
/usr/local/bin/bdinfo
```

### 4. A bdscan telepítése

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
echo $?
```

A `command -v` várhatóan `/usr/local/bin/bdscan` értéket, az `echo $?` pedig
`0` értéket ír ki.

### 5. Az FFmpeg Blu-ray támogatásának ellenőrzése

```bash
ffprobe -v error -protocols | grep -w bluray
```

Ha a kimenet tartalmazza a `bluray` szót, az FFmpeg megfelelő. Ha nincs
kimenet, az adott FFmpeg-build nem tartalmaz libbluray támogatást.

## Használat

### Film vizsgálata teljes útvonallal

Az útvonalat mindig tedd idézőjelek közé. Ez a szóközt tartalmazó neveket is
biztonságosan kezeli:

```bash
bdscan "$HOME/storage/Film.Neve.1080p.Blu-ray"
```

Helyes:

```bash
bdscan "$HOME/storage/Film neve"
```

Helytelen:

```bash
bdscan "\$HOME/storage/Film neve"
```

A helytelen példában a `\` megakadályozza a `$HOME` változó feloldását.

### Vizsgálat a film könyvtárából

```bash
cd "$HOME/storage/Film.Neve.1080p.Blu-ray"
bdscan .
```

Ha nem adsz meg útvonalat, a script szintén az aktuális könyvtárat használja:

```bash
bdscan
```

### Másik riportkönyvtár megadása

A BDInfo riport alapértelmezésben a `~/encode` könyvtárba kerül. Másik cél:

```bash
bdscan --output "$HOME/reports" .
```

Röviden ugyanez:

```bash
bdscan -o "$HOME/reports" .
```

Az alapértelmezett cél tartósan is felülírható a shell beállításában:

```bash
export BDSCAN_OUTPUT_DIR="$HOME/reports"
bdscan .
```

### Szerkeszthető x264 parancs kiírása

```bash
bdscan --ffmpeg .
```

Röviden:

```bash
bdscan -f .
```

## Példa a fontosabb kimenetre

```text
Selected playlist
  MPLS:     00000.MPLS
  Duration: 01:51:37

Stream summary
  Video #0: h264 (High), 1920x1080, yuv420p, 28.00 Mb/s
  Audio #1: eng, pcm_bluray, 6 ch (5.1), 6.91 Mb/s
  Subtitle #2: eng, hdmv_pgs_subtitle
```

Az `und` azt jelenti, hogy az `ffprobe` nem adott át nyelvi címkét. A
`not reported` azt jelenti, hogy az adott bitrátaadat nem volt elérhető.

## Parancssori kapcsolók

```text
Usage: bdscan [OPTIONS] [BLU_RAY_ROOT]

  -o, --output DIR   Riportkönyvtár (alapértelmezés: ~/encode)
  -f, --ffmpeg       Szerkeszthető FFmpeg/libx264 parancs kiírása
  -h, --help         Súgó megjelenítése
```

## Frissítés

### bdscan frissítése

```bash
cd "$HOME/bdscan"
git pull --ff-only
sudo install -m 755 ./bdscan /usr/local/bin/bdscan
```

### BDInfoCLI-ng frissítése

```bash
cd "$HOME/BDInfoCLI-ng"
git pull --ff-only

dotnet publish bdinfo-cli/bdinfo-cli.csproj \
  -c Release \
  -r linux-x64 \
  -o publish

sudo install -m 755 publish/bdinfo /usr/local/bin/bdinfo
```

## Gyakori hibák

### `Required command 'bdinfo' was not found`

A BDInfoCLI-ng nincs telepítve, vagy nem a keresési útvonalon található.

```bash
command -v bdinfo
ls -l /usr/local/bin/bdinfo
```

Ha egyik sem találja, hajtsd végre újra a BDInfoCLI-ng telepítési lépéseit.

### `dotnet: command not found`

A .NET SDK nincs telepítve. Hajtsd végre a „.NET 8 SDK telepítése” részt, majd
ellenőrizd:

```bash
dotnet --list-sdks
```

### `No BDMV directory found`

Nem a Blu-ray gyökérkönyvtárát adtad meg. Ellenőrizd:

```bash
ls -ld "/utvonal/a/filmhez/BDMV"
ls -ld "/utvonal/a/filmhez/BDMV/PLAYLIST"
```

### `This ffprobe build has no bluray/libbluray protocol`

Ellenőrizd és telepítsd újra a Debian FFmpeg-csomagját:

```bash
sudo apt update
sudo apt install --reinstall ffmpeg
ffprobe -v error -protocols | grep -w bluray
```

### Az `ffprobe` nem tudja olvasni a playlistet

Lehetséges okok:

- a Blu-ray még titkosított;
- hiányos a BDMV-struktúra;
- hiányzik egy, a playlist által hivatkozott M2TS fájl;
- nincs olvasási jogosultságod.

Alapvető ellenőrzés:

```bash
find "/utvonal/a/filmhez/BDMV/PLAYLIST" -maxdepth 1 -type f | head
find "/utvonal/a/filmhez/BDMV/STREAM" -maxdepth 1 -type f | head
```

## Eltávolítás

Csak a `bdscan` rendszerparancs eltávolítása:

```bash
sudo rm /usr/local/bin/bdscan
```

A saját könyvtáradban lévő Git-klón és a korábban elkészített riportok ettől
nem törlődnek.
