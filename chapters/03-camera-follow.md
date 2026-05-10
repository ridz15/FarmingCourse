# Chapter 3 — Camera Follow (Cinemachine)

> "Camera adalah mata pemain. Kalau matanya goyang, perutnya mual."

Player sekarang bisa jalan, tapi kalau jalannya keluar viewport, kita kehilangan dia. Kita butuh **kamera yang ngikutin player dengan smooth**, dan punya **boundary** supaya gak ke-luar tepi map.

Cinemachine bikin ini tinggal drag-drop, gak perlu nulis script kamera-tracking yang biasanya 100+ baris.

---

## Tujuan Chapter

Setelah chapter ini, kamu akan punya:

- Cinemachine Virtual Camera (CM Vcam) yang ngikutin Player.
- Camera dengan **smooth damping** (gak jerky).
- **Confiner 2D** — kamera gak keluar dari area tertentu (untuk batas map nanti).
- Pengaturan deadzone supaya kamera gak goyang waktu player gerak kecil.
- Pemahaman **Cinemachine Brain vs Vcam**.

---

## Prasyarat

- [Chapter 2](02-player-character.md) selesai (player bisa jalan).
- Package Cinemachine sudah terinstall (Chapter 1).

---

## Konsep Cinemachine

### Brain + Vcam

Cinemachine bekerja dengan dua tipe komponen:

- **CinemachineBrain** — komponen di Main Camera. Dia "otak" yang dengerin Vcam mana yang aktif.
- **Cinemachine Virtual Camera (Vcam)** — kamera "virtual" terpisah. Bisa banyak. Yang prioritas tertinggi & enabled = aktif. Brain interpolasi antar Vcam saat switching.

Kenapa ribet begitu? Karena nantinya kita mau:

- Kamera default ngikutin player.
- Saat masuk indoor (rumah) → Vcam khusus indoor (zoomed in, fixed area).
- Saat cutscene → Vcam dolly ke spot tertentu.
- Switching antar mereka mulus = built-in di Cinemachine.

### Body & Aim

Setiap Vcam punya 2 modul:

- **Body**: cara kamera bergerak. Contoh: `Framing Transposer` (lock target di area tertentu di screen, dengan damping).
- **Aim**: cara kamera diarahkan / rotasi. Untuk 2D top-down, biasanya `Do Nothing`.

---

## Step 1 — Tambah CinemachineBrain ke Main Camera

Pilih **Main Camera** di Hierarchy. Add Component → **Cinemachine Brain**.

Default settings sudah cukup:

- **Default Blend**: `EaseInOut` 2 sec — durasi transisi default antar Vcam (bisa override per-vcam).
- **Live Camera**: read-only, nanti otomatis ke-set.

---

## Step 2 — Buat Virtual Camera

Hierarchy → klik kanan → **Cinemachine → 2D Camera**. Unity bikin GameObject `CM vcam1`.

> **Note Cinemachine 3.x** (Unity 6 / new versions): nama menu jadi `Cinemachine → CinemachineCamera` (tanpa "vcam"). Step inti sama, beberapa nama property beda. Cek troubleshooting di akhir.

Di Inspector `CM vcam1`:

### Bidang `Follow`

Drag GameObject `Player` dari Hierarchy → field **Follow** di Vcam.

### Bidang `Lens`

- **Orthographic Size**: sesuaikan dengan zoom level yang kamu mau. Default 5 (vertikal 10 unit). Untuk pixel art rapat, set ke 5 atau 6.

### Body: Framing Transposer

Default Body biasanya `Framing Transposer` untuk 2D Vcam. Setting yang bagus:

- **Lookahead Time**: 0.1 (kamera sedikit "memprediksi" gerak player → terasa enak).
- **Lookahead Smoothing**: 5.
- **X Damping**: 1.5
- **Y Damping**: 1.5
- **Z Damping**: 0 (kita 2D).
- **Dead Zone Width**: 0.1
- **Dead Zone Height**: 0.1
- **Soft Zone Width**: 0.5
- **Soft Zone Height**: 0.5
- **Screen X**: 0.5 (player di tengah horizontal)
- **Screen Y**: 0.5 (player di tengah vertikal)

### Apa itu Dead Zone & Soft Zone?

- **Dead Zone**: area di tengah screen di mana player bisa gerak tanpa kamera ikut. Kalau player keluar dari dead zone, kamera mulai tracking.
- **Soft Zone**: area di luar dead zone tapi di dalam soft zone, kamera tracking dengan damping (smooth).
- **Hard zone (di luar soft)**: kamera tracking sangat agresif (instan).

Untuk Stardew-feel, dead zone kecil supaya kamera responsive tapi gak jitter saat WASD diselip-selip.

### Aim: Do Nothing

Pastikan **Aim** = `Do Nothing` (untuk 2D top-down, kamera gak rotasi).

---

## Step 3 — Test Follow

Pencet **Play**. Gerakkan player. Kamera harus ngikutin smooth. Kalau:

- Kamera gak ngikutin: cek field `Follow` di Vcam, harus diisi Player.
- Kamera ngikutin tapi pixel jitter: kembalikan ke Pixel Perfect Camera setting (Crop Frame, Pixel Snapping).
- Player keluar viewport saat Run: turunkan Damping (lebih rendah = lebih responsif).

---

## Step 4 — Add Cinemachine Pixel Perfect

Cinemachine punya extension untuk auto-respect Pixel Perfect Camera setting:

- Pilih CM vcam1 → Inspector → **Add Extension** (paling bawah) → `Cinemachine Pixel Perfect`.

Sekarang ortho size akan otomatis ke-snap ke nilai integer yang valid. Mengurangi sub-pixel jitter.

---

## Step 5 — Confiner 2D (Camera Boundary)

Kalau farm map kamu 50×50 unit, kita gak mau kamera keluar dari batas dan nampilin background hitam. **Confiner** lock kamera di dalam area tertentu.

### Buat Boundary Polygon

1. Hierarchy → klik kanan → Create Empty → namain `CameraBoundary`.
2. Posisi `(0, 0, 0)`.
3. Add Component → **Polygon Collider 2D**.
4. Inspector → centang **Is Trigger**.
5. **Edit Collider** (tombol di Inspector) → drag corner-corner polygon ke pojok-pojok map kamu (estimasi sekarang, bisa di-adjust setelah Chapter 4 tilemap selesai).

   Misal map 30×20:
   - Vertex 1: (-15, -10)
   - Vertex 2: (15, -10)
   - Vertex 3: (15, 10)
   - Vertex 4: (-15, 10)

   Klik di edge polygon → drag handle untuk pindah.

6. Layer: bikin layer baru `Boundary` lalu set ke `Boundary`. Atau biarin `Default` untuk sekarang.

### Pasang Confiner ke Vcam

- Pilih `CM vcam1`.
- Add Extension → **Cinemachine Confiner 2D** (Cinemachine 2.x) atau **Cinemachine Confiner** (3.x).
- Field **Bounding Shape 2D**: drag GameObject `CameraBoundary`.
- **Damping**: 0.5 (smooth saat hit boundary).

Test Play — coba jalan ke pojok map. Kamera harus stop di tepi.

> **Catat untuk diri sendiri**: setelah selesai bikin tilemap di Chapter 4, balik ke sini & adjust polygon `CameraBoundary` ke ukuran map sesungguhnya.

---

## Step 6 — Camera Z Position

Pastikan Vcam Z position = -10 (atau angka negatif). Default Camera 2D punya Z = -10. Kalau Vcam-nya lebih negatif/positif, masih harus ngeliat bidang Z=0 di mana sprite ada.

Cinemachine biasanya handle ini otomatis. Kalau kamu lihat warning "Camera position is in front of objects", set Vcam Z lebih negatif.

---

## Step 7 — (Opsional) Camera Shake untuk Polish

Untuk effect kayak chopping kayu, plant berhasil tumbuh, dll, kita perlu **camera shake**.

Cinemachine 2.x punya **Cinemachine Impulse**. Singkatnya:

1. Pilih CM vcam1 → Add Extension → **Cinemachine Impulse Listener**.
2. Saat mau trigger shake (misal di script harvest), bikin `CinemachineImpulseSource` di GameObject lain (misal di sprite tool), dan panggil `impulseSource.GenerateImpulse()`.

Detailnya kita bahas di Chapter 11 (Polish). Skip untuk sekarang.

---

## Step 8 — Buat ScreenManager (Setup Singleton Pattern)

Kita akan butuh akses ke kamera (untuk koordinat mouse → world misalnya) dari banyak script. Bikin script utility:

`Assets/_Project/Scripts/Utility/ScreenManager.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Utility
{
    /// <summary>
    /// Singleton untuk akses cepat ke Main Camera.
    /// Pattern: instance gak perlu manual inisialisasi -- auto-find saat
    /// Awake.
    /// </summary>
    public class ScreenManager : MonoBehaviour
    {
        public static ScreenManager Instance { get; private set; }

        public Camera MainCamera { get; private set; }

        private void Awake()
        {
            // Singleton enforcement.
            if (Instance != null && Instance != this)
            {
                Destroy(gameObject);
                return;
            }
            Instance = this;

            // Persist across scenes (opsional, bagus untuk save/load nanti).
            DontDestroyOnLoad(gameObject);

            MainCamera = Camera.main;
            if (MainCamera == null)
            {
                Debug.LogError("[ScreenManager] No Main Camera found! Tag camera as 'MainCamera'.");
            }
        }

        /// <summary>
        /// Konversi posisi mouse di screen space ke world space (z=0).
        /// </summary>
        public Vector3 MouseToWorld()
        {
            Vector3 mouse = Input.mousePosition;
            mouse.z = -MainCamera.transform.position.z; // distance to z=0 plane
            return MainCamera.ScreenToWorldPoint(mouse);
        }

        /// <summary>
        /// Cek apakah mouse di posisi screen sedang di atas UI element.
        /// Pakai sebelum process click di world supaya gak nembus UI.
        /// </summary>
        public bool IsMouseOverUI()
        {
            return UnityEngine.EventSystems.EventSystem.current != null &&
                   UnityEngine.EventSystems.EventSystem.current.IsPointerOverGameObject();
        }
    }
}
```

Drag script ini ke GameObject `_Managers` (yang udah kita bikin di Chapter 1).

> **Singleton anti-pattern caution**: terlalu banyak singleton bikin code coupling. Untuk course ini, kita pakai 4 singleton: ScreenManager, TimeManager, InventoryManager, SaveManager. Sisanya event-based.

---

## Step 9 — Test Mouse to World

Buat script kecil sementara untuk test (boleh hapus nanti):

`Scripts/Utility/MouseDebug.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Utility;

public class MouseDebug : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetMouseButtonDown(0))
        {
            Vector3 world = ScreenManager.Instance.MouseToWorld();
            Debug.Log($"Mouse clicked at world: {world}");
        }
    }
}
```

Drag ke `_Managers` GameObject. Play, klik di game view, lihat coords di Console. Harus sesuai dengan koordinat Scene view.

---

## Step 10 — (Opsional) Mini Map / Mini Camera

Stardew gak punya minimap, tapi kalau kamu mau:

- Bikin Vcam kedua dengan ortho size besar (lihat seluruh map).
- Set output ke Render Texture.
- Tampilkan Render Texture di UI Canvas pojok kanan.

Detail di Cinemachine docs. Skip untuk sekarang.

---

## Troubleshooting

### "Cinemachine Brain not found"

Main Camera harus ada CinemachineBrain component. Tambah manual.

### Vcam ada tapi kamera gak follow

- Field `Follow` Vcam terisi?
- Vcam aktif (centang di Hierarchy)?
- Priority Vcam ≥ 1?
- Ada Vcam lain dengan priority lebih tinggi yang mungkin lagi ke-aktif?

### Player gerak tapi kamera diem

- Cinemachine Brain di Camera, bukan di Vcam.
- Camera tag = `MainCamera`.

### Kamera "lompat" saat player gerak diagonal

- Lookahead Time terlalu tinggi → turunkan ke 0.05.
- Lookahead Smoothing terlalu rendah → naikan ke 5–10.

### Pixel jitter parah

- Pixel Perfect Camera + Cinemachine Pixel Perfect Extension keduanya aktif.
- Camera position Z = -10 (jangan -9.99 atau angka aneh).
- Pixels Per Unit konsisten di semua sprite.

### Cinemachine 3.x error "CinemachineConfiner2D not found"

Di Cinemachine 3.x (Unity 6+), nama berubah. Class jadi `CinemachineConfiner2D` masih ada, tapi UI menu mungkin pindah ke "Procedural" submenu. Cek docs Cinemachine 3.x.

---

## Latihan

1. **Switch antar 2 Vcam**: bikin Vcam kedua di posisi tertentu (misal di "secret area"). Set priority awal lebih rendah dari Vcam1. Saat player masuk trigger area, ubah priority Vcam2 jadi 11. Cinemachine otomatis blend.
2. **Camera zoom in/out** dengan scroll mouse: tulis script yang ubah `Lens.OrthographicSize` Vcam berdasarkan `Input.mouseScrollDelta.y`.
3. **Boundary dynamic**: bikin 2 boundary collider — satu untuk farm area, satu untuk indoor. Switch confiner-nya saat player teleport.

---

## Recap

- [x] Cinemachine Brain di Main Camera.
- [x] Vcam follow Player dengan damping smooth.
- [x] Pixel Perfect extension untuk integer scaling.
- [x] Confiner polygon untuk boundary kamera.
- [x] ScreenManager singleton untuk MouseToWorld() helper.

Player jalan, kamera nguntit. Sekarang saatnya bikin **dunia farm** untuk di-jelajahi.

---

## Lanjut

[**Chapter 4 — Tilemap & Membangun Dunia →**](04-tilemap-world.md)

[← Chapter 2](02-player-character.md) | [Daftar Isi](../README.md)
