# Chapter 10 — Save / Load System

> "Game tanpa save = experience yang hilang. Save bug = experience yang lebih hilang."

Sekarang waktunya simpan progress ke file. Saat player tutup game atau reload, semuanya kembali persis di mana mereka tinggal.

---

## Tujuan Chapter

- `SaveManager` singleton yang serialize/deserialize state ke file JSON.
- Save Data berisi:
  - Player position & facing direction.
  - Inventory slot (item id + count).
  - Wallet coin amount.
  - Time (hour, minute, day, season, year).
  - CropTileManager state (per-cell data).
  - Hotbar selection.
- Auto-save tiap end-of-day (in `EndSleep`).
- Manual save (tombol di pause menu) & load.
- Multiple save slot (3 slot).
- Save file location cross-platform (`Application.persistentDataPath`).

---

## Prasyarat

- [Chapter 9](09-shop-npc.md) selesai.

---

## Konsep: JSON Serialization di Unity

Unity punya `JsonUtility` built-in yang serialize MonoBehaviour & ScriptableObject **tanpa** field `[NonSerialized]`. Tapi `JsonUtility` punya batasan:

- Gak support Dictionary.
- Gak support Vector3Int (di JSON).
- Gak support polymorphism (interface, base class).

Workarounds:

- Konversi Dictionary jadi `List<Entry>` saat save. Convert balik saat load.
- Vector3Int → split jadi 3 int x, y, z.
- Save reference ItemSO → simpan `itemId` (string), bukan reference. Saat load, lookup di database.

Alternative: **Newtonsoft.Json** package (lebih powerful, bisa Dictionary). Course ini pakai JsonUtility built-in untuk simplisitas.

---

## Step 1 — Bikin `ItemDatabase`

Lookup table dari `itemId` → `ItemSO`.

`Assets/_Project/Scripts/Inventory/ItemDatabase.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace FarmingCourse.Inventory
{
    [CreateAssetMenu(fileName = "ItemDatabase", menuName = "FarmingCourse/Items/Item Database")]
    public class ItemDatabase : ScriptableObject
    {
        public List<ItemSO> items;

        private Dictionary<string, ItemSO> cache;

        public ItemSO FindById(string id)
        {
            if (string.IsNullOrEmpty(id)) return null;
            EnsureCache();
            return cache.TryGetValue(id, out var v) ? v : null;
        }

        private void EnsureCache()
        {
            if (cache != null) return;
            cache = new Dictionary<string, ItemSO>();
            foreach (var item in items)
            {
                if (item == null || string.IsNullOrEmpty(item.itemId)) continue;
                if (cache.ContainsKey(item.itemId))
                {
                    Debug.LogError($"[ItemDatabase] Duplicate itemId: {item.itemId}");
                    continue;
                }
                cache[item.itemId] = item;
            }
        }

#if UNITY_EDITOR
        // Auto-populate from Project assets via menu.
        [UnityEditor.MenuItem("FarmingCourse/Refresh Item Database")]
        public static void Refresh()
        {
            var db = UnityEditor.AssetDatabase.LoadAssetAtPath<ItemDatabase>("Assets/_Project/ScriptableObjects/Items/ItemDatabase.asset");
            if (db == null) { Debug.LogError("ItemDatabase asset not found."); return; }
            db.items = new List<ItemSO>();
            string[] guids = UnityEditor.AssetDatabase.FindAssets("t:ItemSO");
            foreach (var g in guids)
            {
                var path = UnityEditor.AssetDatabase.GUIDToAssetPath(g);
                var item = UnityEditor.AssetDatabase.LoadAssetAtPath<ItemSO>(path);
                if (item != null) db.items.Add(item);
            }
            UnityEditor.EditorUtility.SetDirty(db);
            UnityEditor.AssetDatabase.SaveAssets();
            Debug.Log($"[ItemDatabase] Refreshed with {db.items.Count} items.");
        }
#endif
    }
}
```

Bikin asset `ItemDatabase.asset` di `Assets/_Project/ScriptableObjects/Items/`. Atau panggil menu **FarmingCourse → Refresh Item Database** untuk auto-populate.

> Note: Menu item refleksi pakai `UnityEditor` namespace. Code di `#if UNITY_EDITOR` supaya gak ke-compile di build.

---

## Step 2 — Bikin `SaveData` Class

`Assets/_Project/Scripts/Save/SaveData.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using FarmingCourse.Farming;

namespace FarmingCourse.Save
{
    [System.Serializable]
    public class SaveData
    {
        public string version = "1.0";

        // Player.
        public PlayerSaveData player = new PlayerSaveData();

        // Inventory.
        public List<InventorySlotSaveData> inventory = new List<InventorySlotSaveData>();
        public int hotbarSelectedIndex;

        // Wallet.
        public int coins;

        // Time.
        public int hour, minute, day, year;
        public Season season;

        // Crop tiles.
        public List<CropTileSaveEntry> cropTiles = new List<CropTileSaveEntry>();

        // Watering can fill.
        public int wateringCanFill;
    }

    [System.Serializable]
    public class PlayerSaveData
    {
        public float posX, posY;
        public float facingX, facingY;
    }

    [System.Serializable]
    public class InventorySlotSaveData
    {
        public int slotIndex;
        public string itemId;
        public int count;
    }

    [System.Serializable]
    public class CropTileSaveEntry
    {
        public int cellX, cellY;
        public TileState state;
        public string cropId;
        public int growthStage;
        public bool watered;
        public int daysSinceWatered;
    }
}
```

---

## Step 3 — `SaveManager`

`Assets/_Project/Scripts/Save/SaveManager.cs`:

```csharp
using System.IO;
using UnityEngine;
using FarmingCourse.Farming;
using FarmingCourse.Inventory;
using FarmingCourse.Inventory.Tools;
using FarmingCourse.Player;
using FarmingCourse.Time;

namespace FarmingCourse.Save
{
    public class SaveManager : MonoBehaviour
    {
        public static SaveManager Instance { get; private set; }

        [Header("References")]
        [SerializeField] private ItemDatabase itemDatabase;
        [SerializeField] private CropDatabase cropDatabase;
        [SerializeField] private WateringCanToolSO wateringCan;
        [SerializeField] private Transform playerTransform;

        private const int MaxSlots = 3;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
        }

        private string GetSavePath(int slot) =>
            Path.Combine(Application.persistentDataPath, $"save_{slot}.json");

        public bool HasSave(int slot) => File.Exists(GetSavePath(slot));

        // ============================================================
        //  SAVE
        // ============================================================
        public void Save(int slot)
        {
            var data = new SaveData();

            // Player.
            if (playerTransform != null)
            {
                data.player.posX = playerTransform.position.x;
                data.player.posY = playerTransform.position.y;
                var movement = playerTransform.GetComponent<PlayerMovement>();
                if (movement != null)
                {
                    data.player.facingX = movement.FacingDirection.x;
                    data.player.facingY = movement.FacingDirection.y;
                }
            }

            // Inventory.
            if (InventoryManager.Instance != null)
            {
                for (int i = 0; i < InventoryManager.TotalSize; i++)
                {
                    var slot_ = InventoryManager.Instance.GetSlot(i);
                    if (slot_ == null || slot_.IsEmpty) continue;
                    data.inventory.Add(new InventorySlotSaveData
                    {
                        slotIndex = i,
                        itemId = slot_.item.itemId,
                        count = slot_.count,
                    });
                }
                data.hotbarSelectedIndex = InventoryManager.Instance.SelectedHotbarIndex;
            }

            // Wallet.
            if (WalletManager.Instance != null) data.coins = WalletManager.Instance.Coins;

            // Time.
            if (TimeManager.Instance != null)
            {
                data.hour = TimeManager.Instance.Hour;
                data.minute = TimeManager.Instance.Minute;
                data.day = TimeManager.Instance.Day;
                data.year = TimeManager.Instance.Year;
                data.season = TimeManager.Instance.Season;
            }

            // Crop tiles.
            if (CropTileManager.Instance != null)
            {
                var entries = CropTileManager.Instance.ExportSave();
                foreach (var e in entries)
                {
                    data.cropTiles.Add(new CropTileSaveEntry
                    {
                        cellX = e.cell.x, cellY = e.cell.y,
                        state = e.state, cropId = e.cropId,
                        growthStage = e.growthStage, watered = e.watered,
                        daysSinceWatered = e.daysSinceWatered,
                    });
                }
            }

            // Watering can fill.
            if (wateringCan != null) data.wateringCanFill = wateringCan.currentWater;

            // Write JSON.
            string json = JsonUtility.ToJson(data, prettyPrint: true);
            string path = GetSavePath(slot);
            try
            {
                File.WriteAllText(path, json);
                Debug.Log($"[SaveManager] Saved to {path}");
            }
            catch (System.Exception ex)
            {
                Debug.LogError($"[SaveManager] Save failed: {ex.Message}");
            }
        }

        // ============================================================
        //  LOAD
        // ============================================================
        public bool Load(int slot)
        {
            string path = GetSavePath(slot);
            if (!File.Exists(path))
            {
                Debug.LogWarning($"[SaveManager] No save at {path}");
                return false;
            }

            SaveData data;
            try
            {
                string json = File.ReadAllText(path);
                data = JsonUtility.FromJson<SaveData>(json);
            }
            catch (System.Exception ex)
            {
                Debug.LogError($"[SaveManager] Load failed: {ex.Message}");
                return false;
            }

            // Player.
            if (playerTransform != null)
            {
                playerTransform.position = new Vector3(data.player.posX, data.player.posY, 0);
                // Facing tidak punya setter publik di PlayerMovement; bisa diabaikan
                // atau tambah method SetFacing kalau perlu.
            }

            // Inventory.
            if (InventoryManager.Instance != null)
            {
                // Clear current.
                for (int i = 0; i < InventoryManager.TotalSize; i++)
                {
                    var s = InventoryManager.Instance.GetSlot(i);
                    if (s != null) { s.item = null; s.count = 0; }
                }
                foreach (var entry in data.inventory)
                {
                    var item = itemDatabase.FindById(entry.itemId);
                    if (item == null) continue;
                    var s = InventoryManager.Instance.GetSlot(entry.slotIndex);
                    if (s != null) { s.item = item; s.count = entry.count; }
                }
                InventoryManager.Instance.SelectHotbar(data.hotbarSelectedIndex);
                // Force inventory UI refresh -- panggil event manual.
                InventoryManager.Instance.SendMessage("OnInventoryChanged", SendMessageOptions.DontRequireReceiver);
            }

            // Wallet.
            if (WalletManager.Instance != null)
            {
                int delta = data.coins - WalletManager.Instance.Coins;
                WalletManager.Instance.Add(delta);
            }

            // Time.
            if (TimeManager.Instance != null)
            {
                TimeManager.Instance.ImportSave(new TimeSaveData
                {
                    hour = data.hour, minute = data.minute, day = data.day,
                    season = data.season, year = data.year,
                });
            }

            // Crop tiles.
            if (CropTileManager.Instance != null && cropDatabase != null)
            {
                var list = new System.Collections.Generic.List<CropTileManager.CropTileSaveEntry>();
                foreach (var e in data.cropTiles)
                {
                    list.Add(new CropTileManager.CropTileSaveEntry
                    {
                        cell = new Vector3Int(e.cellX, e.cellY, 0),
                        state = e.state, cropId = e.cropId,
                        growthStage = e.growthStage, watered = e.watered,
                        daysSinceWatered = e.daysSinceWatered,
                    });
                }
                CropTileManager.Instance.ImportSave(list, cropDatabase);
            }

            // Watering can.
            if (wateringCan != null) wateringCan.currentWater = data.wateringCanFill;

            Debug.Log($"[SaveManager] Loaded from {path}");
            return true;
        }

        public void Delete(int slot)
        {
            string path = GetSavePath(slot);
            if (File.Exists(path)) File.Delete(path);
        }
    }
}
```

> **Pemilihan event manual** untuk `OnInventoryChanged` setelah load: SendMessage hack di atas gak jalan kalau method-nya private. Lebih clean: tambah `public void RaiseChanged()` di InventoryManager yang panggil event-nya.

Edit InventoryManager:

```csharp
public void RaiseInventoryChanged() => OnInventoryChanged?.Invoke();
```

Lalu di SaveManager.Load, ganti SendMessage jadi:

```csharp
InventoryManager.Instance.RaiseInventoryChanged();
```

---

## Step 4 — Auto-Save di End-of-Day

Edit `TimeManager.EndSleep`:

```csharp
public void EndSleep()
{
    day++;
    if (day > daysPerSeason)
    {
        day = 1;
        AdvanceSeason();
    }

    hour = dayStartHour;
    minute = 0;
    IsTimeFlowing = true;

    CropTileManager.Instance?.TickDay();
    OnDayChanged?.Invoke(day, season, year);
    OnTimeChanged?.Invoke(hour, minute);
    OnSleepEnd?.Invoke();

    // Auto-save.
    Save.SaveManager.Instance?.Save(0); // slot 0 = autosave
}
```

---

## Step 5 — UI Save/Load Menu

Bikin Pause Menu sederhana:

1. Canvas → Panel `PauseMenu`. Center, full screen translucent.
2. Tombol: Resume, Save Slot 1/2/3, Load Slot 1/2/3, Quit.

Script `Assets/_Project/Scripts/UI/PauseMenu.cs`:

```csharp
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.Save;

namespace FarmingCourse.UI
{
    public class PauseMenu : MonoBehaviour
    {
        [SerializeField] private GameObject root;
        [SerializeField] private Button[] saveButtons; // size 3
        [SerializeField] private Button[] loadButtons; // size 3
        [SerializeField] private Button resumeButton;
        [SerializeField] private Button quitButton;

        private void Awake()
        {
            root.SetActive(false);
            for (int i = 0; i < saveButtons.Length; i++)
            {
                int slot = i + 1; // skip 0 = autosave
                saveButtons[i].onClick.AddListener(() => SaveManager.Instance?.Save(slot));
                loadButtons[i].onClick.AddListener(() => SaveManager.Instance?.Load(slot));
            }
            resumeButton.onClick.AddListener(Hide);
            quitButton.onClick.AddListener(() =>
            {
#if UNITY_EDITOR
                UnityEditor.EditorApplication.isPlaying = false;
#else
                Application.Quit();
#endif
            });
        }

        private void Update()
        {
            if (Input.GetKeyDown(KeyCode.Escape))
            {
                Toggle();
            }
        }

        public void Toggle() { if (root.activeSelf) Hide(); else Show(); }
        public void Show()
        {
            root.SetActive(true);
            UnityEngine.Time.timeScale = 0f;
        }
        public void Hide()
        {
            root.SetActive(false);
            UnityEngine.Time.timeScale = 1f;
        }
    }
}
```

> **Note**: pakai `UnityEngine.Time.timeScale` untuk pause physics/animasi. Tapi UI animasi tetap jalan kalau pakai `unscaledDeltaTime`.

---

## Step 6 — Load Saat Start

Bikin script `BootLoader.cs` di GameObject `_Managers`:

```csharp
using UnityEngine;
using FarmingCourse.Save;

namespace FarmingCourse.Save
{
    public class BootLoader : MonoBehaviour
    {
        [SerializeField] private bool loadOnStart = true;
        [SerializeField] private int slotToLoad = 0;

        private void Start()
        {
            if (!loadOnStart) return;
            if (SaveManager.Instance == null) return;
            if (SaveManager.Instance.HasSave(slotToLoad))
            {
                SaveManager.Instance.Load(slotToLoad);
            }
        }
    }
}
```

Saat scene mulai, kalau ada save, otomatis load.

---

## Step 7 — Save Versioning (Defensive)

Saat update game, struktur SaveData mungkin berubah. Pakai field `version`:

```csharp
public bool Load(int slot)
{
    // ... read JSON
    if (data.version != "1.0")
    {
        Debug.LogError($"[SaveManager] Unsupported save version: {data.version}");
        return false;
    }
    // ... rest of load
}
```

Kalau di future version kamu rename field, kamu bisa migration code:

```csharp
if (data.version == "1.0") { MigrateV1ToV2(data); data.version = "2.0"; }
```

---

## Step 8 — Save File Path Cross-Platform

`Application.persistentDataPath` resolves ke:

- Windows: `C:\Users\<user>\AppData\LocalLow\<company>\<product>\`
- Mac: `~/Library/Application Support/<company>/<product>/`
- Linux: `~/.config/unity3d/<company>/<product>/`
- Android: `/storage/emulated/0/Android/data/<package>/files/`
- WebGL: pakai IndexedDB (otomatis sync via `JS.SyncFs()` setelah write).

Untuk WebGL, tambahkan setelah file write:

```csharp
#if UNITY_WEBGL && !UNITY_EDITOR
    Application.ExternalEval("FS.syncfs(false, function (err) {})");
#endif
```

---

## Step 9 — Test Save/Load Loop

1. Play, plant beberapa carrot, water, harvest, sell.
2. Tekan Esc → Save Slot 1.
3. Stop play.
4. Cek `Application.persistentDataPath` di Editor: **Window → General → Console → klik kanan log "Saved to..."**. Buka file `save_1.json` di text editor.
5. Verify JSON readable, data benar.
6. Play lagi → Esc → Load Slot 1.
7. Inventory + crop + time + coin pulih.

Kalau gagal, debug langkah-langkah:

- Console error?
- File ada di path?
- JSON valid (bisa parse)?
- ItemDatabase.asset terisi & ada itemId yang dipake?

---

## Step 10 — Cloud Save (Bonus, Opsional)

Untuk Steam build, pakai Steam Cloud API. Untuk WebGL → IndexedDB sudah otomatis (path persistentDataPath disinkronkan ke IndexedDB).

Cross-platform cloud save (Google Cloud, Firebase) di luar scope course ini.

---

## Troubleshooting

### "JsonUtility cannot serialize Vector3Int"

- Vector3Int gak bisa direct ke JSON. Pakai struct `{int cellX; int cellY;}`.

### Save sukses tapi load gak restore

- ItemDatabase / CropDatabase asset terisi di SaveManager Inspector?
- Item yang di-save punya `itemId` non-empty di asset SO?

### "Can't find ScriptableObject Type"

- ScriptableObject reference jangan di-serialize langsung. Save id, lookup di database.

### Save file rusak / corrupt

- Backup save sebelum overwrite (rename file lama jadi `save_1.json.bak`).
- Validasi JSON parsing dengan try-catch.

### Multi-slot pakai timestamp

- Filename pakai timestamp `save_{slot}_{date}.json` untuk versioning manual.

---

## Latihan

1. **Backup save sebelum overwrite**: rename file lama jadi `.bak` sebelum tulis baru.
2. **Save metadata di file terpisah**: `save_1.meta.json` berisi `{ playerName, totalDays, screenshot }`. Tampilkan di main menu list save.
3. **Auto-save tiap N menit** (real time): coroutine di SaveManager.
4. **Encrypt save** sederhana: XOR dengan password sebelum write. Decrypt saat load. (Note: gak secure beneran, tapi cegah cheater amatir.)
5. **Cloud save Steamworks**: ikut Steamworks SDK setup. Skip kalau gak punya developer account.

---

## Recap

- [x] SaveData class dengan player, inventory, time, crop, wallet, watering can.
- [x] ItemDatabase + CropDatabase untuk lookup by id.
- [x] SaveManager Save/Load/Delete dengan JsonUtility.
- [x] Auto-save di end-of-day (TimeManager.EndSleep).
- [x] PauseMenu UI dengan 3 manual save slot + autosave (slot 0).
- [x] BootLoader auto-load di Start.

Game kamu sekarang **persistent**. Player bisa close, balik besok, lanjut main.

Saatnya **polish**: audio, music, build executable.

---

## Lanjut

[**Chapter 11 — Audio, Polish & Build →**](11-polish-build.md)

[← Chapter 9](09-shop-npc.md) | [Daftar Isi](../README.md)
