# Chapter 7 — Inventory & UI

> "Inventory yang bagus = pemain mau coba item baru. Inventory jelek = pemain frustrasi sebelum gameplay."

Hotbar 5 slot di Chapter 5 oke untuk testing, tapi pemain butuh **bag** (inventory utama dengan 24-36 slot), drag-drop antar slot, **stacking** otomatis, dan UI yang dibuka pakai tombol `I` atau `Tab`.

---

## Tujuan Chapter

- `InventoryManager` dengan 1 inventory utama: 36 slot total (5 hotbar + 31 bag, atau 5+25, dll).
- API: `Add(item, count)`, `Remove(item, count)`, `MoveSlot(from, to)`.
- **Auto-stack** saat pickup: kalau slot existing dengan item sama belum penuh, isi dulu.
- UI inventory popup dengan grid slot 6×5 atau 6×6.
- Drag-and-drop antar slot.
- Tooltip muncul saat hover (nama item + deskripsi).
- Hotbar UI tetap visible di bawah saat inventory closed.
- Auto pickup item yang dijatuhkan ke world.

---

## Prasyarat

- [Chapter 6](06-farming-mechanics.md) selesai.

---

## Konsep: Slot-Based Inventory

Pendekatan paling umum: **slot-based**. Tiap slot punya `ItemSO + count`. Berbeda dengan inventory grid Diablo (item bisa multi-cell), kita pakai slot 1×1 fixed.

```
Inventory:
[Hoe x1] [WateringCan x1] [CarrotSeed x10] [Scythe x1] [empty]   <- hotbar (5)
[empty] [empty] [empty] [empty] [empty] [empty]                  <- bag row 1
[empty] [empty] [empty] [empty] [empty] [empty]                  <- bag row 2
[Carrot x12] [empty] [empty] [empty] [empty] [empty]            <- bag row 3
[empty] [empty] [empty] [empty] [empty] [empty]                  <- bag row 4
[empty] [empty] [empty] [empty] [empty] [empty]                  <- bag row 5

Total: 35 slot (5 + 6×5)
```

Slot 0-4 = hotbar (juga bisa di-akses oleh hotbar UI).

---

## Step 1 — Refactor `Hotbar` jadi `InventoryManager`

Hotbar dari Chapter 5 sudah punya struktur Slot. Kita extend jadi InventoryManager dengan 35 slot (5 hotbar + 30 bag).

`Assets/_Project/Scripts/Inventory/InventoryManager.cs`:

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

namespace FarmingCourse.Inventory
{
    /// <summary>
    /// Inventory utama: hotbar (5 slot) + bag (30 slot).
    /// Slot 0-4 hotbar, 5-34 bag.
    /// Pengganti Hotbar singleton dari Chapter 5.
    /// </summary>
    public class InventoryManager : MonoBehaviour
    {
        public static InventoryManager Instance { get; private set; }

        [System.Serializable]
        public class Slot
        {
            public ItemSO item;
            public int count;
            public bool IsEmpty => item == null || count <= 0;
        }

        public const int HotbarSize = 5;
        public const int BagSize = 30;
        public const int TotalSize = HotbarSize + BagSize;

        [SerializeField] private List<Slot> slots = new List<Slot>();
        [SerializeField] private int selectedHotbarIndex = 0;

        public event Action OnInventoryChanged;
        public event Action<int> OnHotbarSelectionChanged;

        public int SelectedHotbarIndex => selectedHotbarIndex;
        public Slot SelectedHotbarSlot => slots[selectedHotbarIndex];
        public ItemSO SelectedHotbarItem => SelectedHotbarSlot.IsEmpty ? null : SelectedHotbarSlot.item;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;

            while (slots.Count < TotalSize) slots.Add(new Slot());
        }

        public Slot GetSlot(int index)
        {
            return (index >= 0 && index < slots.Count) ? slots[index] : null;
        }

        public bool IsHotbarSlot(int index) => index >= 0 && index < HotbarSize;

        // ---- HOTBAR SELECT ----
        public void SelectHotbar(int index)
        {
            if (!IsHotbarSlot(index)) return;
            selectedHotbarIndex = index;
            OnHotbarSelectionChanged?.Invoke(index);
        }

        // ---- ADD ITEM ----
        /// <summary>
        /// Tambah item. Auto-stack ke slot existing dulu (yang sama, belum full),
        /// sisa-nya isi slot kosong.
        /// </summary>
        /// <returns>Jumlah yang gagal ditambahkan (0 = semua masuk).</returns>
        public int Add(ItemSO item, int amount)
        {
            if (item == null || amount <= 0) return amount;
            int remaining = amount;

            // Pass 1: stack ke slot existing.
            for (int i = 0; i < slots.Count && remaining > 0; i++)
            {
                var s = slots[i];
                if (s.IsEmpty || s.item != item) continue;
                int canAdd = item.maxStackSize - s.count;
                if (canAdd <= 0) continue;
                int add = Mathf.Min(canAdd, remaining);
                s.count += add;
                remaining -= add;
            }

            // Pass 2: isi slot kosong.
            for (int i = 0; i < slots.Count && remaining > 0; i++)
            {
                var s = slots[i];
                if (!s.IsEmpty) continue;
                int add = Mathf.Min(item.maxStackSize, remaining);
                s.item = item;
                s.count = add;
                remaining -= add;
            }

            if (remaining < amount) OnInventoryChanged?.Invoke();
            return remaining; // 0 kalau semua masuk
        }

        // ---- REMOVE ITEM ----
        /// <summary>
        /// Buang `amount` instance dari item ini, dimulai dari slot terakhir.
        /// </summary>
        public bool Remove(ItemSO item, int amount)
        {
            if (item == null || amount <= 0) return false;
            int has = CountOf(item);
            if (has < amount) return false;

            int remaining = amount;
            for (int i = slots.Count - 1; i >= 0 && remaining > 0; i--)
            {
                var s = slots[i];
                if (s.IsEmpty || s.item != item) continue;
                int take = Mathf.Min(s.count, remaining);
                s.count -= take;
                remaining -= take;
                if (s.count <= 0) s.item = null;
            }

            OnInventoryChanged?.Invoke();
            return true;
        }

        public int CountOf(ItemSO item)
        {
            if (item == null) return 0;
            int total = 0;
            foreach (var s in slots)
            {
                if (!s.IsEmpty && s.item == item) total += s.count;
            }
            return total;
        }

        public bool RemoveFromSlot(int index, int amount = 1)
        {
            var s = GetSlot(index);
            if (s == null || s.IsEmpty || s.count < amount) return false;
            s.count -= amount;
            if (s.count <= 0) s.item = null;
            OnInventoryChanged?.Invoke();
            return true;
        }

        // ---- SWAP / MOVE SLOT ----
        public void SwapSlots(int a, int b)
        {
            if (a == b) return;
            var sa = GetSlot(a);
            var sb = GetSlot(b);
            if (sa == null || sb == null) return;

            // Stack-merge kalau item sama.
            if (!sa.IsEmpty && !sb.IsEmpty && sa.item == sb.item)
            {
                int canAdd = sa.item.maxStackSize - sb.count;
                int move = Mathf.Min(canAdd, sa.count);
                sb.count += move;
                sa.count -= move;
                if (sa.count <= 0) sa.item = null;
            }
            else
            {
                // Plain swap.
                var tmpItem = sa.item; var tmpCount = sa.count;
                sa.item = sb.item; sa.count = sb.count;
                sb.item = tmpItem; sb.count = tmpCount;
            }

            OnInventoryChanged?.Invoke();
        }
    }
}
```

### Penjelasan

- `Add` dua-pass (stack first, kemudian fill empty) supaya stack tetap tertangani benar.
- `Remove` dari belakang — biar item yang baru di-pickup gak duluan dihapus.
- `SwapSlots` smart — kalau item sama, merge stack instead of swap.
- Event `OnInventoryChanged` cuma fire kalau ada perubahan.

---

## Step 2 — Migrasi dari Hotbar ke InventoryManager

Semua tempat yang reference `Hotbar.Instance` ganti ke `InventoryManager.Instance`. Method-namespace juga sedikit beda:

| Lama (Hotbar) | Baru (InventoryManager) |
|--------------|-------------------------|
| `SelectedItem` | `SelectedHotbarItem` |
| `SelectedSlot` | `SelectedHotbarSlot` |
| `Select(int)` | `SelectHotbar(int)` |
| `ConsumeFromSelected(n)` | `RemoveFromSlot(SelectedHotbarIndex, n)` |
| `OnSelectionChanged` | `OnHotbarSelectionChanged` |
| `OnHotbarChanged` | `OnInventoryChanged` |

Ganti di `PlayerToolHandler.cs` dan `HotbarUI.cs`.

Hapus komponen Hotbar lama dari `_Managers` GameObject. Add Component → InventoryManager. Set default slot 0-4 di Inspector.

> Optional: hapus file `Hotbar.cs` setelah konfirmasi gak ada reference lagi.

---

## Step 3 — Tambahkan Crop Harvest Masuk Inventory

Edit `CropTileManager.TryHarvest`:

```csharp
public bool TryHarvest(Vector3Int cell)
{
    var data = GetTile(cell);
    if (data == null || data.state != TileState.Planted || !data.IsMature) return false;

    // Add ke inventory.
    if (FarmingCourse.Inventory.InventoryManager.Instance != null && data.crop.harvestItem != null)
    {
        int leftover = FarmingCourse.Inventory.InventoryManager.Instance.Add(data.crop.harvestItem, data.crop.harvestAmount);
        if (leftover > 0)
        {
            // Inventory penuh -- spawn item di world (Step 6 nanti). 
            Debug.LogWarning($"[CropTileManager] Inventory full, {leftover}x {data.crop.harvestItem.displayName} dropped to ground.");
        }
    }

    // ... regrow logic sama seperti sebelumnya
    if (data.crop.regrowsAfterHarvest)
    {
        data.growthStage = data.crop.regrowStage;
        UpdateCropVisualStage(cell, data);
    }
    else
    {
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
```

Test harvest → carrot masuk inventory bag. Lihat di Inspector InventoryManager → slot 5 atau slot kosong terdekat akan terisi.

---

## Step 4 — Bikin UI Inventory Window

### Layout

Canvas (yang sudah ada dari Chapter 5) → klik kanan → UI → **Panel** → namain `InventoryWindow`.

Setup Inventory Window:

- Anchor: stretch fill parent? Tidak — center, size 600×400.
- Background: Image dengan sprite paper / wood texture (Sprout Lands "UI" pack atau bikin sendiri).

Dalam `InventoryWindow`:

1. **Title**: TMP Text "Inventory" di atas.
2. **Close Button** (X) di kanan atas.
3. **Bag Grid**: GameObject dengan **Grid Layout Group** component. Spacing 4, cell size 64×64.
4. Add 30 child GameObject `Slot` di Bag Grid (atau spawn dynamic by code). Tiap slot = Image (background) + Image (icon child) + Text TMP (count child).

Untuk simplicity: bikin **Slot Prefab**.

### Slot Prefab

1. Hierarchy → di `Bag Grid`, klik kanan → UI → Image → namain `BagSlot`. Set Color = warna kotak.
2. `BagSlot` → klik kanan → UI → Image → namain `Icon`. Set anchor stretch, padding 8.
3. `BagSlot` → klik kanan → UI → Text - TextMeshPro → namain `Count`. Set anchor bottom-right, font small, color white with shadow.

Drag `BagSlot` ke `Prefabs/UI/InventorySlotUI.prefab`. Hapus dari Hierarchy.

### Spawn Slot Dinamis di Inventory Window

Bikin script:

`Assets/_Project/Scripts/UI/InventoryWindowUI.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.Inventory;

namespace FarmingCourse.UI
{
    public class InventoryWindowUI : MonoBehaviour
    {
        [SerializeField] private Transform bagGridParent;
        [SerializeField] private GameObject slotPrefab;
        [SerializeField] private GameObject windowRoot;
        [SerializeField] private TooltipUI tooltip;

        private List<InventorySlotUI> bagSlots = new List<InventorySlotUI>();
        private bool isOpen = false;

        private void Start()
        {
            BuildBagSlots();
            if (InventoryManager.Instance != null)
                InventoryManager.Instance.OnInventoryChanged += Refresh;
            Close();
            Refresh();
        }

        private void OnDestroy()
        {
            if (InventoryManager.Instance != null)
                InventoryManager.Instance.OnInventoryChanged -= Refresh;
        }

        private void BuildBagSlots()
        {
            for (int i = 0; i < InventoryManager.BagSize; i++)
            {
                var go = Instantiate(slotPrefab, bagGridParent);
                var slotUI = go.GetComponent<InventorySlotUI>();
                int actualIndex = InventoryManager.HotbarSize + i;
                slotUI.Init(actualIndex, tooltip);
                bagSlots.Add(slotUI);
            }
        }

        public void Toggle()
        {
            if (isOpen) Close(); else Open();
        }

        public void Open()
        {
            isOpen = true;
            windowRoot.SetActive(true);
            Refresh();
        }

        public void Close()
        {
            isOpen = false;
            windowRoot.SetActive(false);
            tooltip?.Hide();
        }

        public void Refresh()
        {
            if (InventoryManager.Instance == null) return;
            for (int i = 0; i < bagSlots.Count; i++)
            {
                int index = InventoryManager.HotbarSize + i;
                bagSlots[i].SetSlot(InventoryManager.Instance.GetSlot(index));
            }
        }
    }
}
```

`Assets/_Project/Scripts/UI/InventorySlotUI.cs`:

```csharp
using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;
using FarmingCourse.Inventory;

namespace FarmingCourse.UI
{
    public class InventorySlotUI : MonoBehaviour, IPointerClickHandler, IPointerEnterHandler, IPointerExitHandler, IBeginDragHandler, IDragHandler, IEndDragHandler, IDropHandler
    {
        [SerializeField] private Image icon;
        [SerializeField] private TMPro.TextMeshProUGUI countText;
        [SerializeField] private Image backgroundImage;

        public int slotIndex { get; private set; }
        private InventoryManager.Slot currentSlot;
        private TooltipUI tooltip;

        private static InventorySlotUI dragSource; // global drag-state

        public void Init(int index, TooltipUI tooltip)
        {
            slotIndex = index;
            this.tooltip = tooltip;
        }

        public void SetSlot(InventoryManager.Slot slot)
        {
            currentSlot = slot;
            if (slot == null || slot.IsEmpty)
            {
                icon.enabled = false;
                countText.text = "";
            }
            else
            {
                icon.enabled = true;
                icon.sprite = slot.item.icon;
                countText.text = slot.count > 1 ? slot.count.ToString() : "";
            }
        }

        public void OnPointerClick(PointerEventData ev) { /* opsional: split stack via shift+click */ }

        public void OnPointerEnter(PointerEventData ev)
        {
            if (currentSlot != null && !currentSlot.IsEmpty && tooltip != null)
            {
                tooltip.Show(currentSlot.item, ev.position);
            }
        }

        public void OnPointerExit(PointerEventData ev)
        {
            tooltip?.Hide();
        }

        public void OnBeginDrag(PointerEventData ev)
        {
            if (currentSlot == null || currentSlot.IsEmpty) return;
            dragSource = this;
            // Visual feedback: tipiskan icon di slot.
            icon.color = new Color(1, 1, 1, 0.4f);
        }

        public void OnDrag(PointerEventData ev)
        {
            // Optional: spawn ghost icon yang ngikutin cursor. Skip untuk simplicity.
        }

        public void OnEndDrag(PointerEventData ev)
        {
            if (dragSource == this) icon.color = Color.white;
            // Kalau drag lepas di luar UI -> drop ke world (Step 7).
            dragSource = null;
        }

        public void OnDrop(PointerEventData ev)
        {
            if (dragSource == null || dragSource == this) return;
            InventoryManager.Instance.SwapSlots(dragSource.slotIndex, slotIndex);
            dragSource.icon.color = Color.white;
            dragSource = null;
        }
    }
}
```

### Tooltip

`Assets/_Project/Scripts/UI/TooltipUI.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Inventory;

namespace FarmingCourse.UI
{
    public class TooltipUI : MonoBehaviour
    {
        [SerializeField] private GameObject root;
        [SerializeField] private TMPro.TextMeshProUGUI titleText;
        [SerializeField] private TMPro.TextMeshProUGUI descText;
        [SerializeField] private Vector2 cursorOffset = new Vector2(20, -20);

        private RectTransform rt;

        private void Awake()
        {
            rt = root.GetComponent<RectTransform>();
            Hide();
        }

        public void Show(ItemSO item, Vector2 screenPos)
        {
            root.SetActive(true);
            titleText.text = item.displayName;
            descText.text = item.description;
            rt.position = screenPos + cursorOffset;
        }

        public void Hide()
        {
            root.SetActive(false);
        }
    }
}
```

UI assembly:

1. Bikin TooltipUI prefab di Canvas: Panel kecil dengan title + desc TMP.
2. Drag script TooltipUI ke panel. Set field root, titleText, descText.
3. InventoryWindowUI script di InventoryWindow: drag bagGridParent (GameObject Bag Grid), slotPrefab (BagSlot prefab), windowRoot (root Panel), tooltip (TooltipUI panel).

### Hook Open/Close to Input

Edit `PlayerInputHandler.cs` (sudah ada `OnOpenInventoryPressed` event). Buat script kecil `InventoryToggle.cs`:

```csharp
using UnityEngine;
using FarmingCourse.UI;
using FarmingCourse.Player;

namespace FarmingCourse.UI
{
    public class InventoryToggle : MonoBehaviour
    {
        [SerializeField] private InventoryWindowUI inventoryWindow;
        [SerializeField] private PlayerInputHandler playerInput;

        private void OnEnable()
        {
            if (playerInput != null)
                playerInput.OnOpenInventoryPressed += OnToggle;
        }

        private void OnDisable()
        {
            if (playerInput != null)
                playerInput.OnOpenInventoryPressed -= OnToggle;
        }

        private void OnToggle()
        {
            inventoryWindow.Toggle();
        }
    }
}
```

Drag ke Player atau ke `_Managers`. Set field-nya.

Pencet Play → tekan `I` → window kebuka. Drag-drop antar slot. ✅

---

## Step 5 — Hotbar UI Refactor (Pakai InventorySlotUI)

Hotbar UI dari Chapter 5 masih dipakai. Konsistenkan dengan slot UI baru:

- Hotbar 5 slot ditampilkan di bawah screen (di luar Inventory Window).
- Slot-slotnya juga InventorySlotUI prefab → ke index 0..4.

Edit `HotbarUI.cs` (Chapter 5) untuk pakai `InventoryManager` & `InventorySlotUI`:

```csharp
using UnityEngine;
using FarmingCourse.Inventory;

namespace FarmingCourse.UI
{
    public class HotbarUI : MonoBehaviour
    {
        [SerializeField] private InventorySlotUI[] slotUIs; // size 5
        [SerializeField] private Color selectedColor = new Color(1, 1, 0.4f);
        [SerializeField] private Color unselectedColor = new Color(1, 1, 1, 0.7f);

        private void Start()
        {
            for (int i = 0; i < slotUIs.Length; i++)
            {
                slotUIs[i].Init(i, null /* no tooltip dipakai di hotbar simple */);
            }

            if (InventoryManager.Instance != null)
            {
                InventoryManager.Instance.OnInventoryChanged += Refresh;
                InventoryManager.Instance.OnHotbarSelectionChanged += _ => Refresh();
            }
            Refresh();
        }

        private void OnDestroy()
        {
            if (InventoryManager.Instance != null)
            {
                InventoryManager.Instance.OnInventoryChanged -= Refresh;
            }
        }

        private void Refresh()
        {
            if (InventoryManager.Instance == null) return;
            for (int i = 0; i < slotUIs.Length; i++)
            {
                slotUIs[i].SetSlot(InventoryManager.Instance.GetSlot(i));
                // Highlight selected.
                var img = slotUIs[i].GetComponent<UnityEngine.UI.Image>();
                if (img != null) img.color = (i == InventoryManager.Instance.SelectedHotbarIndex) ? selectedColor : unselectedColor;
            }
        }
    }
}
```

Setup HotbarPanel di Canvas: 5 InventorySlotUI prefab instantiated as children dengan Horizontal Layout Group. Drag ke `slotUIs` array di HotbarUI script.

---

## Step 6 — Drop Item ke World (Item Pickup)

Saat drag item ke luar window, drop ke ground.

### ItemPickup GameObject

Bikin prefab `Assets/_Project/Prefabs/Items/ItemPickup.prefab`:

1. GameObject → 2D Object → Sprites → Square → namain `ItemPickup`.
2. Sprite renderer (ditimpa runtime), Order in Layer 5.
3. Add Component:
   - **CircleCollider2D** with `Is Trigger` = true, radius 0.3.
   - **Rigidbody2D**, BodyType = Kinematic.
4. Tambah script `ItemPickup.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.Inventory
{
    public class ItemPickup : MonoBehaviour
    {
        [SerializeField] private SpriteRenderer spriteRenderer;
        public ItemSO item;
        public int amount = 1;

        public void Configure(ItemSO item, int amount)
        {
            this.item = item;
            this.amount = amount;
            if (spriteRenderer != null && item != null) spriteRenderer.sprite = item.icon;
        }

        private void OnTriggerEnter2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            if (item == null) return;
            int leftover = InventoryManager.Instance.Add(item, amount);
            if (leftover < amount)
            {
                amount = leftover;
                if (amount <= 0) Destroy(gameObject);
            }
        }
    }
}
```

5. Drag ke `Prefabs/Items/`. Set Player tag = `Player` (Inspector Player → tag dropdown).

### Spawn Pickup di Drop

Edit `InventorySlotUI.OnEndDrag`:

```csharp
public void OnEndDrag(PointerEventData ev)
{
    if (dragSource == this) icon.color = Color.white;
    // Kalau drop di luar UI raycast.
    if (!IsPointerOverUI(ev) && currentSlot != null && !currentSlot.IsEmpty)
    {
        // Drop ke world di posisi player.
        var player = GameObject.FindWithTag("Player");
        if (player != null && itemPickupPrefab != null)
        {
            var go = Instantiate(itemPickupPrefab, player.transform.position + (Vector3)Random.insideUnitCircle * 0.5f, Quaternion.identity);
            var pickup = go.GetComponent<ItemPickup>();
            pickup.Configure(currentSlot.item, currentSlot.count);
            InventoryManager.Instance.RemoveFromSlot(slotIndex, currentSlot.count);
        }
    }
    dragSource = null;
}

[SerializeField] private GameObject itemPickupPrefab;

private bool IsPointerOverUI(PointerEventData ev)
{
    var raycastResults = new System.Collections.Generic.List<RaycastResult>();
    EventSystem.current.RaycastAll(ev, raycastResults);
    foreach (var r in raycastResults) if (r.gameObject.GetComponent<UnityEngine.UI.Graphic>() != null) return true;
    return false;
}
```

Drag `ItemPickup.prefab` ke field di slot prefab. Pencet Play → drag carrot keluar window → muncul ItemPickup di sebelah player → langsung di-pickup.

---

## Step 7 — Money / Coin Counter

Sederhana — bikin `WalletManager` singleton:

`Assets/_Project/Scripts/Inventory/WalletManager.cs`:

```csharp
using System;
using UnityEngine;

namespace FarmingCourse.Inventory
{
    public class WalletManager : MonoBehaviour
    {
        public static WalletManager Instance { get; private set; }

        [SerializeField] private int coins = 0;
        public int Coins => coins;
        public event Action<int> OnCoinsChanged;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
        }

        public void Add(int amount)
        {
            coins = Mathf.Max(0, coins + amount);
            OnCoinsChanged?.Invoke(coins);
        }

        public bool Spend(int amount)
        {
            if (coins < amount) return false;
            coins -= amount;
            OnCoinsChanged?.Invoke(coins);
            return true;
        }
    }
}
```

UI: Canvas → Text TMP `CoinText` di pojok kanan atas. Sederhana script untuk update:

```csharp
public class CoinUI : MonoBehaviour
{
    [SerializeField] private TMPro.TextMeshProUGUI text;
    private void Start()
    {
        if (WalletManager.Instance != null)
        {
            WalletManager.Instance.OnCoinsChanged += OnCoins;
            OnCoins(WalletManager.Instance.Coins);
        }
    }
    private void OnCoins(int v) { text.text = $"{v}g"; }
}
```

Untuk dapat coin: nanti di Chapter 9 (Shop). Atau cheat: tekan tombol di-debug.

---

## Step 8 — Tooltip Polish (Optional)

Tooltip current cuma title + desc. Kamu bisa tambah:

- Sell price.
- Crop info (kalau Seed: "Harvest: Carrot, 4 days").
- Tool durability.

Edit `TooltipUI.Show()` dengan switch case berdasarkan tipe item.

---

## Troubleshooting

### Drag-drop gak jalan

- Canvas punya **Graphic Raycaster** component? (Auto kalau pakai UI → Canvas).
- EventSystem ada di scene? (Auto saat bikin Canvas pertama).
- Slot prefab punya **Image** dengan **Raycast Target** ✅?

### Tooltip stuck on screen

- `Hide()` panggil di OnPointerExit?
- Gak ada element lain yang nutupin tooltip canvas order.

### Drag dari Hotbar ke Bag pindahkan pakai SwapSlots

- Pastikan `Init(slotIndex)` dipanggil benar untuk semua slot UI (hotbar 0-4, bag 5-34).

### Inventory window window-nya invisible

- `windowRoot.SetActive(true)` dipanggil?
- Canvas Render Mode: Screen Space - Overlay?
- Sort Order di atas world?

---

## Latihan

1. **Split stack**: shift+klik kanan slot dengan stack > 1 → split jadi 2 stack di slot kosong terdekat.
2. **Sort inventory button**: tombol di Inventory Window → urutkan slot bag berdasarkan tipe (tools dulu, seed, crop) lalu alphabetical.
3. **Trash slot**: slot khusus di window — apa pun yang di-drop ke sana hilang.
4. **Money currency UI**: animasi coin saat di-Add (counter naik bertahap, bukan langsung).
5. **Inventory persistence**: simpan slots ke PlayerPrefs JSON saat scene unload, load saat scene start. Sneak peek Chapter 10.

---

## Recap

- [x] InventoryManager dengan 35 slot (5 hotbar + 30 bag).
- [x] Add, Remove, SwapSlots dengan auto-stack.
- [x] UI Inventory Window: grid + drag-and-drop antar slot.
- [x] Hotbar UI tetap visible, share data dengan inventory.
- [x] Tooltip on hover.
- [x] Drop item ke world → ItemPickup spawn → auto-collect.
- [x] WalletManager + CoinUI.

Inventory full-feature siap. Saatnya nge-flow waktu — siang malam, hari, season.

---

## Lanjut

[**Chapter 8 — Time, Day/Night & Seasons →**](08-time-day-night.md)

[← Chapter 6](06-farming-mechanics.md) | [Daftar Isi](index.md)
