# Chapter 5 — Tools System (Hoe, Watering Can, Seeds)

> "Stardew Valley = Tools + Time. Bikin tool fondasinya benar, gameplay-nya tinggal isi."

Player jalan di dunia farm. Sekarang dia butuh alat: **hoe** (cangkul), **watering can** (siram), **seed bag** (tanam), **scythe** (panen), dan nanti **axe / pickaxe** (kayu/batu). Di chapter ini kita bikin **arsitektur tools yang scalable**.

---

## Tujuan Chapter

Setelah chapter ini:

- Sistem **Item** berbasis ScriptableObject (tool, seed, crop, dll).
- Class hierarchy: `ItemSO` → `ToolSO` (inherit) → `HoeToolSO`, `WateringCanSO`, `SeedSO`, `ScytheSO` (subclass).
- **Hotbar** (5 slot) dengan select via `1-5` keys.
- Player bisa **swing** tool ke arah hadap (animasi swing).
- Tool punya **action target tile** — tile depan player.
- Visual indicator: kotak kuning di tile yang akan kena tool.
- Persiapan untuk Chapter 6 di mana action sebenarnya (till, water, plant) terjadi.

---

## Prasyarat

- [Chapter 4](04-tilemap-world.md) selesai.

---

## Konsep ScriptableObject (SO)

### Apa itu ScriptableObject?

`ScriptableObject` (`SO`) adalah class kayak `MonoBehaviour`, tapi:

- Gak attach ke GameObject. Disimpan sebagai `*.asset` file di `Assets/`.
- Dipakai untuk **data**. Contoh: definisi tool, definisi crop, save data, dll.
- Bisa diedit di Inspector, tanpa nge-build ulang.
- Reusable: 100 GameObject bisa share 1 instance SO.

### Kenapa pakai SO untuk Item?

Bandingin:

**Cara hard-coded** (jelek):

```csharp
public class Hoe : MonoBehaviour
{
    public string itemName = "Hoe";
    public Sprite icon;
    public int durability = 100;
}
```

Setiap variant tool = MonoBehaviour baru. Item baru = perubahan kode.

**Cara SO** (bagus):

```csharp
[CreateAssetMenu(fileName = "Hoe", menuName = "FarmingCourse/Items/Hoe")]
public class HoeToolSO : ToolSO
{
    public int tillRadius = 1;
    // ... config khusus hoe
}
```

Tinggal klik kanan di Project → Create → bikin `Hoe.asset`, `BronzeHoe.asset`, dll. Item baru = bikin asset baru, gak perlu kode baru.

---

## Step 1 — Bikin Base Class `ItemSO`

`Assets/_Project/Scripts/Inventory/ItemSO.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Inventory
{
    /// <summary>
    /// Base class untuk semua item di game (tools, seeds, crops, gifts, dll).
    /// Subclass-nya: ToolSO, SeedSO, CropSO.
    /// </summary>
    public abstract class ItemSO : ScriptableObject
    {
        [Header("Base Item")]
        public string itemId;          // unique id, ex: "hoe", "carrot_seed"
        public string displayName;     // ex: "Hoe", "Carrot Seeds"
        [TextArea(2, 4)]
        public string description;
        public Sprite icon;            // ikon UI 16x16
        public int maxStackSize = 99;  // jumlah max per stack di slot inventory
        public int sellPrice = 0;      // harga jual ke shop. 0 = gak bisa dijual.
        public int buyPrice = 0;       // harga beli di shop. 0 = gak dijual.

        /// <summary>
        /// Override jika item bisa di-"use" (tool, seed, dll).
        /// Default: tidak bisa.
        /// </summary>
        public virtual bool CanUse => false;
    }
}
```

### Penjelasan

- `abstract` — gak bisa diinstantiate langsung. Harus inherit subclass.
- `[Header(...)]` — kasih label section di Inspector.
- `[TextArea(...)]` — multiline string field di Inspector.
- `virtual bool CanUse` — subclass bisa override.

---

## Step 2 — Bikin `ToolSO`

`Assets/_Project/Scripts/Inventory/ToolSO.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Inventory
{
    /// <summary>
    /// Base untuk semua tool yang bisa di-swing (hoe, watering can, axe, dll).
    /// </summary>
    public abstract class ToolSO : ItemSO
    {
        [Header("Tool")]
        public AnimationClip swingAnimation;   // clip swing (per direction nanti override).
        public AudioClip useSfx;
        public float useDuration = 0.4f;       // berapa lama swing animation.
        public int staminaCost = 2;            // (untuk system stamina nanti)

        public override bool CanUse => true;

        /// <summary>
        /// Action utama saat tool digunakan.
        /// `userPosition` = posisi player.
        /// `targetTile` = tile depan player (dihitung di PlayerToolHandler).
        /// </summary>
        /// <returns>true kalau action sukses.</returns>
        public abstract bool Use(Vector3 userPosition, Vector3Int targetTile);
    }
}
```

### Penjelasan

- `abstract bool Use(...)` — subclass **wajib** implement. Bukan virtual yang opsional.
- Method `Use` return `bool` — kasih tahu sukses (bisa untuk play SFX) atau gagal (target invalid).

---

## Step 3 — Bikin `HoeToolSO`

`Assets/_Project/Scripts/Inventory/Tools/HoeToolSO.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Farming; // forward reference -- kita bikin nanti

namespace FarmingCourse.Inventory.Tools
{
    [CreateAssetMenu(fileName = "Hoe", menuName = "FarmingCourse/Items/Tools/Hoe")]
    public class HoeToolSO : ToolSO
    {
        [Header("Hoe-Specific")]
        public int tillRadius = 0;  // 0 = single tile, 1 = 3x3 area (upgrade)

        public override bool Use(Vector3 userPosition, Vector3Int targetTile)
        {
            // Forward call ke CropTileManager (akan kita bikin di Chapter 6).
            // Untuk sekarang, sementara kita print log saja supaya bisa di-test.
            // Nanti baris ini diganti.
            if (CropTileManager.Instance == null)
            {
                Debug.LogWarning("[HoeToolSO] CropTileManager not in scene yet.");
                return false;
            }
            return CropTileManager.Instance.TryTill(targetTile, tillRadius);
        }
    }
}
```

> Compile error karena `CropTileManager` belum ada — wajar. Kita bikin di Chapter 6. Untuk sementara, **comment** baris yang reference `CropTileManager`:

```csharp
        public override bool Use(Vector3 userPosition, Vector3Int targetTile)
        {
            Debug.Log($"[Hoe] Tilling tile {targetTile} (radius {tillRadius})");
            return true;
            // TODO: kembalikan saat Chapter 6:
            // return CropTileManager.Instance.TryTill(targetTile, tillRadius);
        }
```

Save. Compile harus bersih.

---

## Step 4 — Bikin `WateringCanToolSO`

`Assets/_Project/Scripts/Inventory/Tools/WateringCanToolSO.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Inventory.Tools
{
    [CreateAssetMenu(fileName = "WateringCan", menuName = "FarmingCourse/Items/Tools/WateringCan")]
    public class WateringCanToolSO : ToolSO
    {
        [Header("Watering Can-Specific")]
        [Range(1, 100)] public int maxWater = 40;   // kapasitas
        public int waterPerUse = 1;                  // 1 tile per swing
        public int waterRadius = 0;                  // upgrade

        // Field runtime (gak di-Inspector, gak di-save).
        // Karena SO instance shared, kita simpan currentWater di sini supaya
        // semua reference dapat info yang sama. Tapi: SAVE kita harus persist
        // ini ke file save (Chapter 10).
        [System.NonSerialized] public int currentWater = 0;

        public override bool Use(Vector3 userPosition, Vector3Int targetTile)
        {
            if (currentWater <= 0)
            {
                Debug.Log("[WateringCan] Empty! Refill at water source.");
                return false;
            }

            // TODO Chapter 6:
            // bool success = CropTileManager.Instance.TryWater(targetTile, waterRadius);
            // if (success) currentWater -= waterPerUse;
            // return success;

            Debug.Log($"[WateringCan] Watering tile {targetTile}");
            currentWater -= waterPerUse;
            return true;
        }

        public void Refill()
        {
            currentWater = maxWater;
        }
    }
}
```

> **Catatan tentang `[System.NonSerialized]`**: Field SO yang kita ubah di runtime **tetap** akan ke-save oleh Unity di Editor (write-back ke file SO!). Ini behavior aneh — di runtime build mode (built executable), perubahan SO **tidak** persist. Tapi di Editor Play mode, dia persist. Bisa jadi confusing. Pakai `[System.NonSerialized]` untuk eksplisit mengatakan "field ini gak ter-serialize".
>
> Untuk save game beneran, kita pakai SaveManager terpisah di Chapter 10.

---

## Step 5 — Bikin `SeedSO`

`Assets/_Project/Scripts/Inventory/SeedSO.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Inventory
{
    [CreateAssetMenu(fileName = "Seed", menuName = "FarmingCourse/Items/Seed")]
    public class SeedSO : ItemSO
    {
        [Header("Seed")]
        public CropSO cropToPlant;   // referensi ke crop yang dihasilkan
        public int growthDays;       // total hari sampai matang

        public override bool CanUse => true;

        public bool Plant(Vector3Int targetTile)
        {
            // TODO Chapter 6:
            // return CropTileManager.Instance.TryPlant(targetTile, this);
            Debug.Log($"[Seed] Planting {cropToPlant?.displayName ?? "?"} at {targetTile}");
            return true;
        }
    }
}
```

`CropSO` belum ada — kita bikin sekarang.

`Assets/_Project/Scripts/Farming/CropSO.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Inventory;

namespace FarmingCourse.Farming
{
    /// <summary>
    /// Definisi crop: nama, sprite per stage, hasil panen, season ditanamnya.
    /// </summary>
    [CreateAssetMenu(fileName = "Crop", menuName = "FarmingCourse/Crops/Crop")]
    public class CropSO : ScriptableObject
    {
        [Header("Crop")]
        public string displayName;
        public Sprite[] growthStages;     // 4-5 sprite, dari sprout ke matang
        public int daysToGrow;            // total hari (stage diabagi-bagi sama rata)

        [Header("Harvest")]
        public ItemSO harvestItem;        // item yang masuk inventory saat panen
        public int harvestAmount = 1;
        public bool regrowsAfterHarvest;  // true = strawberry, kembali stage tertentu
        public int regrowStage = 2;       // kalau regrow, balik ke stage ini
        public int regrowDays = 3;        // hari untuk grow lagi setelah harvest

        [Header("Season")]
        public Season[] validSeasons;     // bisa ditanam di season-season ini
    }

    public enum Season
    {
        Spring,
        Summer,
        Fall,
        Winter
    }
}
```

---

## Step 6 — Bikin `ScytheToolSO` (Untuk Panen)

`Assets/_Project/Scripts/Inventory/Tools/ScytheToolSO.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Inventory.Tools
{
    [CreateAssetMenu(fileName = "Scythe", menuName = "FarmingCourse/Items/Tools/Scythe")]
    public class ScytheToolSO : ToolSO
    {
        public override bool Use(Vector3 userPosition, Vector3Int targetTile)
        {
            // TODO Chapter 6:
            // return CropTileManager.Instance.TryHarvest(targetTile);
            Debug.Log($"[Scythe] Harvesting tile {targetTile}");
            return true;
        }
    }
}
```

---

## Step 7 — Bikin Asset SO

Sekarang buat asset-nya:

1. Project → klik kanan `Assets/_Project/ScriptableObjects/Tools/` → Create → FarmingCourse → Items → Tools → **Hoe**.
2. Asset `Hoe.asset` muncul. Klik → di Inspector:
   - Item Id: `hoe`
   - Display Name: `Hoe`
   - Description: `Used to till soil into farmable plots.`
   - Icon: drag sprite hoe dari Sprout Lands (cari sprite tool, biasanya di sprite sheet "Tools.png").
   - Max Stack Size: 1 (tool gak stack).
   - Sell Price: 0 (gak bisa dijual)
   - Use Duration: 0.4
   - Stamina Cost: 2
   - Till Radius: 0

3. Buat `WateringCan.asset`:
   - Icon: sprite watering can.
   - Max Water: 40, Water Per Use: 1, Water Radius: 0.
   - Stack 1, Sell 0, Buy 2000.

4. Buat `Scythe.asset`:
   - Standar tool, Stack 1.

### Bikin Crop & Seed Asset

5. `Assets/_Project/ScriptableObjects/Crops/` → klik kanan → Create → FarmingCourse → Crops → **Crop** → namain `Carrot`.
   - Display Name: `Carrot`.
   - Days To Grow: 4.
   - Growth Stages: drag 4 sprite carrot stage (dari Sprout Lands sprite "Crops.png", slice dulu kalau perlu).
   - Harvest Item: kosong (kita isi setelah bikin item carrot).
   - Harvest Amount: 1.
   - Regrows After Harvest: false.
   - Valid Seasons: Spring, Summer.

6. `Assets/_Project/ScriptableObjects/Items/` → bikin asset `CarrotItem` dengan **ItemSO** (perlu sub-class non-abstract, lihat note di bawah).

   Tunggu — `ItemSO` `abstract`! Gak bisa bikin asset langsung. Kita bikin subclass simpel `BasicItemSO`:

   `Assets/_Project/Scripts/Inventory/BasicItemSO.cs`:

   ```csharp
   using UnityEngine;

   namespace FarmingCourse.Inventory
   {
       /// <summary>
       /// Item biasa yang gak punya behavior khusus (carrot, log, stone, dll).
       /// </summary>
       [CreateAssetMenu(fileName = "Item", menuName = "FarmingCourse/Items/Basic Item")]
       public class BasicItemSO : ItemSO
       {
           // No-op. Kosong. Cukup untuk grocery items.
       }
   }
   ```

   Sekarang bisa Create → FarmingCourse → Items → Basic Item → namain `Carrot`. Set icon, stack 99, sell 35.

7. Balik ke `Carrot.asset` (CropSO) → drag `Carrot` (BasicItemSO) ke field **Harvest Item**.

8. `Assets/_Project/ScriptableObjects/Items/` → Create → FarmingCourse → Items → **Seed** → namain `CarrotSeed`.
   - Display Name: `Carrot Seeds`.
   - Crop To Plant: drag `Carrot` (CropSO).
   - Growth Days: 4.
   - Stack 99, Buy 30, Sell 5.

Mantap, sekarang kita punya 1 alur tools + crop + seed.

---

## Step 8 — Hotbar (Inventory Sederhana untuk Tools)

Sebelum chapter 7 (full inventory), kita bikin **hotbar singleton** mini untuk simpan 5 slot tool/item yang aktif.

`Assets/_Project/Scripts/Inventory/Hotbar.cs`:

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

namespace FarmingCourse.Inventory
{
    /// <summary>
    /// Hotbar 5 slot, simpan ItemSO + count.
    /// Singleton sementara -- nanti di Chapter 7 di-merge dengan InventoryManager
    /// yang lebih lengkap.
    /// </summary>
    public class Hotbar : MonoBehaviour
    {
        public static Hotbar Instance { get; private set; }

        [System.Serializable]
        public class Slot
        {
            public ItemSO item;
            public int count;

            public bool IsEmpty => item == null || count <= 0;
        }

        public const int SlotCount = 5;

        [SerializeField] private List<Slot> slots = new List<Slot>();
        [SerializeField] private int selectedIndex = 0;

        public Slot SelectedSlot => slots[selectedIndex];
        public ItemSO SelectedItem => SelectedSlot.IsEmpty ? null : SelectedSlot.item;
        public int SelectedIndex => selectedIndex;

        public event Action OnHotbarChanged;
        public event Action<int> OnSelectionChanged;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;

            // Init 5 empty slot.
            while (slots.Count < SlotCount) slots.Add(new Slot());
        }

        public void SetSlot(int index, ItemSO item, int count)
        {
            if (index < 0 || index >= SlotCount) return;
            slots[index].item = item;
            slots[index].count = count;
            OnHotbarChanged?.Invoke();
        }

        public void Select(int index)
        {
            if (index < 0 || index >= SlotCount) return;
            selectedIndex = index;
            OnSelectionChanged?.Invoke(index);
        }

        public bool ConsumeFromSelected(int amount = 1)
        {
            if (SelectedSlot.IsEmpty || SelectedSlot.count < amount) return false;
            SelectedSlot.count -= amount;
            if (SelectedSlot.count <= 0) SelectedSlot.item = null;
            OnHotbarChanged?.Invoke();
            return true;
        }
    }
}
```

Drag script ini ke `_Managers` GameObject.

---

## Step 9 — Setup Default Hotbar di Editor

Pilih `_Managers` di Hierarchy → di Inspector, expand **Hotbar** component → **Slots**:

- Slot 0: drag `Hoe.asset` ke field Item, count = 1.
- Slot 1: `WateringCan.asset`, count 1.
- Slot 2: `CarrotSeed.asset`, count 10.
- Slot 3: `Scythe.asset`, count 1.
- Slot 4: kosong.

Kalau saat Play dan kamu ubah-ubah slot, ya... default-nya tetap dari Inspector. Nanti di Save System (Chapter 10) kita persist ini.

---

## Step 10 — Script `PlayerToolHandler`

Yang bertanggung jawab: dengar input UseTool, ambil item dari hotbar, panggil `Use` SO-nya, ke target tile depan player.

`Assets/_Project/Scripts/Player/PlayerToolHandler.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Inventory;

namespace FarmingCourse.Player
{
    /// <summary>
    /// Saat player tekan UseTool (left click / R-shoulder),
    /// ambil item dari hotbar -> kalau tool/seed, panggil Use() di tile target.
    /// </summary>
    [RequireComponent(typeof(PlayerInputHandler), typeof(PlayerMovement))]
    public class PlayerToolHandler : MonoBehaviour
    {
        [Header("Targeting")]
        [SerializeField] private float tileSize = 1f;
        [SerializeField] private LayerMask interactableLayer;
        [SerializeField] private GameObject targetIndicatorPrefab; // optional visual

        private PlayerInputHandler input;
        private PlayerMovement movement;
        private PlayerAnimator animator;
        private GameObject targetIndicator;

        private void Awake()
        {
            input = GetComponent<PlayerInputHandler>();
            movement = GetComponent<PlayerMovement>();
            animator = GetComponent<PlayerAnimator>();

            if (targetIndicatorPrefab != null)
            {
                targetIndicator = Instantiate(targetIndicatorPrefab);
                targetIndicator.SetActive(false);
            }
        }

        private void OnEnable()
        {
            input.OnUseToolPressed += HandleUseTool;
            input.OnHotbarPressed += HandleHotbar;
        }

        private void OnDisable()
        {
            input.OnUseToolPressed -= HandleUseTool;
            input.OnHotbarPressed -= HandleHotbar;
        }

        private void Update()
        {
            UpdateTargetIndicator();
        }

        private Vector3Int GetTargetCell()
        {
            // Tile di depan player berdasarkan FacingDirection.
            Vector3 facing = movement.FacingDirection;
            Vector3 worldPos = transform.position + facing * tileSize;
            // Convert ke cell coordinate (asumsi 1 unit = 1 tile, tilemap origin di 0,0).
            return Vector3Int.FloorToInt(worldPos);
        }

        private void UpdateTargetIndicator()
        {
            if (targetIndicator == null) return;

            ItemSO selected = Hotbar.Instance != null ? Hotbar.Instance.SelectedItem : null;
            bool show = selected != null && selected.CanUse;

            targetIndicator.SetActive(show);

            if (show)
            {
                Vector3Int cell = GetTargetCell();
                targetIndicator.transform.position = new Vector3(cell.x + 0.5f, cell.y + 0.5f, 0);
            }
        }

        private void HandleHotbar(int index)
        {
            if (Hotbar.Instance == null) return;
            Hotbar.Instance.Select(index);
        }

        private void HandleUseTool()
        {
            if (Hotbar.Instance == null) return;
            ItemSO item = Hotbar.Instance.SelectedItem;
            if (item == null || !item.CanUse) return;

            Vector3Int target = GetTargetCell();
            bool success = false;

            // Dispatch berdasarkan tipe item.
            switch (item)
            {
                case ToolSO tool:
                    success = tool.Use(transform.position, target);
                    break;
                case SeedSO seed:
                    success = seed.Plant(target);
                    if (success) Hotbar.Instance.ConsumeFromSelected(1);
                    break;
            }

            if (success)
            {
                // TODO Chapter 6: trigger animasi swing
                // Untuk sekarang: log sukses
                Debug.Log($"[PlayerToolHandler] Used {item.displayName} on {target}");
            }
        }
    }
}
```

### Penjelasan

- `OnEnable` / `OnDisable` subscribe & unsubscribe event — pattern aman dari memory leak.
- `GetTargetCell` ambil tile di depan player. Kita asumsi origin tilemap (0,0) di world (0,0). Kalau tilemap kamu offset, perlu adjust.
- `switch (item)` — pattern matching C# 7+. Jalankan branch berdasarkan tipe runtime item.
- `Hotbar.Instance.ConsumeFromSelected(1)` saat plant seed — kurangi 1 dari hotbar slot.

Save. Drag ke GameObject Player.

---

## Step 11 — Tile Target Indicator (Visual)

Bikin GameObject Prefab indicator:

1. Hierarchy → 2D Object → Sprites → Square → namain `TargetIndicator`.
2. Inspector:
   - Sprite: drag sprite kotak kuning (bisa pakai sprite "highlight" dari Sprout Lands, atau bikin sendiri di Piskel: 16×16 kotak transparent dengan border kuning).
   - **Color**: warna kuning translucent (alpha ~0.5).
   - Sorting Layer: `Default` Order 5 (atas tilemap).
   - Hapus collider kalau ada.

3. Drag GameObject ke folder `Prefabs/UI/` di Project → bikin prefab.
4. Hapus dari Hierarchy.
5. Pilih Player → Inspector → field **Target Indicator Prefab** di PlayerToolHandler component → drag prefab.

Pencet Play. Pas hotbar selected = Hoe (slot 0 default), ada kotak kuning di depan player. Jalan, kotak ngikut. Pencet 2, hotbar Watering Can, kotak masih ada. Pencet 5, hotbar empty, kotak hilang.

Klik kiri → Console log "[Hoe] Tilling tile (1, 0, 0)" atau sesuai posisi.

🎉 **Tool system jalan!**

---

## Step 12 — Animasi Swing (Persiapan, Implementasi di Chapter 6)

Idealnya, swing animation mainkan animasi player swing ke arah hadap, lalu trigger callback "swing apex" → eksekusi `tool.Use()`.

Kita simplenya dulu: panggil `Use()` immediately. Polish animasi swing di Chapter 6 atau 11.

Untuk preview: bikin trigger di Animator Controller → buat parameter `Trigger Swing`, transition dari Walk/Idle ke state Swing (misal `PlayerSwingDown.anim`). Set Has Exit Time = true untuk balik ke Walk/Idle.

Trigger di kode:

```csharp
animator.SetTrigger("Swing"); // (hash-cached)
```

---

## Step 13 — Integrasikan Hotbar UI

Untuk preview UI, bikin GameObject Hotbar:

1. Hierarchy → UI → Canvas. Set Render Mode: Screen Space - Overlay.
2. Canvas → klik kanan → UI → Image. Namain `HotbarPanel`. Set anchor bawah-tengah, height 80, width 400.
3. Di HotbarPanel: 5 child Image (slot), pakai Horizontal Layout Group untuk auto-align.
4. Tiap slot: child Image dalam (untuk icon item) + child Text TMP (untuk count).

Skrip untuk update visual (`Scripts/UI/HotbarUI.cs`):

```csharp
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.Inventory;

namespace FarmingCourse.UI
{
    public class HotbarUI : MonoBehaviour
    {
        [System.Serializable]
        public class SlotUI
        {
            public Image background;
            public Image icon;
            public TMPro.TextMeshProUGUI countText;
        }

        [SerializeField] private SlotUI[] slotUIs; // index 0..4
        [SerializeField] private Color selectedColor = new Color(1, 1, 0, 1);
        [SerializeField] private Color unselectedColor = new Color(1, 1, 1, 0.7f);

        private void Start()
        {
            if (Hotbar.Instance != null)
            {
                Hotbar.Instance.OnHotbarChanged += Refresh;
                Hotbar.Instance.OnSelectionChanged += _ => Refresh();
            }
            Refresh();
        }

        private void OnDestroy()
        {
            if (Hotbar.Instance != null)
            {
                Hotbar.Instance.OnHotbarChanged -= Refresh;
            }
        }

        private void Refresh()
        {
            if (Hotbar.Instance == null) return;

            for (int i = 0; i < slotUIs.Length && i < Hotbar.SlotCount; i++)
            {
                var slot = Hotbar.Instance.GetType()
                    .GetField("slots", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance) is { } fi
                    ? ((System.Collections.Generic.List<Hotbar.Slot>)fi.GetValue(Hotbar.Instance))[i]
                    : null;

                // (Lebih baik: expose public getter di Hotbar. Refleksi cuma demo.)
                if (slot == null) continue;

                if (slot.IsEmpty)
                {
                    slotUIs[i].icon.enabled = false;
                    slotUIs[i].countText.text = "";
                }
                else
                {
                    slotUIs[i].icon.enabled = true;
                    slotUIs[i].icon.sprite = slot.item.icon;
                    slotUIs[i].countText.text = slot.count > 1 ? slot.count.ToString() : "";
                }

                slotUIs[i].background.color = (i == Hotbar.Instance.SelectedIndex)
                    ? selectedColor : unselectedColor;
            }
        }
    }
}
```

> **Refleksi itu hack.** Mari refactor Hotbar untuk expose `GetSlot(int index)`:
>
> Tambah ke `Hotbar.cs`:
> ```csharp
> public Slot GetSlot(int index) => (index >= 0 && index < slots.Count) ? slots[index] : null;
> ```
>
> Lalu di HotbarUI, ganti baris reflection jadi:
> ```csharp
> Hotbar.Slot slot = Hotbar.Instance.GetSlot(i);
> ```

Drag `HotbarUI.cs` ke `HotbarPanel`. Set field `slotUIs` (size 5), drag tiap slot UI references.

Pencet Play. Hotbar visual nampak. Pencet 1-5 → highlight pindah. Tekan klik → log di console.

---

## Troubleshooting

### "CropTileManager not found" error

Komentar baris yang reference itu, isi log saja sementara. Kita bikin di Chapter 6.

### Hotbar gak update saat select

- `Hotbar.Instance` null? Pastikan Hotbar component di GameObject `_Managers`.
- Event subscribed? Cek `OnEnable` PlayerToolHandler.

### Indicator gak muncul

- `targetIndicatorPrefab` di Inspector terisi?
- Prefab sprite ada?
- Sorting layer/order di atas tilemap?

### `OnHotbar1` gak ke-trigger

- Action `Hotbar1` (sampai 5) ada di Input Actions asset?
- Method nama persis `OnHotbar1` di PlayerInputHandler?
- PlayerInput Behavior = Send Messages?

---

## Latihan

1. **Buat upgrade sistem**: bikin `IronHoeToolSO` (subclass) dengan `tillRadius = 1` (3×3). Bikin asset `IronHoe.asset`. Set di Hotbar Slot 0 → coba klik → Console log "Tilling tile (X,Y) (radius 1)".
2. **Tool durability**: tambah `int currentDurability` di `ToolSO`. Kurangi tiap `Use()`. Saat 0, gak bisa pakai. Pop log warning.
3. **Stamina**: bikin `PlayerStamina` script di Player. Mulai 100. Tiap pakai tool, kurangi `staminaCost`. Saat < 0, blokir tool use.
4. **Sound effect**: di `tool.Use()` callback, panggil `AudioSource.PlayOneShot(useSfx)`. Bikin `useSfx` per tool berbeda (hoe = thud, watering can = splash, scythe = swoosh). Asset SFX gratis di freesound.org.

---

## Recap

- [x] Hierarki SO: ItemSO → ToolSO → HoeToolSO/WateringCanToolSO/ScytheToolSO; SeedSO; CropSO.
- [x] BasicItemSO untuk simple items.
- [x] Asset Hoe, WateringCan, Scythe, CarrotSeed, Carrot, CarrotItem.
- [x] Hotbar singleton 5 slot dengan select 1-5.
- [x] PlayerToolHandler: dispatch click ke tool/seed Use().
- [x] Visual target indicator di tile depan player.
- [x] Hotbar UI dasar (Image grid + count text).

Sistem alat sudah ada. Saatnya bikin **mekanik farming sebenarnya**: till, water, plant, grow, harvest.

---

## Lanjut

[**Chapter 6 — Farming Mechanics →**](06-farming-mechanics.md)

[← Chapter 4](04-tilemap-world.md) | [Daftar Isi](index.md)
