# Chapter 1 — Setup Project Unity

> Kalau pondasinya rapi, rumah-nya gak gampang miring.

Di chapter ini kita bikin project Unity baru, install package yang butuh, atur folder structure, import asset, dan setup version control. **Nyebelin tapi penting.** Kalau kamu skip step di sini, nanti chapter berikutnya akan banyak error setting yang bikin pusing.

---

## Tujuan Chapter

Setelah chapter ini selesai, kamu akan punya:

- Project Unity 2D baru bernama `StardewClone` (atau apa pun namamu).
- Package: 2D Tilemap Editor, 2D Sprite, Cinemachine, Input System.
- Folder structure rapi (`Scripts/`, `Sprites/`, `Prefabs/`, `Scenes/`, dll).
- Asset Sprout Lands ter-import.
- Pixel Per Unit (PPU) standar yang konsisten = **16**.
- Repo Git lokal dengan `.gitignore` Unity yang benar.
- Scene `MainScene` yang sudah di-save.

---

## Prasyarat

- [Chapter 0](00-pendahuluan.md) selesai.
- Unity 2022.3 LTS terinstall.

---

## Step 1 — Bikin Project Baru

Buka **Unity Hub** → klik tombol biru **New project** (kanan atas).

| Setting | Value | Kenapa |
|---------|-------|--------|
| Editor Version | 2022.3.x LTS | Versi paling stabil untuk learning. |
| Template | **2D (Built-in Render Pipeline)** | Game kita 2D pixel art, gak butuh URP/HDRP. |
| Project name | `StardewClone` (atau bebas) | Hindari spasi & karakter aneh. |
| Location | `D:\UnityProjects\` (atau bebas) | **Jangan di OneDrive / Google Drive folder** — sering bikin file lock issues. |

> **Note Unity 6 / URP**: Kalau kamu pakai template **2D (URP)**, juga oke. Sedikit perbedaan setting di later chapter (terutama lighting di Chapter 8). Kamu boleh pilih ini juga.

Klik **Create project**. Tunggu Unity buka — first time bisa 1–3 menit.

---

## Step 2 — Verifikasi Project Settings

Setelah Editor terbuka, kita pastikan settingnya cocok untuk pixel art.

### Edit → Project Settings

#### Editor

- **Default Behavior Mode**: 2D (otomatis untuk template 2D, tapi cek untuk pastikan).

#### Player

- **Company Name**: nama kamu / studio.
- **Product Name**: `StardewClone`.

#### Quality

Untuk pixel art, kita matiin anti-aliasing supaya pixel-nya tetap tajam:

- Pilih level **Default** (atau apa pun yang aktif).
- **Anti Aliasing**: Disabled.
- **V Sync Count**: Don't Sync (atau Every V Blank, terserah).

#### Graphics (Built-in RP)

Biarin default dulu.

---

## Step 3 — Install Package

Window → **Package Manager** → di dropdown kiri atas pilih **Unity Registry**.

Cari & install:

1. **2D Tilemap Editor** — biasanya udah ada (dependency dari template 2D).
2. **2D Tilemap Extras** — install. Ada Rule Tile, sangat berguna untuk tilemap.
3. **Cinemachine** — install. Untuk kamera pintar.
4. **Input System** — install. Versi modern dari `Input.GetKey`.

> Unity akan tanya: "Restart now to use new Input System?" — Pilih **Yes**. Editor akan restart.

> **Penting tentang Input System**: setelah restart, **Input.GetKey() lama akan tetap berfungsi** karena setting-nya `Both` di **Project Settings → Player → Active Input Handling**. Course ini pakai **gabungan**: Input System untuk player movement (lebih clean), Input lama untuk hotkey simpel. Pastikan settingnya `Both`.

#### Verifikasi Active Input Handling:

- **Edit → Project Settings → Player → Other Settings**
- Scroll ke **Configuration**.
- **Active Input Handling**: pilih `Both` (kalau dropdown belum ada `Both`, pilih `Input System Package (New)` saja).

Klik **Apply** kalau ada notifikasi restart, restart Editor lagi (Unity bisa reli minta restart 1–2x).

---

## Step 4 — Folder Structure

Folder rapi = mental kamu rapi. Bikin folder ini di `Assets/` (klik kanan di Project window → Create → Folder):

```
Assets/
├── _Project/                  # Optional: container utama (underscore biar di top)
│   ├── Animations/
│   ├── Audio/
│   │   ├── Music/
│   │   └── SFX/
│   ├── Materials/
│   ├── Prefabs/
│   │   ├── Player/
│   │   ├── Tools/
│   │   ├── Crops/
│   │   └── UI/
│   ├── ScriptableObjects/
│   │   ├── Items/
│   │   ├── Tools/
│   │   ├── Crops/
│   │   └── Seasons/
│   ├── Scenes/
│   ├── Scripts/
│   │   ├── Player/
│   │   ├── Farming/
│   │   ├── Inventory/
│   │   ├── Time/
│   │   ├── UI/
│   │   ├── Save/
│   │   └── Utility/
│   ├── Sprites/
│   │   ├── Characters/
│   │   ├── Tilesets/
│   │   ├── Tools/
│   │   ├── Crops/
│   │   └── UI/
│   └── UI/                    # Untuk UXML, USS kalau pakai UI Toolkit
└── ThirdParty/                # Asset dari luar (Sprout Lands, dll)
```

Kenapa `_Project` di-prefix underscore? Supaya selalu ada di paling atas list (alphabetical sort).

> **Tip**: Drag-drop folder dari File Explorer ke `Assets/` di Project window juga bisa.

---

## Step 5 — Import Asset Pack

Kita pakai **Sprout Lands** dari Cup Nooble. Steps:

1. Buka https://cupnooble.itch.io/sprout-lands-asset-pack
2. Klik **Download Now** → "No thanks, just take me to the downloads" (atau bayar dukung kreatornya, suggested).
3. Download `Sprout Lands - Sprites - Basic pack.zip`.
4. Extract zip ke folder `ThirdParty/SproutLands/` di project kamu (langsung di filesystem, bukan via Unity).
5. Balik ke Unity → Project window. Asset akan auto-import (lihat progress bar di kanan bawah).

Setelah import selesai, kamu akan punya banyak file PNG di `Assets/ThirdParty/SproutLands/`.

### Setting Sprite Default

Default-nya Unity import sprite dengan PPU = 100, filter = Bilinear → bikin pixel art jadi blur. Kita harus override.

**Pilih semua sprite** di SproutLands folder (Ctrl+A di folder tertentu yang berisi PNG). Di Inspector:

- **Texture Type**: `Sprite (2D and UI)`
- **Sprite Mode**: `Single` (atau `Multiple` kalau itu sprite sheet — kita tweaking per-asset nanti)
- **Pixels Per Unit**: **16**
- **Filter Mode**: **Point (no filter)**
- **Compression**: **None**

Klik **Apply**. Tunggu re-import.

> **Kenapa PPU = 16?** Karena Sprout Lands grid-nya 16x16 pixel. Dengan PPU 16, satu tile = 1 unit Unity. Memudahkan kalkulasi posisi (player jalan 1 unit = 1 tile).

> Kalau asset pack kamu pakai 32x32 atau 48x48, sesuaikan PPU = 32 atau 48. **Konsistenkan untuk semua sprite di project**.

---

## Step 6 — Save Scene & Buat MainScene

Default project punya scene `SampleScene` di `Assets/Scenes/`. Mari kita bersihin:

1. Hierarchy → kanan-klik di SampleScene → tidak usah dihapus, tapi **rename** jadi `MainScene` (atau hapus & bikin baru).
2. Pindahkan `MainScene.unity` ke `Assets/_Project/Scenes/`.
3. **File → Save** (`Ctrl+S`).
4. **File → Build Settings** → seret `MainScene` ke list **Scenes In Build** (di atas). Pastikan ada di index 0.

---

## Step 7 — Setup Camera

Default `Main Camera` punya:

- **Projection**: Orthographic (bagus untuk 2D)
- **Size**: 5 (jadi viewport vertikal = 10 units)

Untuk pixel art 16 PPU, size 5 = vertikal 160 pixel pas mata kita. Resolusi 1920×1080 → satu pixel game ditampilkan sebagai (1080/160) ≈ 6.75 pixel layar → angka ganjil → muncul pixel jitter pas player jalan.

**Solusi sederhana**: pakai camera size yang menghasilkan integer scaling. Untuk 1080p:

- 1080 / 6 = 180 → camera size = `5.625` (180/2/16)

Tapi paling gampang: install `Pixel Perfect Camera` (sudah ada di package 2D).

#### Tambahkan Pixel Perfect Camera

1. Pilih **Main Camera** di Hierarchy.
2. Inspector → **Add Component** → cari `Pixel Perfect Camera`.
3. Set:
   - **Assets Pixels Per Unit**: 16
   - **Reference Resolution X / Y**: 320 / 180 (rasio 16:9, bisa di-upscale 6x ke 1920×1080 dengan integer scaling)
   - **Upscale Render Texture**: ✅
   - **Pixel Snapping**: ✅

4. Set **Background**: warna hitam atau biru langit (terserah).

> **Catatan**: Reference Resolution 320×180 itu pilihan personal. Stardew Valley pakai sekitar 320×180 (ditampilkan di 1920×1080 dengan upscale 6x). Untuk feel lebih dekat-up, pakai 384×216 atau 480×270.

---

## Step 8 — Setup Git

Di terminal (Git Bash kalau Windows), masuk ke root folder project:

```bash
cd D:/UnityProjects/StardewClone
```

Init repo:

```bash
git init
git branch -M main
```

### Buat .gitignore Unity

Bikin file `.gitignore` di root project. Isinya (copy dari https://github.com/github/gitignore/blob/main/Unity.gitignore):

```gitignore
# This .gitignore file should be placed at the root of your Unity project directory
#
# Get latest from https://github.com/github/gitignore/blob/main/Unity.gitignore
#
/[Ll]ibrary/
/[Tt]emp/
/[Oo]bj/
/[Bb]uild/
/[Bb]uilds/
/[Ll]ogs/
/[Uu]ser[Ss]ettings/

# MemoryCaptures can get excessive in size.
# They also could contain extremely sensitive data
/[Mm]emoryCaptures/

# Recordings can get excessive in size
/[Rr]ecordings/

# Uncomment this line if you wish to ignore the asset store tools plugin
# /[Aa]ssets/AssetStoreTools*

# Autogenerated Jetbrains Rider plugin
/[Aa]ssets/Plugins/Editor/JetBrains*

# Visual Studio cache directory
.vs/

# Gradle cache directory
.gradle/

# Autogenerated VS/MD/Consulo solution and project files
ExportedObj/
.consulo/
*.csproj
*.unityproj
*.sln
*.suo
*.tmp
*.user
*.userprefs
*.pidb
*.booproj
*.svd
*.pdb
*.mdb
*.opendb
*.VC.db

# Unity3D generated meta files
*.pidb.meta
*.pdb.meta
*.mdb.meta

# Unity3D generated file on crash reports
sysinfo.txt

# Builds
*.apk
*.aab
*.unitypackage
*.app

# Crashlytics generated file
crashlytics-build.properties

# Packed Addressables
/[Aa]ssets/[Aa]ddressable[Aa]ssets[Dd]ata/*/*.bin*

# Temporary auto-generated Android Assets
/[Aa]ssets/[Ss]treamingAssets/aa.meta
/[Aa]ssets/[Ss]treamingAssets/aa/*
```

> **Penting**: di .gitignore Unity, kita **tidak ignore** folder `Assets/` dan `ProjectSettings/` — itu yang harus tracked. Kita **ignore** `Library/`, `Temp/`, `Logs/` karena Unity bisa regenerate dari `Assets/`.

### Setup Git LFS untuk asset besar

Kalau kamu impor banyak gambar besar / audio, Git LFS membantu:

```bash
git lfs install
git lfs track "*.psd" "*.wav" "*.mp3" "*.fbx"
git add .gitattributes
```

Skip kalau Sprout Lands aja (file kecil-kecil).

### First Commit

```bash
git add .
git commit -m "chore: initial Unity project setup with Sprout Lands assets"
```

Push ke GitHub kalau mau (bikin repo kosong di GitHub dulu):

```bash
git remote add origin https://github.com/USERNAME/StardewClone.git
git push -u origin main
```

---

## Step 9 — Bikin Empty Test Object

Untuk verifikasi semua jalan, kita bikin GameObject simpel.

1. Hierarchy → klik kanan → Create Empty → namain `_Managers`. Ini akan jadi container untuk script-script global (TimeManager, SaveManager, dll) di chapter berikutnya. Letaknya di posisi (0,0,0).
2. Bikin GameObject lain: klik kanan → 2D Object → **Sprites → Square**. Namain `TestSprite`. Ini sprite hijau/putih default.
3. Pilih `TestSprite` → Inspector → set posisi ke (0, 0, 0).
4. **Pencet Play**. Liat di Game view, harusnya muncul kotak putih di tengah.

Kalau muncul, **selesai**. Stop Play.

---

## Step 10 — Folder Structure Validation

Buka file explorer ke project root. Harusnya kamu lihat:

```
StardewClone/
├── Assets/
│   ├── _Project/
│   │   ├── Animations/
│   │   ├── Audio/
│   │   ├── Materials/
│   │   ├── Prefabs/
│   │   ├── ScriptableObjects/
│   │   ├── Scenes/
│   │   │   └── MainScene.unity
│   │   ├── Scripts/
│   │   ├── Sprites/
│   │   └── UI/
│   └── ThirdParty/
│       └── SproutLands/
│           └── (banyak file PNG)
├── Library/                 (di-ignore Git)
├── Logs/                    (di-ignore Git)
├── Packages/
│   ├── manifest.json
│   └── packages-lock.json
├── ProjectSettings/
│   ├── ... (banyak .asset)
├── Temp/                    (di-ignore Git)
├── UserSettings/            (di-ignore Git)
└── .gitignore
```

Kalau matches, **kamu jago**.

---

## Troubleshooting

### Sprite blur padahal sudah Point Filter

- Cek **Pixel Perfect Camera** → "Upscale Render Texture" centang.
- Cek camera **Background Type** kalau pakai URP, jangan Skybox.
- Restart Editor (kadang setting baru terapply setelah restart).

### "Assembly with name X has invalid version" / "Multiple precompiled assemblies"

- Window → Package Manager → uninstall package yang konflik, install ulang.
- Hapus folder `Library/`, restart Unity (regenerate dari Assets, bisa lama).

### Input System gak detect input

- Edit → Project Settings → Player → Active Input Handling = `Both`. Restart.
- Pastikan Camera punya Tag `MainCamera` (default). Beberapa script Input System nyari camera by tag.

### Git LFS error "this exceeds GitHub's file size limit"

- File > 100MB harus pakai LFS. Track sebelum commit.
- Kalau udah ke-commit non-LFS, jalankan `git lfs migrate import --include="*.png"` (hati-hati, rewrites history).

---

## Latihan

1. **Coba ganti Pixel Perfect Camera reference resolution** dari 320×180 jadi 480×270. Apa yang berubah di Game view?
2. **Bikin sprite dummy player**: drag salah satu PNG character dari Sprout Lands ke Hierarchy. Liat propertinya di Inspector.
3. **Coba commit sebuah perubahan kecil** (misal: rename `_Managers` → `_Systems`). Pakai `git status`, `git diff`, `git commit`. Familiarisasikan diri dengan flow.
4. **Bikin file `README.md` di root project** dengan deskripsi pendek game-mu. Commit.

---

## Recap

Yang sudah kamu punya setelah chapter ini:

- [x] Project Unity 2D dengan Built-in RP (atau URP).
- [x] Package: 2D Tilemap, Cinemachine, Input System.
- [x] Folder structure rapi.
- [x] Sprout Lands assets ter-import dengan PPU 16, Point filter.
- [x] Pixel Perfect Camera setup di Main Camera.
- [x] Git repo dengan .gitignore Unity yang proper.
- [x] `MainScene` di-save di build settings.

Kalau ada yang skip, balik dulu sebelum lanjut. Chapter 2 akan langsung mulai coding player.

---

## Lanjut

[**Chapter 2 — Player Character & Movement →**](02-player-character.md)

[← Chapter 0](00-pendahuluan.md) | [Daftar Isi](index.md)
