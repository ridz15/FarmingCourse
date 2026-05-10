# Chapter 12 — Next Steps & Resources

> "Ending course = starting your own project."

Selamat! Kamu udah punya farming game playable dengan core Stardew-style mechanics. Sekarang... mau dibawa ke mana?

Chapter ini bukan tutorial — ini **roadmap**. Pilih satu (atau beberapa) yang menarik kamu, dan dive in.

---

## Refleksi: Apa yang Sudah Kamu Pelajari

Sepanjang course ini kamu udah praktikin:

| Topik | Chapter |
|-------|---------|
| Project setup, package manager, version control | 1 |
| Input System modern, Rigidbody2D, animasi blend tree | 2 |
| Cinemachine 2D camera | 3 |
| Tilemap, Rule Tile, layered rendering, Y-sort | 4 |
| ScriptableObject pattern (data-driven design) | 5, 6 |
| Singleton pattern (event-driven managers) | 6, 8 |
| Slot-based inventory, drag-and-drop UI | 7 |
| Real-time + game-time sync, scheduling | 8 |
| Shop & dialog UI | 9 |
| JSON serialization & save versioning | 10 |
| Audio mixing, particle, polish | 11 |

Itu fondasi solid. Banyak game beda (RPG, simulation, sandbox) reuse pattern yang sama.

---

## Roadmap A: Lengkapin Stardew Clone Kamu

Versi mini tinggal 80% dari Stardew. Mau bikin lebih lengkap? Tambah ini:

### A1. Combat & Mining

- Underground "mine" dengan tilemap berbeda + procedural generation.
- Enemy AI (slime patrol, attack player).
- Sword tool: swing → deal damage to enemies.
- Player health & damage / death.

Resource: cari "Unity 2D combat tutorial" / "tile-based dungeon generator".

### A2. Fishing

- Fishing rod tool. Lempar ke water tile.
- Mini-game: timing bar, klik di green zone untuk catch.
- Fish Database: 30+ jenis dengan rarity & season/time conditions.

### A3. Cooking & Crafting

- Recipe system (CraftingRecipeSO): input items + output item.
- Cooking station GameObject + UI.
- Energy / hunger system.

### A4. NPC Sosial Sim

- Per-NPC schedule (multiple waypoints by hour).
- Heart system (gift items → naik / turun friendship).
- Per-NPC dialogue tree (multi-page, branching).
- Marriage/dating system (unlock at 8 hearts).

### A5. Animals (Cow, Chicken)

- Coop dan barn building.
- Hewan punya happiness, hunger, age.
- Daily produce (egg, milk).
- Tool baru: bucket (untuk milking), shears.

### A6. Mining & Forging

- Pickaxe untuk hancurin batu di mine. Drop ore.
- Furnace: ore + coal → bar.
- Forge: bar + tool → upgraded tool (radius +1, dll).

### A7. Buildings & Decoration

- Carpenter NPC: bayar coin + materials → build coop, barn, kandang.
- Furniture placement system.
- Multi-story house upgrade.

### A8. Festival Events

- Spring Festival, Summer Beach Day, Fall Pumpkin, Winter Stardrop.
- Trigger di hari tertentu, pindah ke "festival scene", mini-game.

---

## Roadmap B: Pivot ke Game Lain

Pondasi Stardew clone bisa jadi base untuk genre berbeda:

### B1. Top-Down Adventure (Zelda-like)

- Hapus tilling/farming, fokus combat & dungeon.
- Heart container & boss.
- Tilemap dungeon dengan room-based loading.

### B2. Cozy Resource Sim (Animal Crossing)

- Hapus combat. Semua peaceful.
- Decoration & house customization.
- Real-time clock (sync dengan jam OS).

### B3. Roguelike Shooter

- Procedural dungeon (Wave Function Collapse atau BSP).
- Gun tool dengan projectile physics.
- Permadeath: save hilang saat mati.

### B4. Tower Defense

- Tilemap untuk grid placement.
- Wave system (Time-based).
- Tower SO inheritance, similar pola Tool.

---

## Roadmap C: Multiplayer

Agak rumit, tapi achievable:

### C1. Local Co-op (Shared Screen)

- 2 PlayerInput components di scene, satu pakai keyboard, satu gamepad.
- Cinemachine Group Composer (frame both players).
- Resource sharing atau split inventory.

### C2. Online Multiplayer

- Pakai **Mirror Networking** (gratis), **Photon PUN**, atau **Unity Netcode for GameObjects**.
- Concept: Server-authoritative state. CropTileManager di server, sync ke client.
- Save di cloud / dedicated server.

Belajar dari: https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@latest

---

## Resource Belajar Lanjutan

### YouTube Channels (Inggris)

- **Brackeys** (legendary, banyak tutorial Unity 2D walaupun kanal sudah dormant).
- **Code Monkey** — tutorial Unity dengan code clean.
- **Sebastian Lague** — algorithmic / procedural generation.
- **Game Maker's Toolkit** — game design analysis.
- **Unity Official** — official tutorials & news.

### YouTube (Bahasa Indonesia)

- **Skillevel Academy**
- **Aldo Apriawan**
- **Tutorial Unity Indonesia**

### Buku

- **"Game Programming Patterns" oleh Robert Nystrom** — gratis online di gameprogrammingpatterns.com. Pattern Singleton, Observer, Component, Entity-Component-System dijelaskan dengan game examples.
- **"The Art of Game Design" oleh Jesse Schell** — lebih ke design (bukan code), tapi penting.
- **"Mostly Harmless" oleh Bert Bates** (Head First C#) — bagus untuk pemula C#.

### Komunitas

- **r/Unity3D** dan **r/gamedev** di Reddit.
- **Unity Forum** — official, banyak veteran.
- **Discord servers**: Unity Indonesia, Cherno Game Dev, Brackeys.
- **Facebook Group**: Indonesia Game Developer.
- **GitHub** — banyak open source Unity project, baca code orang lain.

### Asset Library

- **Itch.io** — banyak free pixel art & SFX.
- **Kenney.nl** — high-quality CC0 game assets.
- **OpenGameArt.org** — community-uploaded.
- **Freesound.org** — SFX & ambient.
- **Pixabay.com** — music gratis (cek lisensi).

### Tools Lanjutan

- **DOTween** (free / Pro) — tween library, bikin animasi UI mudah banget.
- **Odin Inspector** — Inspector serialization super powerful (paid).
- **Unity Test Framework** — unit testing.
- **Rider** — IDE, jauh lebih cepat dari VS Community untuk Unity.

---

## Tips Karier

Kalau kamu ingin ke arah karier game dev:

### 1. Bikin portfolio

3-5 project complete, polished, playable di browser (WebGL build di itch.io). Showcase code di GitHub.

### 2. Spesialisasi

Studio biasanya mempekerjakan untuk role specific:
- **Gameplay Programmer** — system design (yang course ini ngasih intro).
- **Tools Programmer** — bikin editor extensions.
- **Graphics / Shader Programmer**.
- **Network Programmer**.
- **AI Programmer** — pathfinding, behavior tree.

Pilih satu, dive deep.

### 3. Jam ikut!

Game jam = kompetisi 48-72 jam bikin game dari scratch. **Ludum Dare**, **Global Game Jam**, **GMTK Jam** populer. Tekanan bikin kamu cepat, dan portfolio kaya.

### 4. Open source contribute

Cari Unity package open source di GitHub, baca code, kirim PR untuk fix bug atau add fitur. CV impressive.

### 5. Networking

Twitter, LinkedIn, Discord, ikut event lokal. Industri kecil, kenalan penting.

---

## Refactor Course Project

Setelah nyelesain course, banyak hal di project ini yang bisa di-improve:

1. **Decouple singleton** dengan ServiceLocator atau dependency injection (Zenject / VContainer). Bikin testable.
2. **Tambah unit test** dengan Unity Test Framework. Test InventoryManager.Add/Remove logic, TimeManager day advance, dll.
3. **State Machine untuk Player** — Idle / Walking / SwingingTool / Sleeping / Dialogue. Ganti boolean spaghetti.
4. **Addressables** untuk lazy load asset (kalau game grow besar).
5. **Localization** — pakai Unity Localization Package. Translate ke English / Spanish / dll.
6. **ECS / DOTS** — kalau performance jadi concern (10,000+ tile farming?), refactor ke DOTS.

---

## Bagaimana Kalau Stuck?

Setiap dev, dari pemula sampai senior, stuck. Pendekatan saya:

1. **Reproduce konsisten**. Bisa bikin bug muncul tiap kali? Bagus.
2. **Isolate**. Buang code yang gak perlu, sampai cuma yang minim. Sering bug ketebak waktu kita simplify.
3. **Read error message**. Beneran baca. Bukan cuma scan. 80% error langsung kasih tahu solusinya.
4. **Print everything** (`Debug.Log`). Sebar log di tiap baris dari titik kerja sampai titik error.
5. **Ask AI** (ChatGPT / Claude) dengan **konteks lengkap** — kasih kode + error message + apa yang udah kamu coba.
6. **Sleep on it**. Beneran. 30 menit nge-stare di bug yang gak ke-solve, bangun pagi langsung kebayang.
7. **Ask community**. Stack Overflow (kalau pertanyaan general) atau Unity Forum (Unity-specific). Pertanyaan jelas, kode singkat reproducible.

---

## Pesan Terakhir

Kamu udah ngeprintilan banyak banget kode di course ini. Itu bagus. Tapi **kode yang kamu pahami > kode yang kamu copy**.

Pesan saya:

- **Bikin game-mu sendiri** sekarang. Apapun. Brutal jelek juga gak apa.
- **Selesain proyek**. 1 game finished > 10 game prototype.
- **Share progress** di sosmed. Accountability + komunitas akan support.
- **Belajar dari critique**. Kalau player bilang "kontrol jelek", jangan defensive. Tanya kenapa, fix.
- **Don't burn out**. Side project harus fun. Kalau jadi PR, istirahat.

Game dev itu marathon, bukan sprint. Pelan-pelan tapi konsisten = win.

---

## Terima Kasih

Course ini gratis dan akan terus diupdate. Kalau bermanfaat:

- ⭐ Star repo ini di GitHub.
- Share ke teman yang juga lagi belajar Unity.
- Kontribusi: typo fix, chapter tambahan, terjemahan ke bahasa lain — open Issue atau kirim PR.

Punya question, suggestion, atau mau pamer game-mu? Buka [Issue](../../issues) dengan tag `discussion`.

---

**Selamat ngoding, dan jangan lupa: have fun farming!** 🌾🌽🥕

— *FarmingCourse Team*

[← Chapter 11](11-polish-build.md) | [← Daftar Isi](../README.md)
