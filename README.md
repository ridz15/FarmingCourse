# Farming Course — Membuat Game Farming ala Stardew Valley dengan Unity 2D

> Course lengkap, gratis, berbahasa Indonesia, untuk membuat **game farming 2D top-down** ala *Stardew Valley* dari nol sampai bisa dimainkan, menggunakan **Unity Engine**.

![Banner](assets/banner-placeholder.md)

---

## Untuk Siapa Course Ini?

Course ini cocok kalau kamu:

- Pemula Unity yang sudah tahu cara buka Unity Hub & install Editor, tapi belum pernah bikin game utuh.
- Sudah pernah ngikutin tutorial movement / shooter sederhana, tapi pengen tahu cara nyusun **sistem game yang besar** (farming, inventory, save/load).
- Penggemar *Stardew Valley*, *Harvest Moon*, *Coral Island*, *Sun Haven* dan ingin paham cara di balik layarnya.
- Ingin belajar **arsitektur kode** yang rapi: ScriptableObject, event-driven, save system — bukan sekadar "asal jalan".

Tidak harus jago C#. Tapi minimal kamu harus tahu:

- Apa itu variable, function, class.
- Bedanya `int`, `float`, `bool`, `string`.
- Cara baca error di Console Unity.

Kalau belum, baca dulu [chapters/00-pendahuluan.md](chapters/00-pendahuluan.md) — di situ ada link gratis untuk catch-up.

---

## Apa yang Akan Kamu Bangun?

Di akhir course, kamu akan punya prototype game farming dengan fitur:

- Karakter top-down 2D dengan animasi jalan 4 arah.
- Dunia tilemap (rumput, jalan, air, pasir) dengan collider.
- **Sistem alat** (hoe, watering can, seed bag, scythe) yang bisa di-switch lewat hotbar.
- **Sistem farming**:
  - Cangkul tanah pakai hoe → jadi tanah tilled.
  - Siram pakai watering can → tanah jadi watered.
  - Tanam seed → tumbuh bertahap (4 stage) per hari.
  - Panen → masuk inventory.
- **Inventory & hotbar** dengan stacking item.
- **Time system**: jam dalam game, day/night cycle, ganti hari (sleep).
- **Seasons** (Spring, Summer, Fall, Winter) yang mempengaruhi crop.
- **NPC & shop**: beli seed, jual hasil panen.
- **Save / load** ke file JSON.
- **Audio** (music + SFX), build ke Windows/Mac/WebGL.

Lihat preview gameplay yang akan kita bangun di [chapters/00-pendahuluan.md#preview](chapters/00-pendahuluan.md#preview-akhir-course).

---

## Daftar Isi (Course Outline)

| # | Chapter | Estimasi | Output |
|---|---------|----------|--------|
| 0 | [Pendahuluan & Persiapan](chapters/00-pendahuluan.md) | 30 menit | Mindset & tools terinstall |
| 1 | [Setup Project Unity](chapters/01-setup-project.md) | 45 menit | Project 2D URP siap, package terinstall |
| 2 | [Player Character & Movement](chapters/02-player-character.md) | 1.5 jam | Karakter jalan 4 arah dengan animasi |
| 3 | [Camera Follow (Cinemachine)](chapters/03-camera-follow.md) | 30 menit | Kamera ngikutin player dengan smooth |
| 4 | [Tilemap & Membangun Dunia](chapters/04-tilemap-world.md) | 2 jam | Map farm, collider, layer rendering |
| 5 | [Tools System (Hoe, Watering Can, Seed)](chapters/05-tools-system.md) | 2 jam | Hotbar + alat aktif & swing animation |
| 6 | [Farming Mechanics](chapters/06-farming-mechanics.md) | 3 jam | Till, water, plant, grow, harvest |
| 7 | [Inventory & UI](chapters/07-inventory-system.md) | 2.5 jam | Inventory grid + hotbar + drag drop |
| 8 | [Time, Day/Night & Seasons](chapters/08-time-day-night.md) | 2 jam | Jam jalan, sleep, ganti musim |
| 9 | [Shop & NPC](chapters/09-shop-npc.md) | 2 jam | NPC shopkeeper, beli/jual item |
| 10 | [Save / Load System](chapters/10-save-load.md) | 1.5 jam | Save state ke file, load saat start |
| 11 | [Audio, Polish & Build](chapters/11-polish-build.md) | 1.5 jam | Music, SFX, build executable |
| 12 | [Next Steps & Resources](chapters/12-next-steps.md) | — | Roadmap lanjutan |

**Total estimasi**: ±20 jam belajar aktif. Bisa dicicil 1–2 chapter per hari.

---

## Cara Pakai Course Ini

1. **Ikuti urutan**. Tiap chapter dibangun di atas chapter sebelumnya. Jangan loncat.
2. **Ketik kode sendiri**, jangan copy-paste mentah. Otot otak baru terbentuk waktu kamu mengetik & error.
3. **Selesaikan latihan** di akhir tiap chapter sebelum lanjut. Kalau stuck, lihat solusi di folder [scripts/](scripts/).
4. **Commit progress kamu**. Bikin repo Git pribadi, commit di akhir tiap chapter. Nanti waktu bug, gampang revert.
5. **Tanya kalau bingung**. Buka [Issues](../../issues) di repo ini, kasih tag chapter & error message.

Format tiap chapter:

- **Tujuan Chapter** — apa yang akan kamu bisa setelah selesai.
- **Prasyarat** — chapter sebelumnya yang harus selesai.
- **Konsep Inti** — penjelasan teori sebelum koding.
- **Langkah Praktek** — step-by-step dengan screenshot description & kode.
- **Penjelasan Kode** — kenapa kode ditulis seperti itu, bukan cuma "ini kodenya".
- **Troubleshooting** — error umum & cara memperbaikinya.
- **Latihan** — soal untuk memperkuat pemahaman.
- **Recap** — ringkasan apa yang sudah dibangun.

---

## Tools yang Diperlukan

| Tool | Versi | Wajib? | Link |
|------|-------|--------|------|
| Unity Hub | terbaru | wajib | https://unity.com/download |
| Unity Editor | **2022.3 LTS** atau **6000.0 LTS** | wajib | install via Hub |
| Visual Studio Community / VS Code / Rider | apa saja | wajib | https://code.visualstudio.com |
| Git | terbaru | sangat dianjurkan | https://git-scm.com |
| Aseprite / Piskel / Photoshop | apa saja | opsional (asset sudah disiapkan) | https://www.piskelapp.com (gratis) |

> Course ini ditulis dan diuji di **Unity 2022.3 LTS** (versi 2022.3.40f1). Versi 2023+ dan Unity 6 (6000.0 LTS) juga didukung — perbedaan kecil akan ditandai dengan box `> Note Unity 6:`.

Spec PC minimum yang nyaman: 8GB RAM, SSD, GPU integrated terbaru juga cukup (game ini ringan).

---

## Asset yang Dipakai

Course ini pakai asset gratis (CC0 / royalty-free) supaya kamu bebas nge-tweak:

- **Tilemap & character**: [Sprout Lands by Cup Nooble](https://cupnooble.itch.io/sprout-lands-asset-pack) (free + paid pack)
- **Karakter alternatif**: [Modern Interiors / Exteriors by LimeZu](https://limezu.itch.io/) (free)
- **Audio**: [freesound.org](https://freesound.org) (CC0), [Open Game Art](https://opengameart.org)
- **Font**: [Pixel Operator](https://www.dafont.com/pixel-operator.font) (CC0)

Cara import & setting akan dijelaskan di [Chapter 1](chapters/01-setup-project.md#step-5-import-asset-pack).

---

## Struktur Repo

```
FarmingCourse/
├── README.md                  # File yang sedang kamu baca
├── chapters/                  # Materi course (markdown)
│   ├── 00-pendahuluan.md
│   ├── 01-setup-project.md
│   ├── ... (sampai 12)
├── scripts/                   # Kode jadi (untuk referensi kalau stuck)
│   ├── Player/
│   ├── Farming/
│   ├── Inventory/
│   └── ...
└── assets/                    # Catatan & link asset
```

---

## Filosofi & Gaya Course Ini

- **Bahasa Indonesia**, santai, tapi tetap teknis. Komentar kode boleh campur Inggris-Indonesia (sesuaikan kebiasaan kamu).
- **Why before how**. Setiap kali nulis kode, dijelasin dulu **kenapa** desainnya begitu, bukan cuma copy.
- **Beneran bisa di-build**. Tiap chapter ditutup dengan state project yang **bisa di-Play & gak crash**. Kalau Play-nya rusak, ada catatan troubleshooting.
- **Realistic shortcuts allowed**. Kita bukan bikin Stardew clone 100%. Course ini fokus ke fondasi yang bisa kamu lanjutkan sendiri.

---

## Kontribusi

Nemu typo, error, atau punya saran? Buka Issue atau kirim PR! Lihat [CONTRIBUTING.md](CONTRIBUTING.md) (akan ditambah).

---

## Lisensi

Materi course (markdown & kode contoh) di-rilis dengan lisensi **MIT**. Lihat [LICENSE](LICENSE).

Asset pihak ketiga (Sprout Lands, dll) tunduk pada lisensi masing-masing — cek halaman asset-nya.

---

## Siap?

Lanjut ke [Chapter 0 — Pendahuluan & Persiapan →](chapters/00-pendahuluan.md)

Selamat ngoding, dan have fun farming! 🌱

— *FarmingCourse Team*
