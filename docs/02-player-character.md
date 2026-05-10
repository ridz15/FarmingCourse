# Chapter 2 — Player Character & Movement

> "First, make it move. Then make it look good. Then make it feel right."

Game farming dimulai dari karakter yang bisa jalan ke mana saja di farm. Di chapter ini kita bikin player dengan **gerakan 4 arah** (atas/bawah/kiri/kanan), **animasi idle + walk**, dan **collider** supaya nanti bisa nabrak benda.

---

## Tujuan Chapter

Setelah chapter ini selesai, kamu akan punya:

- GameObject `Player` dengan sprite character dari Sprout Lands.
- `Rigidbody2D` + `BoxCollider2D` di player.
- Script `PlayerInputHandler.cs` (baca input via Input System).
- Script `PlayerMovement.cs` (gerakkan player berdasarkan input).
- Script `PlayerAnimator.cs` (transisi animasi idle/walk per arah).
- Animator Controller dengan 8 animasi (Idle Up/Down/Left/Right + Walk Up/Down/Left/Right).
- Player bisa jalan dengan WASD/arrow keys, gak nembus tembok.

---

## Prasyarat

- [Chapter 1](01-setup-project.md) selesai (project setup, asset terimport).

---

## Konsep Inti

### Mengapa pisah Input, Movement, Animation?

Anti-pola yang sering pemula buat: satu script "PlayerController.cs" gede yang baca input, gerakin, animate, attack, dll. Awalnya enak, tapi waktu mau:

- Tambah multiplayer → susah pisah AI controller vs human controller.
- Tambah cutscene → susah disable input tanpa freeze movement.
- Debug "kenapa idle anim gak jalan" → harus baca 500 baris kode.

**Pisah jadi 3 script kecil** punya keuntungan:

1. `PlayerInputHandler` — *expose* input sebagai property/event. Gak peduli sumbernya keyboard, gamepad, atau AI.
2. `PlayerMovement` — *konsumsi* input dari handler, ubah jadi gerakan. Gak peduli darimana input datang.
3. `PlayerAnimator` — baca state dari movement (apakah lagi gerak? arahnya?), set animasi.

Pemisahan ini disebut **Single Responsibility Principle**. Ribet di awal, hemat banget di akhir.

### Rigidbody2D + Collider2D

Untuk physics 2D, kombinasi yang kita pakai:

- **Rigidbody2D** — body fisika. Jenis `Dynamic` (kita pakai ini), `Kinematic`, atau `Static`.
- **Collider2D** — bentuk untuk deteksi tabrakan. `BoxCollider2D`, `CircleCollider2D`, `CapsuleCollider2D`, dll.

Untuk top-down, kita pakai `Dynamic` Rigidbody supaya bisa kena tabrakan, **TAPI** kita matiin gravity (`Gravity Scale = 0`) karena kita gak punya konsep "atas-bawah" gravitasi di top-down.

### Input System Modern

`Input.GetKey(KeyCode.W)` masih jalan, tapi Input System baru (`InputAction`) lebih fleksibel:

- Bisa rebinding di runtime.
- Support keyboard + gamepad simultan tanpa nulis `if` panjang.
- Support multiple players.

Course ini pakai **PlayerInput component** dengan **InputAction asset**, cara yang paling user-friendly.

---

## Step 1 — Persiapkan Sprite Character

Kita pakai sprite character dari Sprout Lands. Filenya biasanya `Basic Charakter Spritesheet.png` (di Sprout Lands ada typo "Charakter" — jangan di-fix, biar match).

### Slice Sprite Sheet

Pilih file PNG character:

- Inspector → **Sprite Mode**: `Multiple` → **Apply**.
- Klik tombol **Sprite Editor** (di Inspector). Window Sprite Editor muncul.
- Di Sprite Editor: dropdown **Slice** (kiri atas) → **Type**: `Grid By Cell Size` → **Pixel Size**: `48 x 48` (cek Sprout Lands dokumentasi, biasanya 48x48 untuk character mereka. Kalau beda, lihat di README pack).
- Pivot: `Bottom`. (Pivot bawah = anchor di kaki, bagus untuk top-down karena rendering order pakai Y position.)
- Klik **Slice** → tombol **Apply** (kanan atas) → close Sprite Editor.

Sekarang sprite jadi multiple frame yang bisa dipakai untuk animasi. Klik panah expand ▶ di sprite di Project window untuk lihat frame-frame-nya, biasanya numbered `_0`, `_1`, `_2`, ...

### Layout Sprite Sheet

Sprout Lands character spritesheet biasanya layout-nya begini (per row):

| Row | Frame 0–5 |
|-----|-----------|
| 0   | Idle Down |
| 1   | Walk Down |
| 2   | Idle Up   |
| 3   | Walk Up   |
| 4   | Idle Right (kalau gak ada Idle Left, mirror saja) |
| 5   | Walk Right |
| 6+  | Tools, dll |

> Layout-nya bisa beda tergantung pack. Buka file PNG di image viewer untuk konfirmasi mata kamu sendiri.

---

## Step 2 — Buat GameObject Player

Hierarchy → klik kanan → **2D Object → Sprites → Square** → namain `Player`.

(Square cuma placeholder. Kita akan ganti spritenya.)

Set Inspector:

- Position: `(0, 0, 0)`.
- **Sprite Renderer** komponen → drag salah satu frame Idle Down dari Project ke field **Sprite**.
- **Sorting Layer**: bikin sorting layer baru `Characters` (Edit → Project Settings → Tags and Layers → Sorting Layers → klik + tambah `Characters`). Set sprite renderer Player ke layer `Characters`.
- **Order in Layer**: 0 (default).

### Tambah Komponen

Dengan `Player` terpilih, klik **Add Component**:

1. **Rigidbody 2D**:
   - Body Type: `Dynamic`.
   - Gravity Scale: `0`.
   - Constraints → Freeze Rotation **Z**: ✅ (player gak boleh muter).
   - Collision Detection: `Continuous`.
   - Sleeping Mode: `Never Sleep`.
   - Interpolation: `Interpolate` (smooth pas physics step != frame).

2. **Box Collider 2D**:
   - Untuk top-down, kolider biasanya cuma di kaki (separuh bawah sprite).
   - Klik **Edit Collider** (icon kotak) di Inspector → drag handle untuk reshape.
   - Set **Size**: `(0.6, 0.4)` (sekitar 60% lebar, 40% tinggi sprite).
   - Set **Offset**: `(0, -0.4)` (geser ke bawah).

   > Angka di atas asumsi sprite 48×48 px @ PPU 16 = 3×3 unit. Sesuaikan sampai oval keliling kaki, gak nutup kepala.

### Layer untuk Collision

Kita siapkan Layer (beda dengan Sorting Layer):

- Edit → Project Settings → Tags and Layers → **Layers** → tambah:
  - `Player`
  - `Obstacles`
  - `Interactable`

- Set GameObject Player → Inspector → **Layer** dropdown → pilih `Player`.

---

## Step 3 — Setup Input Action Asset

Untuk pakai Input System modern:

1. Project window → klik kanan di `Assets/_Project/` → Create → folder `Input`.
2. Di folder `Input/`: klik kanan → Create → **Input Actions**. Nama: `PlayerInputActions`.
3. Double-click → **Input Actions Editor** terbuka.

Di window Input Actions:

- **Action Maps** (kiri) → klik `+` → buat map: **Gameplay**.
- Dengan map Gameplay terpilih, **Actions** (tengah) → klik `+` → bikin action: **Move**.
  - Action Type: `Value`
  - Control Type: `Vector 2`
- Pada action `Move`, klik panah ▶ untuk expand → klik `+` di Bindings → pilih **Add Up/Down/Left/Right Composite** (sub-menu `2D Vector`).
  - Composite Type: `2D Vector`.
  - Mode: `Digital Normalized` (output -1 / 0 / 1, bukan analog).
  - Set bindings:
    - Up: `<Keyboard>/w`
    - Down: `<Keyboard>/s`
    - Left: `<Keyboard>/a`
    - Right: `<Keyboard>/d`
  - Tambah composite kedua untuk arrow keys: Up/Down/Left/Right = `<Keyboard>/upArrow`, dst.
  - Tambah binding biasa untuk gamepad: `<Gamepad>/leftStick` (langsung Vector2).

- Bikin action lagi: **Interact**.
  - Action Type: `Button`.
  - Binding: `<Keyboard>/space`, `<Gamepad>/buttonSouth`.

- Bikin action: **UseTool**.
  - Type: `Button`.
  - Binding: `<Mouse>/leftButton`, `<Gamepad>/rightShoulder`.

- Bikin action: **OpenInventory**.
  - Type: `Button`.
  - Binding: `<Keyboard>/i`, `<Keyboard>/tab`.

- Bikin action: **Hotbar1** sampai **Hotbar5**:
  - Type: `Button`.
  - Binding: `<Keyboard>/1` sampai `/5`.

Klik **Save Asset** (kanan atas).

### Generate C# Wrapper Class

Pilih asset `PlayerInputActions` → di Inspector:

- Centang **Generate C# Class**.
- Class Name: `PlayerInputActions` (default).
- Namespace: `FarmingCourse.Input` (atau bebas).
- Klik **Apply**.

Unity akan generate file `PlayerInputActions.cs`. Dipakai opsional — kita pakai pendekatan PlayerInput component yang lebih sederhana.

---

## Step 4 — Tambah PlayerInput Component

Kembali ke GameObject `Player`:

- Add Component → **Player Input**.
- Field **Actions**: drag asset `PlayerInputActions`.
- **Default Map**: `Gameplay`.
- **Behavior**: `Send Messages` (paling simpel; Unity panggil method `OnMove`, `OnInteract`, dll di script di GameObject yang sama).

> **Behavior** options:
> - `Send Messages`: Unity panggil method `OnXyz()` via reflection. Simpel, tapi lambat & error-prone (typo nama gak ke-detect).
> - `Broadcast Messages`: sama kayak Send tapi propagate ke children.
> - `Invoke Unity Events`: lewat UnityEvent di Inspector. Visual.
> - `Invoke C# Events`: register event handler manual. Paling fleksibel.
>
> Untuk learning, `Send Messages` udah cukup.

---

## Step 5 — Script PlayerInputHandler

Buat script di `Assets/_Project/Scripts/Player/PlayerInputHandler.cs`:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

namespace FarmingCourse.Player
{
    /// <summary>
    /// Membaca input dari PlayerInput component (Send Messages mode)
    /// dan mengekspos hasilnya sebagai property + event yang bisa
    /// dikonsumsi script lain (Movement, ToolHandler, dst).
    /// </summary>
    [RequireComponent(typeof(PlayerInput))]
    public class PlayerInputHandler : MonoBehaviour
    {
        // Input vector 2D dari WASD / left stick. Range -1..1 per axis.
        public Vector2 MoveInput { get; private set; }

        // Apakah input gerakan sedang non-zero?
        public bool IsMoving => MoveInput.sqrMagnitude > 0.01f;

        // Event yang dipancarkan satu kali saat tombol ditekan.
        public event System.Action OnInteractPressed;
        public event System.Action OnUseToolPressed;
        public event System.Action OnOpenInventoryPressed;
        public event System.Action<int> OnHotbarPressed; // index 0..4

        // ---- Callback dari PlayerInput "Send Messages" ----

        private void OnMove(InputValue value)
        {
            MoveInput = value.Get<Vector2>();
        }

        private void OnInteract(InputValue value)
        {
            if (value.isPressed) OnInteractPressed?.Invoke();
        }

        private void OnUseTool(InputValue value)
        {
            if (value.isPressed) OnUseToolPressed?.Invoke();
        }

        private void OnOpenInventory(InputValue value)
        {
            if (value.isPressed) OnOpenInventoryPressed?.Invoke();
        }

        private void OnHotbar1(InputValue v) { if (v.isPressed) OnHotbarPressed?.Invoke(0); }
        private void OnHotbar2(InputValue v) { if (v.isPressed) OnHotbarPressed?.Invoke(1); }
        private void OnHotbar3(InputValue v) { if (v.isPressed) OnHotbarPressed?.Invoke(2); }
        private void OnHotbar4(InputValue v) { if (v.isPressed) OnHotbarPressed?.Invoke(3); }
        private void OnHotbar5(InputValue v) { if (v.isPressed) OnHotbarPressed?.Invoke(4); }
    }
}
```

### Penjelasan

- `[RequireComponent(typeof(PlayerInput))]` — Unity otomatis tambah `PlayerInput` component kalau belum ada. Mencegah lupa.
- Method `OnMove(InputValue value)` — Unity panggil ini setiap kali action `Move` ada update value. `value.Get<Vector2>()` ambil value yang sesuai.
- `OnXxxPressed` events — pakai `?.Invoke()` (null-conditional) supaya gak crash kalau gak ada listener.
- `IsMoving` property — utility supaya script lain gak perlu kalkulasi sendiri.

Save script. Drag ke GameObject `Player` di Hierarchy.

---

## Step 6 — Script PlayerMovement

Buat `Assets/_Project/Scripts/Player/PlayerMovement.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Player
{
    /// <summary>
    /// Menggerakkan player via Rigidbody2D berdasarkan input
    /// dari PlayerInputHandler. Mengekspos arah hadap (FacingDirection)
    /// untuk dipakai animator.
    /// </summary>
    [RequireComponent(typeof(Rigidbody2D), typeof(PlayerInputHandler))]
    public class PlayerMovement : MonoBehaviour
    {
        [Header("Movement")]
        [SerializeField] private float moveSpeed = 4f;
        [SerializeField] private float runMultiplier = 1.5f;

        [Header("Smoothing")]
        [Tooltip("Lerp factor untuk smoothing velocity. 1 = instant, 0 = stuck.")]
        [Range(0.05f, 1f)]
        [SerializeField] private float velocitySmoothing = 1f;

        // Komponen ter-cache.
        private Rigidbody2D rb;
        private PlayerInputHandler input;

        // Arah hadap terakhir (dipakai animator + tool placement).
        // Default hadap bawah seperti Stardew.
        public Vector2 FacingDirection { get; private set; } = Vector2.down;

        // Apakah saat ini bisa bergerak? (di-disable saat dialog, cutscene, dll.)
        public bool CanMove { get; set; } = true;

        private void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            input = GetComponent<PlayerInputHandler>();
        }

        private void FixedUpdate()
        {
            if (!CanMove)
            {
                rb.velocity = Vector2.zero;
                return;
            }

            Vector2 desired = input.MoveInput.normalized * moveSpeed;

            // (Bonus) tahan Shift untuk lari — pakai Input.GetKey lama
            // karena belum kita masukkan ke Input Actions. Boleh hapus
            // kalau gak dipakai.
            if (Input.GetKey(KeyCode.LeftShift))
            {
                desired *= runMultiplier;
            }

            // Smooth velocity supaya gak instant (opsional).
            rb.velocity = Vector2.Lerp(rb.velocity, desired, velocitySmoothing);

            // Update facing direction ketika player benar-benar bergerak.
            if (input.IsMoving)
            {
                Vector2 inp = input.MoveInput;
                // Prioritas axis dengan magnitude lebih besar (cardinal direction).
                if (Mathf.Abs(inp.x) > Mathf.Abs(inp.y))
                    FacingDirection = inp.x > 0 ? Vector2.right : Vector2.left;
                else
                    FacingDirection = inp.y > 0 ? Vector2.up : Vector2.down;
            }
        }
    }
}
```

### Penjelasan

- `FixedUpdate` (bukan `Update`) — semua manipulasi physics dilakukan di sini supaya frame-rate independent.
- `rb.velocity` (bukan `rb.MovePosition` atau `transform.Translate`) — biar physics engine yang handle collision response.
- `FacingDirection` ke-update **hanya saat moving** — kalau player diem, dia tetap menghadap ke arah terakhir.
- Algoritma "prioritas axis dengan magnitude lebih besar" mengurangi flicker animasi waktu user nge-tap diagonal.
- `CanMove` property → di-toggle false oleh dialog system, cutscene, sleep menu, dll.
- `[SerializeField] private` — bidang serialized di Inspector tapi gak public (encapsulation tetap).

Save. Drag ke GameObject `Player`.

### Test Sekarang

Pencet **Play**. Tekan WASD. Player harus gerak. Kalau gak gerak, cek:

- Console untuk error.
- `PlayerInput` component → field `Actions` udah di-set ke `PlayerInputActions`.
- `PlayerInput` → Behavior = `Send Messages`.
- Layer `Player` udah di-set.

---

## Step 7 — Setup Animasi

Sekarang waktunya animasi.

### Bikin Animation Clip

Player harus terpilih di Hierarchy. Buka window **Animation** (Window → Animation → Animation, atau `Ctrl+6`).

Window Animation muncul. Kalau kosong (no animator yet), klik **Create**:

- Save dialog: simpan ke `Assets/_Project/Animations/Player/PlayerIdleDown.anim`.
- Sekaligus Unity bikin **Animator Controller** di folder yang sama. Save juga: `PlayerAnimator.controller`.

Sekarang Animator Controller `PlayerAnimator` ke-attach otomatis ke GameObject Player (cek Inspector → Animator component).

### Buat Animasi `PlayerIdleDown`

Di window Animation:

- Pastikan dropdown clip = `PlayerIdleDown`.
- Tombol **Add Property** → Sprite Renderer → Sprite → klik `+`.
- Frame 0:00 muncul, kunci property Sprite.
- Drag frame Idle Down (frame 0 dari spritesheet) ke timeline frame 0:00.
- Drag frame Idle Down kedua (kalau ada) ke 0:30 (atau pakai sample rate 8 fps untuk lebih lambat). Untuk Sprout Lands biasanya 4 frame idle.
- Set **Samples**: 8 (di kanan atas Animation window — kalau gak kelihatan, klik gear icon → Show Sample Rate).

Kalau cuma 1 frame idle, gak masalah — animasi tetap valid (statis).

### Buat Animasi Lainnya

Klik dropdown clip name di Animation window → **Create New Clip**:

- `PlayerIdleUp` — frame Idle Up dari sprite.
- `PlayerIdleLeft` — frame Idle Left.
- `PlayerIdleRight` — frame Idle Right.
- `PlayerWalkDown` — 4-6 frame walk down (sample rate 8-10 fps untuk gerakan natural).
- `PlayerWalkUp`
- `PlayerWalkLeft`
- `PlayerWalkRight`

> **Tip**: Kalau Sprout Lands gak punya idle untuk left/right, pakai frame walk pertama, atau mirror sprite (set `Sprite Renderer → Flip X`). Lebih praktis: Walk Right + flip X untuk Walk Left.

> **Catatan flip X dengan Box Collider**: Kalau kita pakai `Flip X` untuk left, kolider tetap sama (gak ke-flip). Pas-pas saja untuk symmetric character.

### Setup Animator Controller

Buka Animator window (Window → Animation → Animator). Pilih GameObject Player supaya window-nya sync.

Kamu akan lihat node-node animasi yang udah dibuat. Setup-nya bakal kayak gini:

```
[Any State] ─────┐
                 │
              ┌──▼──────────────┐
              │  Locomotion     │  (Sub-State Machine atau Blend Tree)
              │                 │
              │  ┌───────────┐  │
              │  │  Idle     │  │ (4 directions via blend tree)
              │  └───────────┘  │
              │  ┌───────────┐  │
              │  │  Walk     │  │
              │  └───────────┘  │
              └─────────────────┘
```

Untuk simplenya, kita pakai **Blend Tree** 2D Cartesian.

#### Setup Parameter

Animator → tab **Parameters** (kiri):

- `+` → Float → **MoveX**
- `+` → Float → **MoveY**
- `+` → Bool → **IsMoving**

#### Buat Blend Tree Idle

Di Animator graph: klik kanan area kosong → **Create State → From New Blend Tree**. Namain `Idle`.

Double-click `Idle` blend tree → masuk ke graph blend tree.

- Inspector → **Blend Type**: `2D Simple Directional`.
- Parameters: X = MoveX, Y = MoveY.
- Klik `+` 4 kali untuk Add Motion Field. Set:
  | Motion | PosX | PosY |
  |--------|------|------|
  | PlayerIdleDown | 0 | -1 |
  | PlayerIdleUp | 0 | 1 |
  | PlayerIdleLeft | -1 | 0 |
  | PlayerIdleRight | 1 | 0 |

> Drag clip animation ke kolom Motion-nya.

Klik tombol **Compute Position** kalau perlu auto-set posisi dari motion (opsional).

#### Buat Blend Tree Walk

Klik back arrow di breadcrumb Animator → balik ke main graph. Buat blend tree baru `Walk` dengan struktur sama, tapi pakai animasi PlayerWalkXxx.

#### Transitions

Sekarang, dengan 2 blend tree (Idle, Walk) di Animator:

- Klik kanan `Idle` → **Make Transition** → klik `Walk`.
- Klik panah transition Idle→Walk → Inspector:
  - **Has Exit Time**: ❌ uncheck (transition langsung).
  - **Transition Duration**: 0 (instant).
  - **Conditions**: `+` → `IsMoving = true`.

- Buat transition balik: Walk → Idle.
  - Has Exit Time: ❌
  - Conditions: `IsMoving = false`.

- **Default state**: klik kanan `Idle` → **Set as Layer Default State**. (Idle harus default biar gak crash kalau MoveX/Y belum di-set.)

Save scene.

---

## Step 8 — Script PlayerAnimator

Buat `Assets/_Project/Scripts/Player/PlayerAnimator.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Player
{
    /// <summary>
    /// Menerjemahkan state PlayerMovement (FacingDirection, IsMoving)
    /// ke parameter Animator Controller.
    /// </summary>
    [RequireComponent(typeof(Animator), typeof(PlayerMovement), typeof(PlayerInputHandler))]
    public class PlayerAnimator : MonoBehaviour
    {
        // Hash parameter (lebih cepat dari string).
        private static readonly int MoveX_Hash = Animator.StringToHash("MoveX");
        private static readonly int MoveY_Hash = Animator.StringToHash("MoveY");
        private static readonly int IsMoving_Hash = Animator.StringToHash("IsMoving");

        private Animator animator;
        private PlayerMovement movement;
        private PlayerInputHandler input;

        private void Awake()
        {
            animator = GetComponent<Animator>();
            movement = GetComponent<PlayerMovement>();
            input = GetComponent<PlayerInputHandler>();
        }

        private void Update()
        {
            // Kirim facing direction sebagai MoveX/MoveY, supaya
            // blend tree pakai arah hadap untuk pilih frame yang tepat
            // (baik saat walk maupun idle).
            Vector2 facing = movement.FacingDirection;
            animator.SetFloat(MoveX_Hash, facing.x);
            animator.SetFloat(MoveY_Hash, facing.y);
            animator.SetBool(IsMoving_Hash, input.IsMoving);
        }
    }
}
```

### Penjelasan

- `Animator.StringToHash(...)` → cache string jadi int. Pemanggilan `SetFloat(int, float)` lebih cepat dari `SetFloat(string, float)`. Best practice.
- Kita kirim `FacingDirection` (bukan `MoveInput`) karena saat player diem (`IsMoving=false`), MoveInput = 0 → blend tree fallback ke center. Dengan FacingDirection, idle anim tetap menghadap arah terakhir.

Save. Drag ke GameObject Player.

### Test Animasi

Pencet **Play**. WASD untuk gerak, lepas untuk idle. Animasinya harus:

- Walk down saat S/Down arrow.
- Walk up saat W/Up.
- Walk left/right.
- Idle setelah lepas — menghadap arah terakhir.

Kalau frame idle salah arah, cek kembali blend tree position values (PosX, PosY).

---

## Step 9 — Buat Tembok untuk Test Collision

Bikin GameObject test:

1. Hierarchy → 2D Object → Sprites → Square → namain `Wall`.
2. Set posisi `(3, 0, 0)`.
3. Inspector → Add Component → **Box Collider 2D**.
4. **Layer**: `Obstacles`.

Pencet Play → coba tabrak Wall → Player harus berhenti.

Kalau player nembus, cek:

- Wall punya **BoxCollider2D**? Yes.
- Player **Rigidbody2D** Body Type = **Dynamic**? Yes.
- Project Settings → Physics 2D → Layer Collision Matrix → Player vs Obstacles centang? Yes.

---

## Step 10 — Bonus: Footstep Audio (Optional, Skip kalau Belum)

Kita bahas detailnya di Chapter 11 (Audio). Untuk sekarang, lewati. Yang penting movement & animasi udah jalan.

---

## Step 11 — Save sebagai Prefab

GameObject Player akan dipakai di banyak scene + bisa kena reset waktu kita load save game. Lebih aman jadiin prefab.

1. Drag GameObject `Player` dari Hierarchy ke `Assets/_Project/Prefabs/Player/` di Project window.
2. Unity bikin file `Player.prefab`. GameObject di Hierarchy berubah jadi biru = sekarang prefab instance.

Save scene (`Ctrl+S`).

---

## Troubleshooting

### "Move action doesn't trigger OnMove method"

- `PlayerInput` Behavior = `Send Messages`? Yes.
- Method namanya `OnMove` (camelCase, persis sama dengan action name)? Yes.
- Method-nya `private` atau `public`, gak masalah.
- Action Map yang aktif `Gameplay`? Cek di Inspector PlayerInput → `Default Map`.

### Player gerak tapi sprite gak ke-flip / animasi salah

- Cek blend tree position di animator window.
- Cek script PlayerAnimator beneran nge-set MoveX/MoveY.
- Pakai window **Animator** saat Play untuk lihat parameter realtime — kalau MoveX cuma berubah jadi 0 ke ±1 sangat cepat, oversampling. Pakai `velocitySmoothing` lebih tinggi.

### Player gak gerak diagonal mulus, jadi cuma cardinal

- Itu fitur, bukan bug. `FacingDirection` selalu cardinal (atas/bawah/kiri/kanan), tapi `MoveInput.normalized * speed` boleh diagonal. Sprite tetap menghadap salah satu arah dominan.

### Player terbang ke atas / jatuh

- `Rigidbody2D → Gravity Scale = 0`.
- Body Type ≠ `Static`.

### Sprite blink / flicker pas jalan

- Pixel Perfect Camera → Pixel Snapping ✅ + Crop Frame ✅.
- Camera Follow di Chapter 3 akan mengurangi ini.

---

## Latihan

1. **Tambah action `Run`** ke Input Actions: hold `Shift` atau gamepad `B` button. Tambah event di `PlayerInputHandler` dan flag `IsRunning`. Ganti `Input.GetKey(LeftShift)` di `PlayerMovement` jadi pakai `input.IsRunning`. Cleaner!
2. **Animasi `PlayerRun`** (kalau spritesheet ada). Bikin blend tree ketiga `Run`, transisi Idle ↔ Walk ↔ Run dengan parameter `Speed` (float). Set MoveSpeed dinamis.
3. **Footstep particle / dust effect** — bikin partikel kecil di kaki saat gerak. Pakai Particle System + simple emit (lihat docs Unity).
4. **Dash skill** — tap `Space` untuk dash 1 unit cepat ke arah hadap. Tambah cooldown 1 detik.

---

## Recap

Yang sudah jalan:

- [x] Player GameObject dengan Rigidbody + Collider.
- [x] Input System dengan PlayerInputActions asset.
- [x] Script terpisah: Input, Movement, Animator.
- [x] Blend tree 2D untuk Idle dan Walk × 4 arah.
- [x] Player jalan dengan WASD/arrow, smooth, gak nembus tembok.
- [x] Prefab Player tersimpan.

Player kita sudah bisa keluyuran. Sekarang kameranya harus ngikutin.

---

## Lanjut

[**Chapter 3 — Camera Follow (Cinemachine) →**](03-camera-follow.md)

[← Chapter 1](01-setup-project.md) | [Daftar Isi](index.md)
