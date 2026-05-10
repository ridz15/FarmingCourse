# Chapter 4 — Tilemap & Membangun Dunia

> "Map yang baik ngajarin pemain pakai mata, bukan pakai tutorial."

Sekarang saatnya bikin **dunia**. Kita pakai Unity Tilemap untuk paint tile (rumput, jalan, air, pasir) dengan cepat, plus **Rule Tile** supaya tile auto-connect (rumput pinggir map otomatis pakai sprite "tepi rumput").

---

## Tujuan Chapter

Setelah chapter ini:

- Paham `Grid`, `Tilemap`, `Tilemap Renderer`, `Tilemap Collider 2D`.
- Punya 4 tilemap layer:
  - `Ground` — rumput, pasir, jalan (no collider)
  - `Decoration` — flowers, batu kecil (no collider)
  - `Obstacles` — pohon, batu besar, pagar (with collider, sorted by Y)
  - `Water` — air (with collider, gak bisa dilewati)
- Bisa paint tile cepat dengan Tile Palette.
- Pakai **Rule Tile** untuk auto-tiling rumput.
- Set up sorting layers untuk depth top-down.

---

## Prasyarat

- [Chapter 3](03-camera-follow.md) selesai.
- Asset Sprout Lands ter-import (Chapter 1).

---

## Konsep Tilemap

### Apa itu Tilemap?

Tilemap adalah **grid of tile**, di mana tiap cell bisa kamu set sprite. Lebih efisien dari nge-spawn ribuan GameObject sprite individual:

- Dirender dengan **batch** (1 draw call untuk 1 tilemap).
- Bisa di-paint pakai Tile Palette tool (kayak Photoshop).
- Tile Asset (`*.asset` file) reusable.

### Hierarki Komponen

```
Grid (GameObject) ───── komponen "Grid" — define cell size & layout
└── Tilemap (GameObject) ── komponen "Tilemap" + "Tilemap Renderer"
    ├── (banyak Tile asset di-paint)
    └── Optional: Tilemap Collider 2D + Composite Collider 2D
```

Satu Grid bisa punya **banyak Tilemap children** untuk layering.

### Tile Asset Types

- **Tile** (basic) — 1 sprite, tile statis.
- **Animated Tile** — beberapa sprite cycling (untuk air, api).
- **Rule Tile** — auto-pick sprite berdasarkan tetangga (untuk rumput dengan tepi otomatis).
- **Random Tile** — pilih random dari pool sprite (untuk variation).

---

## Step 1 — Buat Grid & Tilemap

Hierarchy → klik kanan → **2D Object → Tilemap → Rectangular**.

Unity bikin:

```
Grid
└── Tilemap
```

Grid akan kita rename jadi `World`, dan Tilemap default rename jadi `Ground`.

### Pengaturan Cell Size

Pilih `World` (Grid) → Inspector → komponen Grid:

- **Cell Size**: `(1, 1, 0)` (default). Karena PPU = 16 dan sprite tile 16×16, satu cell = 1 unit Unity. ✅

---

## Step 2 — Buat 4 Tilemap Layers

Hierarchy → klik kanan `World` → 2D Object → Tilemap → Rectangular. Ulangi sampai punya 4 children dari `World`:

```
World (Grid)
├── Ground         (Sorting Order = 0)
├── Decoration     (Sorting Order = 1)
├── Obstacles      (Sorting Order = 2, with collider)
└── Water          (Sorting Order = 1, with collider)
```

> Beberapa Unity version lain: bikin Tilemap baru lewat klik kanan di Hierarchy, gak harus child dari Grid. Pastikan kamu drag jadi child `World` setelahnya.

### Set Sorting

Pilih masing-masing Tilemap → Inspector → **Tilemap Renderer** → atur:

| Tilemap | Sorting Layer | Order in Layer |
|---------|---------------|----------------|
| Ground | `Default` | 0 |
| Decoration | `Default` | 1 |
| Obstacles | `Default` | 2 |
| Water | `Default` | 1 |

Tapi untuk Obstacles, kita mau sort berdasarkan **Y position** (semakin bawah, semakin di depan — untuk ngasih kesan 2.5D). Ini akan kita setup di Step 8.

---

## Step 3 — Buka Tile Palette

Window → 2D → **Tile Palette**. Kalau gak ada, install package `2D Tilemap Editor`.

Window Tile Palette muncul. Klik dropdown **Create New Palette**:

- Name: `MainPalette`
- Grid: Rectangle
- Cell Size: Manual `(1, 1, 0)`
- Save ke: `Assets/_Project/Sprites/Tilesets/`

---

## Step 4 — Drag Sprite ke Palette

Buka folder Sprout Lands di Project. Cari spritesheet ground (biasanya namanya `Tilled_Dirt.png`, `Grass.png`, `Sand.png`, `Water.png`, dll). 

Pastikan spritesheet ground sudah di-slice:

1. Pilih PNG → **Sprite Mode**: Multiple → **Sprite Editor**.
2. Slice → Grid By Cell Size → Pixel Size: `16 x 16` (Sprout Lands ground tiles).
3. Pivot: `Center`.
4. Slice & Apply.

Sekarang drag spritesheet (atau frame individual) dari Project → ke Tile Palette window.

Unity tanya: "Choose folder to save tile asset". Pilih `Assets/_Project/Sprites/Tilesets/`.

Unity bikin satu file `Tile.asset` per sprite frame yang di-drag. Sekarang Palette punya tile yang bisa di-paint.

> **Banyak tile = banyak file**. Itu memang. Tile asset kecil, fine. Kalau mau cleaner, taruh subfolder per kategori.

---

## Step 5 — Paint Ground Layer (Rumput)

Di Tile Palette window:

- **Active Tilemap** dropdown (atas) → pilih `Ground`.
- Pilih tile rumput di palette.
- Pilih tool brush (kuas) di toolbar Tile Palette.
- Di Scene view, klik & drag → paint tile.

Tools yang berguna:

- **B** — Brush (paint single tile).
- **U** — Box / Rectangle (drag rectangle, fill).
- **F** — Bucket / Flood fill.
- **G** — Picker (eyedropper, ambil tile dari yang ada).
- **D** — Eraser.

Paint area rumput sekitar 30×20 cell.

Kalau salah paint, **Shift+klik** untuk erase.

---

## Step 6 — Rule Tile untuk Rumput dengan Auto-Tiling

Default tile rumput cuma 1 sprite. Tapi Sprout Lands punya 47 variants rumput (rumput tengah, tepi atas, tepi kanan, pojok, dst). Kalau kita paint manual milih variant yang tepat, lama. **Rule Tile** otomatis pilih variant berdasarkan neighbor.

### Buat Rule Tile

Project → klik kanan `Tilesets/` → Create → **2D → Tiles → Rule Tile**. Namain `GrassRuleTile`.

Pilih `GrassRuleTile` → Inspector:

- **Default Sprite**: drag sprite rumput "tengah" (yang paling generic).
- **Default Game Object**: kosong.
- **Default Collider Type**: `None` (rumput gak collide).

Bagian **Tiling Rules**:

- Klik `+` untuk tambah rule. Tiap rule punya:
  - **Sprite**: variant sprite untuk kondisi ini.
  - **3×3 grid**: kondisi neighbor.
    - Hijau: ada tile yang sama.
    - Merah dengan X: bukan tile yang sama (atau kosong).
    - Putih: don't care.

Untuk setup penuh 47-tile, banyak rule. Tapi kamu bisa pakai versi "blob tile" yang lebih sederhana.

> **Tip**: di Asset Store ada banyak Rule Tile pre-made untuk Sprout Lands. Atau gunakan **Rule Tile Set Helper** tool (Roy-Theunissen di GitHub).

Untuk learning, buat 4 rules manual:

1. **Center**: semua 4 direct neighbor (atas, bawah, kiri, kanan) = same tile → sprite tengah rumput.
2. **Top edge**: kiri/kanan/bawah = same, atas = bukan → sprite "rumput dengan tepi atas".
3. **Bottom edge**: similar.
4. **Left/right edge**: similar.

(Demo step-by-step bikin Rule Tile penuh akan jadi terlalu panjang. Lihat [Unity docs Rule Tile](https://docs.unity3d.com/Packages/com.unity.2d.tilemap.extras@2.0/manual/RuleTile.html) untuk panduan lengkap dengan gambar.)

### Drag Rule Tile ke Palette

Drag `GrassRuleTile.asset` ke Palette window → muncul sebagai tile baru.

Sekarang paint pakai Rule Tile → tepi-tepi otomatis pakai sprite yang tepat.

> **Shortcut**: Kalau bikin Rule Tile sendiri ribet, pakai dulu **basic Tile** (1 sprite per variant), paint manual dengan brush. Pemain gak akan ngeluh kalau rumput gak punya tepi cantik. Polish later.

---

## Step 7 — Decoration Layer

Aktifkan **Decoration** sebagai active tilemap di Tile Palette.

Drag decoration sprites dari Sprout Lands ke palette (bunga, jamur, batu kecil, daun jatuh, dll).

Paint sparse — tergantung mood map. **Random Tile** asset (Create → 2D → Tiles → Random Tile) cocok di sini supaya tiap brush stroke random pilih variant.

---

## Step 8 — Obstacles Layer (with Collider)

Aktifkan **Obstacles** tilemap.

Pilih GameObject `Obstacles` → Add Component:

1. **Tilemap Collider 2D** — generate collider per tile yang di-paint.
2. **Composite Collider 2D** — gabung tile colliders adjacent jadi satu polygon (lebih efisien).
3. **Rigidbody 2D** — Body Type = `Static`.

Set:

- **Tilemap Collider 2D** → centang **Used By Composite**.
- **Composite Collider 2D** → Geometry Type: `Polygons`.

Sekarang paint tile pohon, batu, pagar di tilemap Obstacles. Player harus nabrak.

### Sprite Pohon: Pivot Bawah

Untuk pohon (yang tinggi), kita mau player bisa "berdiri di belakang" pohon — sprite pohon ke-render di atas player.

Dua hal yang harus diatur:

1. Sprite pohon di-import dengan **pivot = Bottom**. Kalau belum: di Sprite Editor → Pivot → Custom → set ke `(0.5, 0)`.
2. Tilemap Renderer Obstacles → **Mode**: `Individual`.

Tilemap default mode `Chunk` render semua tile sebagai satu mesh dengan satu Z. `Individual` mode render tiap tile terpisah, bisa sort per-tile.

3. Tambahkan komponen **Custom Axis Sort** atau pakai **2D Renderer Sort Axis**:
   - Edit → Project Settings → Graphics → **Custom Axis Sort** = `(0, 1, 0)` → Unity sort sprite by Y position (semakin bawah, semakin di depan).
   - Atau (URP): Project Settings → Graphics → Scriptable Render Pipeline Settings → 2D Renderer Data → Transparency Sort Mode = Custom Axis, Custom Axis = `(0, 1, 0)`.

Sekarang pohon yang Y-nya lebih kecil (di bawah) akan ke-render di depan player kalau player di atas. Tapi pohon di atas player (Y lebih besar) → ke belakang.

### Tile Pohon Khusus

Pohon biasanya 2 tile tinggi (kepala daun + batang). Bikin Tile asset baru:

- Project → klik kanan → Create → 2D → Tiles → Tile.
- Set Sprite ke spritesheet pohon (combined 2-tile).
- Mungkin perlu sprite di-slice ulang dengan ukuran 16×32 untuk 2-tile sekaligus.
- Atau bikin 2 tile asset (atas dan bawah), paint manual.

---

## Step 9 — Water Layer

Aktifkan **Water** tilemap.

Set komponen di GameObject Water:

- **Tilemap Collider 2D** + **Composite Collider 2D** + **Rigidbody 2D Static** (sama kayak Obstacles).
- Atau opsional: **Used By Effector** (untuk water effector swimming feature). Skip dulu.

Paint area sungai/danau.

### Animasi Air

Pakai **Animated Tile**:

- Create → 2D → Tiles → Animated Tile.
- Frames: drag sprite air frame 0, 1, 2, 3.
- Speed: 4 fps (slow ripple).

Drag ke palette, paint di Water tilemap. Air sekarang berkilauan. ✨

---

## Step 10 — Set Camera Boundary Sesuaikan

Inget `CameraBoundary` polygon dari Chapter 3? Sekarang kita tahu ukuran map sebenarnya. Adjust polygonnya untuk match tepi-tepi map farm.

> **Tip**: kalau map nanti expand ke area town, indoor, dll, kita akan punya banyak polygon boundary. Switch confiner per-area saat player masuk trigger.

---

## Step 11 — Spawn Player di Tengah Farm

Pilih Player → Inspector → ubah Position ke titik di tengah area rumput, misal `(0, 0, 0)`.

Pencet Play. Player harus bisa keliling, nabrak pohon/batu/air, gak ke-luar boundary kamera.

---

## Step 12 — Performance Tips

Tilemap besar (>100×100) kadang lag. Tips:

1. **Tilemap Renderer Mode**: pakai `Chunk` kalau gak butuh per-tile sort. Lebih cepat.
2. **Composite Collider** untuk semua collider tilemap. Mengurangi count kolider.
3. **Disable raycast** di tilemap yang gak perlu interaksi mouse.
4. **Tilemap Anchor** override untuk deal dengan rendering issue di tile besar.

Untuk farm 30×20, gak ada concern performance.

---

## Step 13 — Bikin Prefab Map Region (Opsional)

Kalau kamu mau punya beberapa "region" (Forest, Beach, Mountain), bikin prefab:

1. Buat Empty GameObject `ForestRegion`.
2. Sub-grid di dalam: Grid + Tilemaps.
3. Drag jadi prefab.

Switching antar region nanti pakai SceneManager.LoadScene atau additive scene loading. Skip dulu, kita single-scene.

---

## Troubleshooting

### Tile gak muncul saat di-paint

- Active Tilemap di Tile Palette window udah ke-set ke yang benar?
- Tile asset valid (gak corrupt)? Coba bikin baru.
- Sprite tile ke-import dengan PPU 16, Pivot Center?

### Player nembus pohon/batu

- Tilemap Obstacles punya Tilemap Collider 2D + Composite Collider 2D + Rigidbody 2D static?
- Tilemap Collider centang `Used By Composite`?
- Layer collision matrix Player vs Obstacle layer ke-enable? (Project Settings → Physics 2D)

### Player ke-render di belakang rumput

- Sorting layer Tilemap Ground = 0, Decoration = 1, Obstacles individual.
- Sorting Layer Player = `Characters` (yang di-add Chapter 2). Default Sorting Layer order: Default, Characters → Characters lebih di atas.
- Sprite Renderer Order in Layer Player = 0 atau lebih.

### Water bisa di-jalanin player

- Water Tilemap punya Collider sebagaimana Obstacles?
- Sebagai alternative: bikin water cuma visual, dan pasang **invisible Wall** di sekeliling air.

### Performance drop saat scrolling map

- Tilemap renderer mode chunk.
- Disable Composite jika gak ada collider perlu.
- Profiler (Window → Analysis → Profiler) untuk diagnose.

---

## Latihan

1. **Bikin tilemap untuk indoor rumah** — sub-grid kecil, lantai kayu, dinding.
2. **Trigger door**: bikin GameObject Trigger di depan pintu rumah, pas player masuk → pindah scene atau teleport ke posisi indoor.
3. **Random rock generator**: tulis script editor (`MenuItem`) yang random spawn batu di posisi random pada tilemap Obstacles.
4. **Tilemap brush custom**: Unity 2D Tilemap Extras punya **GameObject Brush**. Coba paint prefab pohon (yang punya animasi shake) sebagai tile.

---

## Recap

- [x] Grid + 4 Tilemap layer (Ground, Decoration, Obstacles, Water).
- [x] Tile Palette dengan asset Sprout Lands.
- [x] Rule Tile untuk auto-tiling rumput (atau manual paint).
- [x] Composite Collider untuk Obstacles & Water.
- [x] Custom Axis Sort untuk depth top-down (sprite Y-sort).
- [x] Animated Tile untuk air berkilauan.
- [x] Camera Boundary updated.

Dunia kita sudah ada. Sekarang waktunya bikin **alat-alat farming**.

---

## Lanjut

[**Chapter 5 — Tools System →**](05-tools-system.md)

[← Chapter 3](03-camera-follow.md) | [Daftar Isi](index.md)
