# Chapter 6 — Farming Mechanics (Till, Water, Plant, Grow, Harvest)

> "Inti dari farming game ada di sini. Yang bagus, gimmicks yang lain bisa nanti."

Ini chapter terberat tapi paling memuaskan. Kita bikin **sistem farming penuh**: cangkul tanah, tanam seed, siram, tunggu hari, panen. Dengan data persistent per-tile.

---

## Tujuan Chapter

Setelah chapter ini, kamu punya:

- `CropTileManager` singleton — *single source of truth* untuk state semua tile farming.
- Data per-tile: `Untouched | Tilled | Watered | Planted (with crop & stage) | Mature`.
- Visual swap: tile berubah sprite saat tilled / watered.
- Crop sprite spawn di tile dengan stage progression.
- `TryTill`, `TryWater`, `TryPlant`, `TryHarvest` API yang dipanggil ToolSO/SeedSO.
- Manual `AdvanceDay()` method untuk testing growth (Chapter 8 connect ke time system beneran).

---

## Prasyarat

- [Chapter 5](05-tools-system.md) selesai (tool dispatch jalan).

---

## Konsep: Data Model Tile

Setiap tile farming bisa dalam salah satu state:

```
Untouched (rumput biasa) 
    ↓ Hoe (Use)
Tilled (tanah coklat kering)
    ↓ Watering Can (Use)
Watered (tanah coklat basah)
    ↓ Seed (Use)
Planted (Watered + crop sprite stage 0)
    ↓ next day (TickGrowth)
Planted stage 1, 2, 3... → Mature
    ↓ Scythe (Use)
Empty (Tilled lagi, atau revert ke Untouched)
```

Untuk simpannya, kita pakai struct `CropTileData`:

```csharp
public class CropTileData
{
    public TileState state;
    public CropSO crop;          // null kalau gak ditanam
    public int growthStage;      // 0..crop.growthStages.Length-1
    public int daysSinceLastWatered;
    public bool watered;
}
```

Di-store di `Dictionary<Vector3Int, CropTileData>` di CropTileManager.

---

## Step 1 — Buat Layer Tilemap "Farmable"

Kita butuh tilemap baru untuk overlay tanah tilled / watered di atas Ground.

Hierarchy → `World` (Grid) → klik kanan → 2D Object → Tilemap → Rectangular → namain `Farmable`.

Set Tilemap Renderer:

- Sorting Layer: `Default`.
- Order in Layer: 1 (di atas Ground=0, tapi di bawah Decoration=2).

> Atau kita pakai Decoration tilemap saja — tapi cleaner punya tilemap khusus untuk farming state.

---

## Step 2 — Buat Tile Asset untuk Tilled & Watered

Slice spritesheet `Tilled_Dirt.png` di Sprout Lands (16×16 grid). Drag salah satu sprite tilled (tanah kering) ke Palette atau langsung jadi Tile asset:

- Project → klik kanan → Create → 2D → Tiles → Tile → namain `TilledDirt_Single`.
- Set Sprite ke sprite tilled tanah kering (single 1×1 tile, gak perlu rule).

Bikin tile lain:

- `WateredDirt_Single` — sprite tanah basah (warna gelap).

> Sprout Lands punya banyak variant (corner, edge, dll). Untuk learning, pakai single tile. Polish dengan rule tile setelahnya.

---

## Step 3 — Bikin `CropTileManager`

`Assets/_Project/Scripts/Farming/CropTileManager.cs`:

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Tilemaps;
using FarmingCourse.Inventory;

namespace FarmingCourse.Farming
{
    public enum TileState
    {
        Untouched,
        Tilled,
        Watered,
        Planted    // includes growthStage 0..n
    }

    /// <summary>
    /// State per tile farming. Light enough untuk store di Dictionary.
    /// </summary>
    [System.Serializable]
    public class CropTileData
    {
        public TileState state = TileState.Untouched;
        public CropSO crop;
        public int growthStage;
        public bool watered;
        public int daysSinceWatered;

        public bool IsMature =>
            state == TileState.Planted &&
            crop != null &&
            growthStage >= crop.growthStages.Length - 1;
    }

    /// <summary>
    /// Single source of truth untuk semua tile farming.
    /// Tools (Hoe, WateringCan, Seed, Scythe) panggil method di sini.
    /// </summary>
    public class CropTileManager : MonoBehaviour
    {
        public static CropTileManager Instance { get; private set; }

        [Header("Tilemap References")]
        [SerializeField] private Tilemap groundTilemap;
        [SerializeField] private Tilemap farmableTilemap;
        [SerializeField] private Tilemap obstaclesTilemap;

        [Header("Tile Sprites")]
        [SerializeField] private TileBase tilledTile;
        [SerializeField] private TileBase wateredTile;

        [Header("Crop Visuals")]
        [SerializeField] private GameObject cropVisualPrefab;  // SpriteRenderer simple

        // State semua tile farming.
        private Dictionary<Vector3Int, CropTileData> tiles = new Dictionary<Vector3Int, CropTileData>();
        // Mapping cell -> visual GameObject untuk update sprite stage.
        private Dictionary<Vector3Int, SpriteRenderer> cropVisuals = new Dictionary<Vector3Int, SpriteRenderer>();

        // Event hook untuk UI / quest / save.
        public event Action<Vector3Int, CropTileData> OnTileChanged;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
        }

        // ============================================================
        //  PUBLIC API
        // ============================================================

        public CropTileData GetTile(Vector3Int cell)
        {
            return tiles.TryGetValue(cell, out var data) ? data : null;
        }

        /// <summary>
        /// Cangkul tile. Sukses kalau tile masih untouched & gak ada obstacle.
        /// Radius > 0 till area persegi (1 = 3x3, 2 = 5x5).
        /// </summary>
        public bool TryTill(Vector3Int center, int radius = 0)
        {
            bool anySuccess = false;
            for (int dx = -radius; dx <= radius; dx++)
            for (int dy = -radius; dy <= radius; dy++)
            {
                Vector3Int cell = new Vector3Int(center.x + dx, center.y + dy, center.z);
                if (TryTillSingle(cell)) anySuccess = true;
            }
            return anySuccess;
        }

        private bool TryTillSingle(Vector3Int cell)
        {
            if (!IsTillable(cell)) return false;

            var data = GetOrCreate(cell);
            if (data.state != TileState.Untouched && data.state != TileState.Tilled)
                return false; // udah planted/watered, gak bisa till lagi

            data.state = TileState.Tilled;
            farmableTilemap.SetTile(cell, tilledTile);
            OnTileChanged?.Invoke(cell, data);
            return true;
        }

        public bool TryWater(Vector3Int center, int radius = 0)
        {
            bool any = false;
            for (int dx = -radius; dx <= radius; dx++)
            for (int dy = -radius; dy <= radius; dy++)
            {
                Vector3Int cell = new Vector3Int(center.x + dx, center.y + dy, center.z);
                if (TryWaterSingle(cell)) any = true;
            }
            return any;
        }

        private bool TryWaterSingle(Vector3Int cell)
        {
            var data = GetTile(cell);
            if (data == null) return false;
            if (data.state == TileState.Untouched) return false;
            if (data.watered) return false; // udah basah

            data.watered = true;
            data.daysSinceWatered = 0;

            // Visual: kalau Planted, biarin tanah-nya keliatan watered di bawah crop.
            // Solusi simple: pakai watered tile selalu kalau watered=true.
            farmableTilemap.SetTile(cell, wateredTile);

            OnTileChanged?.Invoke(cell, data);
            return true;
        }

        public bool TryPlant(Vector3Int cell, SeedSO seed)
        {
            var data = GetTile(cell);
            if (data == null || data.state == TileState.Untouched) return false;
            if (data.state == TileState.Planted) return false; // udah ada crop
            if (seed == null || seed.cropToPlant == null) return false;

            data.state = TileState.Planted;
            data.crop = seed.cropToPlant;
            data.growthStage = 0;

            SpawnCropVisual(cell, data);
            OnTileChanged?.Invoke(cell, data);
            return true;
        }

        public bool TryHarvest(Vector3Int cell)
        {
            var data = GetTile(cell);
            if (data == null) return false;
            if (data.state != TileState.Planted) return false;
            if (!data.IsMature) return false;

            // Kasih item ke inventory. Untuk sekarang, log saja -- Chapter 7
            // hubungin ke InventoryManager.
            Debug.Log($"[CropTileManager] Harvested {data.crop.harvestAmount}x {data.crop.harvestItem.displayName}");
            // TODO: InventoryManager.Instance.Add(data.crop.harvestItem, data.crop.harvestAmount);

            // Regrow logic.
            if (data.crop.regrowsAfterHarvest)
            {
                data.growthStage = data.crop.regrowStage;
                UpdateCropVisualStage(cell, data);
            }
            else
            {
                // Kembalikan ke tilled (gak watered) supaya bisa ditanami ulang.
                data.state = TileState.Tilled;
                data.crop = null;
                data.growthStage = 0;
                data.watered = false;
                farmableTilemap.SetTile(cell, tilledTile);
                DestroyCropVisual(cell);
            }

            OnTileChanged?.Invoke(cell, data);
            return true;
        }

        /// <summary>
        /// Dipanggil oleh TimeManager saat ganti hari.
        /// </summary>
        public void TickDay()
        {
            // Buat copy keys untuk avoid modifying-during-iteration.
            var keys = new List<Vector3Int>(tiles.Keys);
            foreach (var cell in keys)
            {
                var data = tiles[cell];
                if (data.state != TileState.Planted) continue;

                if (data.watered)
                {
                    // Tumbuh.
                    if (data.growthStage < data.crop.growthStages.Length - 1)
                    {
                        data.growthStage++;
                        UpdateCropVisualStage(cell, data);
                    }
                    // Reset watered (perlu siram ulang besoknya).
                    data.watered = false;
                    farmableTilemap.SetTile(cell, tilledTile);
                }
                else
                {
                    data.daysSinceWatered++;
                    // Optional: kalau gak disiram 3 hari, mati. Untuk sekarang skip.
                }

                OnTileChanged?.Invoke(cell, data);
            }
        }

        // ============================================================
        //  HELPERS
        // ============================================================

        private bool IsTillable(Vector3Int cell)
        {
            // Tile harus di tilemap Ground (rumput) dan gak punya obstacle.
            if (groundTilemap == null) return true; // permissive saat dev
            if (groundTilemap.GetTile(cell) == null) return false;
            if (obstaclesTilemap != null && obstaclesTilemap.GetTile(cell) != null) return false;
            return true;
        }

        private CropTileData GetOrCreate(Vector3Int cell)
        {
            if (!tiles.TryGetValue(cell, out var data))
            {
                data = new CropTileData();
                tiles[cell] = data;
            }
            return data;
        }

        private void SpawnCropVisual(Vector3Int cell, CropTileData data)
        {
            if (cropVisualPrefab == null) return;
            if (cropVisuals.ContainsKey(cell)) return;

            Vector3 worldPos = farmableTilemap.GetCellCenterWorld(cell);
            var go = Instantiate(cropVisualPrefab, worldPos, Quaternion.identity, transform);
            var sr = go.GetComponent<SpriteRenderer>();
            if (sr != null && data.crop != null && data.crop.growthStages.Length > 0)
            {
                sr.sprite = data.crop.growthStages[0];
            }
            cropVisuals[cell] = sr;
        }

        private void UpdateCropVisualStage(Vector3Int cell, CropTileData data)
        {
            if (!cropVisuals.TryGetValue(cell, out var sr)) return;
            if (data.crop == null) return;
            int stage = Mathf.Clamp(data.growthStage, 0, data.crop.growthStages.Length - 1);
            sr.sprite = data.crop.growthStages[stage];
        }

        private void DestroyCropVisual(Vector3Int cell)
        {
            if (cropVisuals.TryGetValue(cell, out var sr) && sr != null)
            {
                Destroy(sr.gameObject);
            }
            cropVisuals.Remove(cell);
        }
    }
}
```

### Penjelasan

Mari unpack design choices:

- **Singleton**. Ada banyak Player tools yang nge-call manager. Singleton bikin gampang. Trade-off: testability, tapi untuk learning oke.
- **Dictionary store** vs ScriptableObject store. Dictionary in-memory cepat. SO atau JSON dipakai untuk save/load nanti.
- **Watered = bool, daysSinceWatered = int**. Crop tumbuh hanya kalau watered hari itu. Setelah TickDay, watered direset.
- **Visual GameObject untuk crop**, bukan tile. Karena sprite crop sering punya pivot beda dan animasi lebih mudah di GameObject.
- **`OnTileChanged` event**. Quest system, tutorial, save system bisa subscribe.
- **`IsTillable`** check Ground tilemap. Kalau player coba till di water atau di luar map, gagal.

---

## Step 4 — Setup CropTileManager di Scene

Pilih `_Managers` GameObject → Add Component → **Crop Tile Manager**.

Set fields di Inspector:

- **Ground Tilemap**: drag GameObject `Ground` (tilemap).
- **Farmable Tilemap**: drag GameObject `Farmable`.
- **Obstacles Tilemap**: drag GameObject `Obstacles`.
- **Tilled Tile**: drag asset `TilledDirt_Single`.
- **Watered Tile**: drag asset `WateredDirt_Single`.
- **Crop Visual Prefab**: kita bikin prefabnya berikutnya.

### Bikin Crop Visual Prefab

1. Hierarchy → 2D Object → Sprites → Square → namain `CropVisual`.
2. Sprite: kosongkan (akan di-set runtime).
3. Sorting Layer: `Default`, Order in Layer: 3 (atas Decoration, di bawah Player).
4. Hapus collider kalau ada.
5. Drag ke `Prefabs/Crops/CropVisual.prefab`. Hapus dari Hierarchy.
6. Drag prefab ke field **Crop Visual Prefab** di CropTileManager.

---

## Step 5 — Restore Tool Calls ke CropTileManager

Buka `HoeToolSO.cs` → ganti `Use()`:

```csharp
public override bool Use(Vector3 userPosition, Vector3Int targetTile)
{
    if (CropTileManager.Instance == null) return false;
    return CropTileManager.Instance.TryTill(targetTile, tillRadius);
}
```

`WateringCanToolSO.cs`:

```csharp
public override bool Use(Vector3 userPosition, Vector3Int targetTile)
{
    if (currentWater <= 0) return false;
    if (CropTileManager.Instance == null) return false;

    bool success = CropTileManager.Instance.TryWater(targetTile, waterRadius);
    if (success) currentWater -= waterPerUse;
    return success;
}
```

`SeedSO.cs`:

```csharp
public bool Plant(Vector3Int targetTile)
{
    if (CropTileManager.Instance == null) return false;
    return CropTileManager.Instance.TryPlant(targetTile, this);
}
```

`ScytheToolSO.cs`:

```csharp
public override bool Use(Vector3 userPosition, Vector3Int targetTile)
{
    if (CropTileManager.Instance == null) return false;
    return CropTileManager.Instance.TryHarvest(targetTile);
}
```

Save semua. Compile.

---

## Step 6 — Test Manual

Pencet **Play**.

1. Slot 0 (Hoe) selected.
2. Hadap bawah → klik kiri di tile rumput → Console log "Used Hoe on (X, Y)" + tile berubah jadi tanah coklat. ✅
3. Pencet `2` → Watering Can. Awal water = 0 — gagal. Refill manual untuk test:
   - Stop play. Edit `WateringCanToolSO.cs` Awake atau di Inspector default `currentWater = 40`.
   - **Atau** bikin tile water source: kalau player adjacent tile Water, refill.
4. Klik di tile tilled → tanah jadi watered (lebih gelap).
5. Pencet `3` → CarrotSeed. Klik di tile watered → muncul crop visual stage 0.
6. **Manual TickDay** — kita belum punya time system. Tambahkan tombol shortcut di CropTileManager untuk test:

   ```csharp
   private void Update()
   {
   #if UNITY_EDITOR
       if (Input.GetKeyDown(KeyCode.T))
       {
           Debug.Log("[CropTileManager] Manual TickDay triggered.");
           TickDay();
       }
   #endif
   }
   ```

   Tekan `T` di game → crop tumbuh ke stage 1. Tekan T lagi → stage 2. Lanjut sampai mature.

7. Pencet `4` → Scythe. Klik di crop mature → log "Harvested 1x Carrot" + sprite hilang + tanah balik tilled. ✅

🎉 **Loop farming dasar jalan!**

---

## Step 7 — Refill Watering Can di Air

Bikin script kecil:

`Assets/_Project/Scripts/Farming/WaterRefillTrigger.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Inventory;
using FarmingCourse.Inventory.Tools;

namespace FarmingCourse.Farming
{
    /// <summary>
    /// Saat player adjacent tile water dan UseTool sambil pegang watering can,
    /// refill. Cek di PlayerToolHandler.
    /// </summary>
    public static class WaterRefillTrigger
    {
        public static bool TryRefill(WateringCanToolSO can, Vector3Int targetTile, UnityEngine.Tilemaps.Tilemap waterTilemap)
        {
            if (can == null || waterTilemap == null) return false;
            if (waterTilemap.GetTile(targetTile) == null) return false; // bukan air

            can.Refill();
            Debug.Log("[WaterRefill] Watering can refilled.");
            return true;
        }
    }
}
```

Lalu edit `PlayerToolHandler.HandleUseTool`:

```csharp
private void HandleUseTool()
{
    if (Hotbar.Instance == null) return;
    ItemSO item = Hotbar.Instance.SelectedItem;
    if (item == null || !item.CanUse) return;

    Vector3Int target = GetTargetCell();
    bool success = false;

    switch (item)
    {
        case Inventory.Tools.WateringCanToolSO can:
            // Cek refill dulu (kalau target adalah air).
            if (CropTileManager.Instance != null &&
                Farming.WaterRefillTrigger.TryRefill(can, target,
                    Farming.CropTileManager.Instance.GetWaterTilemap()))
            {
                success = true;
                break;
            }
            success = can.Use(transform.position, target);
            break;

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
        Debug.Log($"[PlayerToolHandler] Used {item.displayName} on {target}");
    }
}
```

Tambah `GetWaterTilemap()` di `CropTileManager.cs`:

```csharp
[SerializeField] private Tilemap waterTilemap;
public Tilemap GetWaterTilemap() => waterTilemap;
```

Set field `Water Tilemap` di Inspector → drag GameObject Water.

Test: dekat tile air → klik dengan watering can → log "refilled". ✅

---

## Step 8 — Visual Polish: Watering Can Animation Drips

Saat siram, kita mau ada animasi drips. Quick & dirty:

1. Bikin Particle System child dari Player.
2. Configure: emit 5 particles burst, downward, blue color.
3. Trigger di `HandleUseTool`:

```csharp
[SerializeField] private ParticleSystem waterDripsParticles;
// ...
case WateringCanToolSO can:
    if (success && waterDripsParticles != null) waterDripsParticles.Emit(5);
    break;
```

Detail polish bisa di Chapter 11.

---

## Step 9 — Crop Tinggi / Multi-Tile (Opsional)

Beberapa crop besar (tomato, corn) butuh 2 tile. Buat sekarang skip — semua crop kita 1 tile. Kalau mau extend:

- Tambah `int width, height` di CropSO.
- Saat `TryPlant`, occupy multiple cells (mark semua sebagai planted, link ke same data).
- Spawn visual dengan offset Y.

Kompleks, tunda untuk Chapter 12 (next steps).

---

## Step 10 — Save State (Persiapan Save System)

Untuk persist ke file (Chapter 10), kita butuh serializable representation. Tambah method di CropTileManager:

```csharp
[System.Serializable]
public class CropTileSaveEntry
{
    public Vector3Int cell;
    public TileState state;
    public string cropId;       // simpan id, bukan reference SO
    public int growthStage;
    public bool watered;
    public int daysSinceWatered;
}

public List<CropTileSaveEntry> ExportSave()
{
    var list = new List<CropTileSaveEntry>();
    foreach (var kv in tiles)
    {
        list.Add(new CropTileSaveEntry
        {
            cell = kv.Key,
            state = kv.Value.state,
            cropId = kv.Value.crop != null ? kv.Value.crop.name : null,
            growthStage = kv.Value.growthStage,
            watered = kv.Value.watered,
            daysSinceWatered = kv.Value.daysSinceWatered,
        });
    }
    return list;
}

public void ImportSave(List<CropTileSaveEntry> entries, CropDatabase db)
{
    // Clear current state.
    foreach (var v in cropVisuals.Values) if (v != null) Destroy(v.gameObject);
    cropVisuals.Clear();
    tiles.Clear();
    farmableTilemap.ClearAllTiles();

    foreach (var e in entries)
    {
        var data = new CropTileData
        {
            state = e.state,
            crop = db.FindById(e.cropId),
            growthStage = e.growthStage,
            watered = e.watered,
            daysSinceWatered = e.daysSinceWatered,
        };
        tiles[e.cell] = data;

        switch (e.state)
        {
            case TileState.Tilled:
                farmableTilemap.SetTile(e.cell, e.watered ? wateredTile : tilledTile);
                break;
            case TileState.Watered:
                farmableTilemap.SetTile(e.cell, wateredTile);
                break;
            case TileState.Planted:
                farmableTilemap.SetTile(e.cell, e.watered ? wateredTile : tilledTile);
                SpawnCropVisual(e.cell, data);
                if (data.crop != null && e.growthStage > 0)
                {
                    UpdateCropVisualStage(e.cell, data);
                }
                break;
        }
    }
}
```

Untuk `CropDatabase`, buat helper kecil:

`Assets/_Project/Scripts/Farming/CropDatabase.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace FarmingCourse.Farming
{
    [CreateAssetMenu(fileName = "CropDatabase", menuName = "FarmingCourse/Crops/Database")]
    public class CropDatabase : ScriptableObject
    {
        public List<CropSO> crops;

        public CropSO FindById(string id)
        {
            if (string.IsNullOrEmpty(id)) return null;
            return crops.Find(c => c.name == id);
        }
    }
}
```

Bikin asset `CropDatabase.asset` di `Assets/_Project/ScriptableObjects/Crops/`. Drag semua CropSO yang ada (Carrot dulu) ke listnya.

(Save manager-nya sendiri di Chapter 10. Sekarang fungsinya tersedia.)

---

## Step 11 — Tile Variants Visual (Polish Opsional)

Sprout Lands punya banyak variant tilled (corner, edge, center). Untuk auto-tiling saat banyak tile tilled berdekatan, pakai **Rule Tile** untuk tilled juga. Tapi ini polish, skip untuk learning.

---

## Troubleshooting

### `CropTileManager.Instance` null

- Komponen ada di GameObject di Scene aktif?
- Awake order issue: `Hotbar.Instance` mungkin di-akses sebelum Awake. Pastikan GameObject `_Managers` aktif & punya komponen.

### Tile gak berubah saat di-till

- `Ground Tilemap` field terisi?
- Tile target ada di Ground tilemap (bukan luar map)?
- `TilledTile` asset terisi?
- Cek log error di console.

### Crop visual numpuk waktu plant 2x

- `TryPlant` reject kalau state == Planted. Cek branch return false.

### Crop gak tumbuh saat tekan T

- TickDay branch: `data.watered` harus true. Kamu sudah water-nya?

### Watering can refill gak jalan

- Water Tilemap field terisi?
- Target tile beneran di Water tilemap?

---

## Latihan

1. **Hari hujan auto-water all crops**: tambah method `WaterAll()` di CropTileManager. Panggil saat `TimeManager.IsRaining` (Chapter 8).
2. **Decay**: tile tilled tanpa crop akan revert ke Untouched setelah 5 hari. Implement di `TickDay`.
3. **Crop quality**: tambah field `int quality` ke CropTileData. Random 1-5 saat harvest. Item harvest dengan quality ≥ 4 pakai sprite "shiny" (pakai 2 ItemSO terpisah atau quality flag di item).
4. **Special crop "Strawberry"** (regrows): bikin asset Strawberry CropSO dengan `regrowsAfterHarvest = true`, regrowStage = 2, regrowDays = 3. Test alur regrow.
5. **Sound effect** per action: hoe = "thud", watering = "splash", harvest = "snap". Triggered di CropTileManager method-nya (panggil `AudioManager.Play(...)` — kita bikin di Chapter 11).

---

## Recap

- [x] CropTileManager singleton dengan dictionary state per tile.
- [x] TileState: Untouched, Tilled, Watered, Planted (with stage).
- [x] TryTill / TryWater / TryPlant / TryHarvest API.
- [x] TickDay untuk grow stage.
- [x] Visual swap tile (tilled / watered sprite).
- [x] Crop visual GameObject dengan sprite per stage.
- [x] Refill watering can di tile water.
- [x] Save export / import structure (ready for Chapter 10).

**Kamu sekarang punya farming game yang playable**. Selanjutnya: bikin inventory & UI yang lebih lengkap.

---

## Lanjut

[**Chapter 7 — Inventory & UI →**](07-inventory-system.md)

[← Chapter 5](05-tools-system.md) | [Daftar Isi](index.md)
