# Chapter 0 — Pendahuluan & Persiapan

> "Yang penting bukan seberapa jago kamu sekarang, tapi seberapa cepat kamu mau belajar."

Selamat datang! Sebelum coding satu baris pun, di chapter ini kita siapkan **mindset, tools, dan ekspektasi**. Investasi 30 menit di sini akan menghemat berjam-jam frustrasi di chapter selanjutnya.

---

## Tujuan Chapter

Setelah chapter ini, kamu akan:

- Tahu **arsitektur besar** game farming yang akan kita bangun (gambar besar di kepala kamu).
- Punya **Unity Hub + Unity 2022.3 LTS** terinstall.
- Punya **IDE** (Visual Studio / VS Code / Rider) yang sudah di-setup untuk Unity.
- Tahu cara baca **Console Unity** dan paham bedanya `Log`, `Warning`, `Error`.
- Punya **mental model**: kapan harus baca dokumentasi vs trial-and-error.

---

## Preview Akhir Course

Bayangkan ending state-nya begini:

- Kamu pencet **Play** di Unity.
- Karakter kamu spawn di farm yang acak-acakan, masih banyak rumput dan batu.
- Kamu equip **hoe** (tekan `1`) → klik tanah → tanah jadi tilled (kotak coklat).
- Equip **watering can** (`2`) → siram tanah → warnanya jadi gelap.
- Equip **seed** (`3`) → klik tanah watered → muncul sprout kecil.
- Tekan **B** untuk tidur → hari ganti, crop tumbuh ke stage berikutnya.
- 4 hari kemudian → crop matang → equip scythe → klik → masuk inventory.
- Buka **shop** di rumah trader → jual hasil panen → dapat coin.
- Beli seed baru → tanam lagi. Loop selesai.

Itu yang kita target. Bukan AAA, tapi **playable & extendable**.

---

## Arsitektur High-Level

Sebelum koding, kita pahami dulu **bagaimana sistem-sistem ini saling bicara**:

```
┌─────────────────────────────────────────────────────────────┐
│                          PLAYER                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ PlayerInput │→│ PlayerMovement │  │ PlayerToolHandler  │ │
│  └─────────────┘  └──────────────┘  └────────────────────┘ │
└────────────────────────────────────────┬────────────────────┘
                                         │ uses tool / item
                                         ▼
                         ┌──────────────────────────────┐
                         │      TOOL  (ScriptableObject) │
                         │  Hoe / Watering / Seed / ...  │
                         └─────┬────────────────┬───────┘
                               │ acts on        │ requires
                               ▼                ▼
              ┌─────────────────────┐   ┌──────────────────┐
              │  CropTileManager     │   │  Inventory       │
              │  (Dictionary<Vector3Int, CropTileData>)     │
              │                     │   │  - hotbar        │
              │  - SetTilled()      │   │  - bag           │
              │  - SetWatered()     │   └──────────────────┘
              │  - PlantSeed()      │
              │  - TickGrowth()  ←──┐
              │  - Harvest()        │
              └─────────────────────┘
                                    │ called every "new day"
                                    │
                         ┌──────────┴────────────┐
                         │   TimeManager         │
                         │   - Hour / Day / Season│
                         │   - OnDayChanged event│
                         └───────────────────────┘
```

Kalau kelihatan ribet, gak apa. Kita akan bangun **satu per satu**, dan tiap kotak baru jadi jelas waktu kita sampai chapter-nya.

Tiga prinsip arsitektur yang akan kita pegang:

1. **ScriptableObject untuk data**. Item, crop, tool — semua jadi *asset file*, bukan hard-coded di MonoBehaviour. Bisa di-edit di Inspector tanpa nge-build ulang.
2. **Event-driven communication**. Inventory tidak nge-poll TimeManager. TimeManager **broadcast** event `OnDayChanged`, sistem lain *subscribe*. Lebih bersih, mudah di-extend.
3. **Single responsibility**. Satu script, satu pekerjaan. `PlayerMovement` cuma gerakin player. `PlayerInput` cuma baca input. Tergoda nyatuin? Jangan. Capek nge-debug nanti.

---

## Setup Tools

### 1. Install Unity Hub

Unity Hub adalah launcher buat manage versi Unity Editor. Download di https://unity.com/download → install seperti aplikasi biasa.

Buka Unity Hub → login pakai akun Unity (gratis) atau Google.

### 2. Install Unity Editor 2022.3 LTS

Di Hub → tab **Installs** → tombol **Install Editor** → cari `2022.3.x LTS` (yang paling baru, misal `2022.3.40f1`).

Saat dialog *Add modules*, centang:

- ✅ **Microsoft Visual Studio Community** (kalau Windows & belum punya IDE)
- ✅ **Documentation** (offline docs, sangat berguna)
- ✅ **Build Support: Windows / Mac / Linux** sesuai OS kamu
- ✅ **WebGL Build Support** (opsional, kalau mau publish ke browser)

Klik **Install**. Tunggu 10–30 menit (size ±5 GB).

> **Note Unity 6**: Kalau kamu pakai Unity 6 (6000.0 LTS), semua kode di course ini akan jalan. Beberapa nama menu agak beda, akan ditandai.

### 3. Install IDE

**Visual Studio 2022 Community** (Windows): biasanya udah ke-install bareng Unity. Pas pertama buka script, Unity akan ngasih dialog "set as default external editor". Pilih VS Community.

**VS Code** (cross-platform, ringan): install dari https://code.visualstudio.com lalu install extension:

- C# Dev Kit (Microsoft)
- Unity (Microsoft)
- Unity Tools (Tobiah Zarlez) — opsional

**Rider** (paling powerful, berbayar tapi ada lisensi student gratis): https://www.jetbrains.com/rider/

Cara set IDE default di Unity: **Edit → Preferences → External Tools → External Script Editor**.

### 4. Install Git

Download dari https://git-scm.com → install. Kalau di Windows, centang opsi "Git Bash" supaya bisa pakai terminal unix-like.

Setup awal di terminal:

```bash
git config --global user.name "Nama Kamu"
git config --global user.email "email@kamu.com"
```

### 5. (Opsional) Editor Pixel Art

Kalau mau bikin / edit asset sendiri:

- **Piskel** — gratis, web-based, https://www.piskelapp.com
- **Aseprite** — paling pro, USD 19.99 (atau compile sendiri dari source di GitHub: gratis)
- **LibreSprite** — fork gratis Aseprite

Course ini pakai asset Sprout Lands jadi kamu **gak wajib** install ini.

---

## Quick Tour Unity Editor

Buka Unity Hub → **New Project** → pilih template **2D (Built-in Render Pipeline)** → kasih nama `FarmingCourse_Test` → **Create**.

(Ini cuma test, kita bikin project beneran di Chapter 1.)

Setelah Editor terbuka, kenali 5 panel utama:

```
┌──────────────────────────────────────────────────────────┐
│ Toolbar (Play / Pause / Step)                            │
├─────────────┬─────────────────────────┬──────────────────┤
│             │                         │                  │
│  Hierarchy  │      Scene / Game       │    Inspector     │
│             │                         │                  │
│  (semua     │  (viewport,             │  (properti       │
│   GameObject│   Scene = editor view,  │   GameObject     │
│   di scene) │   Game = preview play)  │   yang dipilih)  │
│             │                         │                  │
├─────────────┴─────────────────────────┤                  │
│                                       │                  │
│       Project (file & asset)          │                  │
│                                       │                  │
└───────────────────────────────────────┴──────────────────┘
                                  Console (di tab terpisah)
```

**Hierarchy**: list semua GameObject di Scene aktif. Klik salah satu → propertinya muncul di Inspector.

**Scene**: editor view. Pakai mouse middle-click drag untuk pan, scroll untuk zoom, tahan `Alt + LMB drag` untuk rotate (di 2D biasanya gak dipakai).

**Game**: preview yang dilihat player saat Play.

**Project**: file di `Assets/` folder.

**Inspector**: panel paling penting. Semua tweaking value game terjadi di sini.

**Console** (Window → General → Console): tempat error & log muncul. **Buka selalu** — 80% bug ketebak dari sini.

---

## Mental Model: Kapan Baca Docs?

Pemula sering stuck karena gak tahu kapan harus googling vs baca dokumentasi vs nanya orang. Aturan praktis:

1. **Error pas compile / play** → langsung baca **error message di Console**. Klik error-nya → Unity bawa kamu ke baris yang error.
2. **Method / class Unity yang asing** (misal `Vector3.MoveTowards`) → buka https://docs.unity3d.com → search. Atau di IDE, hover nama method → "Go to Definition".
3. **"Gimana cara bikin XYZ"** (misal: bikin animasi swing tool) → cari di YouTube + StackOverflow. Banyak. Tapi **jangan langsung copy** — pahami konsepnya, baru tulis ulang.
4. **Stuck > 30 menit** → buka Issue di repo ini, atau tanya di Discord Unity Indonesia / r/Unity3D.

---

## Cara Baca Console

Saat Play, Console mungkin spam pesan:

- **🔵 Log** (`Debug.Log`): info biasa, gak masalah. Bisa kamu pakai untuk debugging sendiri.
- **🟡 Warning** (`Debug.LogWarning`): bukan error, tapi sesuatu yang sebaiknya kamu perhatikan. Misal sprite gak ada Pixel Per Unit yang konsisten.
- **🔴 Error** (`Debug.LogError` atau exception): **harus dibetulin**. Game mungkin masih jalan tapi behavior-nya rusak.

Tombol toolbar Console yang harus kamu hafal:

- **Clear** — bersihkan list error (sering bikin lega).
- **Collapse** — gabungkan error yang sama.
- **Clear on Play** — auto-clear waktu pencet Play. **Aktifkan ini.**
- **Error Pause** — auto-pause game waktu ada error. Aktifkan untuk debugging serius.

Klik double error message → Editor bawa kamu langsung ke baris kode penyebabnya.

---

## Persiapan Mental

Beberapa hal yang harus kamu terima sejak awal:

1. **Kamu akan stuck. Berkali-kali.** Itu normal. Game dev = problem solving 90%, coding 10%.
2. **Compile error itu teman**. Mereka kasih tahu masalah persis di mana. Yang ngeri itu bug logika tanpa error — di situ kamu butuh `Debug.Log` skill.
3. **Iterasi > Perfeksionisme**. Lebih baik punya prototype jelek yang jalan daripada dokumen rapi tanpa kode.
4. **Save & commit sering**. `Ctrl+S` itu reflex. `git commit -m "..."` di akhir tiap session.
5. **Kalau butuh istirahat, istirahat**. Bug-stuck hari ini sering ke-solve dalam 30 detik besok pagi. Otak butuh tidur.

---

## Latihan Pemanasan

Sebelum lanjut ke Chapter 1, coba ini di project test kamu:

1. **Bikin GameObject baru**: Hierarchy → klik kanan → Create Empty → rename "Hello".
2. **Tambahkan komponen Rigidbody2D**: dengan GameObject "Hello" terpilih → Inspector → Add Component → Rigidbody 2D.
3. **Bikin script** `Hello.cs`:
   - Project window → klik kanan → Create → C# Script → nama `Hello`.
   - Drag script ke GameObject "Hello".
   - Double-click script untuk buka di IDE.
4. Tulis kode ini di `Hello.cs`:

   ```csharp
   using UnityEngine;

   public class Hello : MonoBehaviour
   {
       void Start()
       {
           Debug.Log("Halo dari Farming Course!");
       }

       void Update()
       {
           if (Input.GetKeyDown(KeyCode.Space))
           {
               Debug.Log("Spasi ditekan di frame " + Time.frameCount);
           }
       }
   }
   ```

5. Save (`Ctrl+S` di IDE). Balik ke Unity → tunggu compile (lihat lingkaran spinning di kanan bawah).
6. Pencet **Play**. Lihat Console. Tekan spacebar beberapa kali.

Kalau kamu lihat:

```
Halo dari Farming Course!
Spasi ditekan di frame 124
Spasi ditekan di frame 198
...
```

🎉 **Lulus.** Kamu sudah punya alur dasar: tulis script → Unity compile → Play → lihat hasil.

Kalau gak muncul, biasanya:

- IDE belum save (pastikan Ctrl+S).
- Script belum di-attach ke GameObject.
- Ada compile error di Console (warna merah). Klik untuk lihat detailnya.

---

## Recap

Yang sudah kamu punya:

- [x] Unity Hub + Unity 2022.3 LTS terinstall.
- [x] IDE terhubung ke Unity.
- [x] Tahu 5 panel utama Editor.
- [x] Bisa bikin GameObject + script + jalankan & lihat log.
- [x] Mindset: stuck = normal, error = teman, commit sering.

Kamu siap untuk Chapter 1.

---

## Lanjut

[**Chapter 1 — Setup Project Unity →**](01-setup-project.md)

[← Kembali ke Daftar Isi](../README.md)
