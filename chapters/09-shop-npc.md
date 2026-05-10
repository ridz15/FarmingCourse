# Chapter 9 — Shop & NPC

> "NPC jelek tetap NPC. Yang bikin mereka hidup itu konsistensi, bukan AI canggih."

Player butuh tempat **beli seed & jual hasil panen**. Stardew punya Pierre's General Store. Kita bikin versi sederhana: NPC `Trader`, dialog box, shop UI dengan tab Buy/Sell.

---

## Tujuan Chapter

- NPC `Trader` di tilemap dengan sprite & idle animation.
- Dialog system simpel (typewriter effect).
- Interact dengan NPC → pop dialog → option "Open Shop".
- Shop UI: Buy tab (item yang dijual NPC) + Sell tab (otomatis daftarkan item-item dari inventory yang `sellPrice > 0`).
- Beli/jual transact dengan WalletManager.
- ShopInventory sebagai ScriptableObject (bisa bikin shop berbeda untuk NPC berbeda).

---

## Prasyarat

- [Chapter 8](08-time-day-night.md) selesai (TimeManager, sleep system).
- WalletManager dari Chapter 7.

---

## Step 1 — `NPC` Component

`Assets/_Project/Scripts/NPC/NPCData.cs`:

```csharp
using UnityEngine;

namespace FarmingCourse.NPC
{
    [CreateAssetMenu(fileName = "NPC", menuName = "FarmingCourse/NPC/NPC Data")]
    public class NPCData : ScriptableObject
    {
        public string npcId;
        public string displayName;
        public Sprite portrait;       // dipakai di dialog box
        [TextArea(3, 6)]
        public string greetingText;   // baris pertama saat interact
        public ShopInventorySO shop;  // null = NPC ini gak punya shop
    }
}
```

`Assets/_Project/Scripts/NPC/NPC.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Player;
using FarmingCourse.UI;

namespace FarmingCourse.NPC
{
    [RequireComponent(typeof(Collider2D))]
    public class NPC : MonoBehaviour
    {
        [SerializeField] private NPCData data;

        private bool playerInRange;
        private PlayerInputHandler playerInput;

        private void OnTriggerEnter2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            playerInRange = true;
            playerInput = other.GetComponent<PlayerInputHandler>();
            if (playerInput != null) playerInput.OnInteractPressed += OnInteract;
        }

        private void OnTriggerExit2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            playerInRange = false;
            if (playerInput != null) playerInput.OnInteractPressed -= OnInteract;
            DialogueUI.Instance?.Hide();
        }

        private void OnInteract()
        {
            if (!playerInRange || data == null) return;
            DialogueUI.Instance?.ShowGreeting(data, OnGreetingDone);
        }

        private void OnGreetingDone()
        {
            // Setelah greeting, kalau ada shop → tampilkan tombol "Open Shop".
            if (data.shop != null)
            {
                DialogueUI.Instance?.ShowShopOption(data, OpenShop);
            }
        }

        private void OpenShop()
        {
            DialogueUI.Instance?.Hide();
            ShopUI.Instance?.Open(data.shop);
        }
    }
}
```

---

## Step 2 — `DialogueUI`

UI Layout:

1. Canvas → klik kanan → UI → Panel → namain `DialogueBox`. Anchor bawah, sticky bottom, height 200, width full.
2. Anak: portrait (Image kotak kiri), name text TMP, body text TMP.
3. Tombol `Continue` (kanan bawah).
4. Tombol `Open Shop` (hidden by default).

`Assets/_Project/Scripts/UI/DialogueUI.cs`:

```csharp
using System;
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.NPC;

namespace FarmingCourse.UI
{
    public class DialogueUI : MonoBehaviour
    {
        public static DialogueUI Instance { get; private set; }

        [SerializeField] private GameObject root;
        [SerializeField] private Image portrait;
        [SerializeField] private TMPro.TextMeshProUGUI nameText;
        [SerializeField] private TMPro.TextMeshProUGUI bodyText;
        [SerializeField] private Button continueButton;
        [SerializeField] private Button shopButton;

        [SerializeField] private float typeSpeed = 30f;

        private Coroutine typeCoroutine;
        private Action onContinueDone;
        private Action onShopClicked;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
            root.SetActive(false);
            continueButton.onClick.AddListener(OnContinueClicked);
            shopButton.onClick.AddListener(OnShopClicked);
        }

        public void ShowGreeting(NPCData data, Action onDone)
        {
            root.SetActive(true);
            shopButton.gameObject.SetActive(false);
            portrait.sprite = data.portrait;
            nameText.text = data.displayName;

            onContinueDone = onDone;
            if (typeCoroutine != null) StopCoroutine(typeCoroutine);
            typeCoroutine = StartCoroutine(TypewriterCoroutine(data.greetingText));
        }

        public void ShowShopOption(NPCData data, Action onShop)
        {
            shopButton.gameObject.SetActive(data.shop != null);
            onShopClicked = onShop;
        }

        public void Hide()
        {
            root.SetActive(false);
            if (typeCoroutine != null) StopCoroutine(typeCoroutine);
        }

        private IEnumerator TypewriterCoroutine(string text)
        {
            bodyText.text = "";
            float charsPerSecond = typeSpeed;
            float t = 0f;
            int displayed = 0;
            while (displayed < text.Length)
            {
                t += UnityEngine.Time.unscaledDeltaTime;
                int target = Mathf.FloorToInt(t * charsPerSecond);
                if (target > text.Length) target = text.Length;
                if (target > displayed)
                {
                    displayed = target;
                    bodyText.text = text.Substring(0, displayed);
                }
                yield return null;
            }
            bodyText.text = text;
        }

        private void OnContinueClicked()
        {
            // Kalau text masih ngetik, skip ke akhir.
            if (typeCoroutine != null) StopCoroutine(typeCoroutine);
            // (Kalau user mau next page, bisa expand ini ke array of texts.)
            onContinueDone?.Invoke();
        }

        private void OnShopClicked()
        {
            onShopClicked?.Invoke();
        }
    }
}
```

Drag ke `_Managers` atau ke `DialogueBox` di Canvas. Set field references.

Test: spawn NPC GameObject di scene dengan NPCData asset → walk ke NPC → tekan Space → dialog muncul.

---

## Step 3 — `ShopInventorySO`

`Assets/_Project/Scripts/Shop/ShopInventorySO.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using FarmingCourse.Inventory;

namespace FarmingCourse.Shop
{
    [CreateAssetMenu(fileName = "ShopInventory", menuName = "FarmingCourse/Shop/Inventory")]
    public class ShopInventorySO : ScriptableObject
    {
        [System.Serializable]
        public class StockEntry
        {
            public ItemSO item;
            public int priceOverride = -1; // -1 = pakai item.buyPrice
            public bool unlimited = true;
            public int stockCount;          // dipakai kalau unlimited=false
        }

        public List<StockEntry> stock = new List<StockEntry>();

        public int GetPrice(StockEntry e) => e.priceOverride >= 0 ? e.priceOverride : e.item.buyPrice;
    }
}
```

Buat asset `PierreShop.asset`:
- Stock: tambah CarrotSeed (price 30), Hoe (kalau pengen replace), dst.

---

## Step 4 — `ShopUI`

UI Layout:

1. Canvas → klik kanan → UI → Panel → `ShopWindow`. Center, size 800×500.
2. Tab buttons: `Buy`, `Sell` (top).
3. Item list (Scroll View) yang menampilkan StockEntry.
4. Right panel: item detail + price + Confirm button.
5. Coin display.

`Assets/_Project/Scripts/UI/ShopUI.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.Inventory;
using FarmingCourse.Shop;

namespace FarmingCourse.UI
{
    public class ShopUI : MonoBehaviour
    {
        public static ShopUI Instance { get; private set; }

        [SerializeField] private GameObject root;
        [SerializeField] private Transform listContent;
        [SerializeField] private GameObject shopRowPrefab;
        [SerializeField] private Button buyTabButton;
        [SerializeField] private Button sellTabButton;
        [SerializeField] private TMPro.TextMeshProUGUI coinText;

        private ShopInventorySO currentShop;
        private bool buyMode = true;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
            root.SetActive(false);
            buyTabButton.onClick.AddListener(() => SetMode(true));
            sellTabButton.onClick.AddListener(() => SetMode(false));
        }

        private void Start()
        {
            if (WalletManager.Instance != null)
            {
                WalletManager.Instance.OnCoinsChanged += UpdateCoinText;
            }
        }

        public void Open(ShopInventorySO shop)
        {
            currentShop = shop;
            root.SetActive(true);
            SetMode(true);
            UpdateCoinText(WalletManager.Instance?.Coins ?? 0);
        }

        public void Close()
        {
            root.SetActive(false);
            currentShop = null;
        }

        private void SetMode(bool buy)
        {
            buyMode = buy;
            RefreshList();
        }

        private void RefreshList()
        {
            // Clear children.
            foreach (Transform t in listContent) Destroy(t.gameObject);

            if (currentShop == null) return;

            if (buyMode)
            {
                foreach (var entry in currentShop.stock)
                {
                    if (entry.item == null) continue;
                    var go = Instantiate(shopRowPrefab, listContent);
                    var row = go.GetComponent<ShopRowUI>();
                    row.SetupBuy(entry, currentShop, OnBuy);
                }
            }
            else
            {
                if (InventoryManager.Instance == null) return;
                for (int i = 0; i < InventoryManager.TotalSize; i++)
                {
                    var slot = InventoryManager.Instance.GetSlot(i);
                    if (slot == null || slot.IsEmpty) continue;
                    if (slot.item.sellPrice <= 0) continue;
                    var go = Instantiate(shopRowPrefab, listContent);
                    var row = go.GetComponent<ShopRowUI>();
                    row.SetupSell(slot, i, OnSell);
                }
            }
        }

        private void OnBuy(ShopInventorySO.StockEntry entry)
        {
            int price = currentShop.GetPrice(entry);
            if (WalletManager.Instance == null || !WalletManager.Instance.Spend(price)) return;
            if (!entry.unlimited)
            {
                if (entry.stockCount <= 0) return;
                entry.stockCount--;
            }
            int leftover = InventoryManager.Instance.Add(entry.item, 1);
            if (leftover > 0)
            {
                // Inventory penuh → kembalikan uang.
                WalletManager.Instance.Add(price);
                if (!entry.unlimited) entry.stockCount++;
                Debug.Log("[Shop] Inventory full, refunded.");
            }
            RefreshList();
        }

        private void OnSell(int slotIndex)
        {
            var slot = InventoryManager.Instance.GetSlot(slotIndex);
            if (slot == null || slot.IsEmpty) return;
            int gain = slot.item.sellPrice;
            WalletManager.Instance.Add(gain);
            InventoryManager.Instance.RemoveFromSlot(slotIndex, 1);
            RefreshList();
        }

        private void UpdateCoinText(int v)
        {
            if (coinText != null) coinText.text = $"{v}g";
        }
    }
}
```

`Assets/_Project/Scripts/UI/ShopRowUI.cs`:

```csharp
using System;
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.Inventory;
using FarmingCourse.Shop;

namespace FarmingCourse.UI
{
    public class ShopRowUI : MonoBehaviour
    {
        [SerializeField] private Image icon;
        [SerializeField] private TMPro.TextMeshProUGUI nameText;
        [SerializeField] private TMPro.TextMeshProUGUI priceText;
        [SerializeField] private Button actionButton;
        [SerializeField] private TMPro.TextMeshProUGUI actionLabel;

        public void SetupBuy(ShopInventorySO.StockEntry entry, ShopInventorySO shop, Action<ShopInventorySO.StockEntry> onClick)
        {
            icon.sprite = entry.item.icon;
            nameText.text = entry.item.displayName;
            priceText.text = $"{shop.GetPrice(entry)}g";
            actionLabel.text = "Buy";
            actionButton.onClick.RemoveAllListeners();
            actionButton.onClick.AddListener(() => onClick(entry));
        }

        public void SetupSell(InventoryManager.Slot slot, int slotIndex, Action<int> onClick)
        {
            icon.sprite = slot.item.icon;
            nameText.text = $"{slot.item.displayName} (x{slot.count})";
            priceText.text = $"{slot.item.sellPrice}g";
            actionLabel.text = "Sell 1";
            actionButton.onClick.RemoveAllListeners();
            actionButton.onClick.AddListener(() => onClick(slotIndex));
        }
    }
}
```

Bikin prefab `ShopRowUI.prefab` di Canvas: panel horizontal dengan icon, nama, harga, button.

---

## Step 5 — Setup NPC di Scene

1. Hierarchy → 2D Object → Sprites → Square → namain `Pierre`.
2. Drag sprite NPC dari Sprout Lands ke Sprite Renderer.
3. Position di tile rumah/store (mis. (10, 5)).
4. Add Component:
   - **BoxCollider2D** with **Is Trigger**=true. Ukuran sedikit lebih besar dari sprite supaya player gampang trigger.
   - **NPC** script.
5. Bikin asset `PierreNPC.asset` (NPCData) dengan:
   - npcId = "pierre"
   - displayName = "Pierre"
   - portrait = drag sprite portrait
   - greetingText = "Selamat datang ke toko! Apa yang bisa kubantu?"
   - shop = drag `PierreShop.asset`
6. Drag asset `PierreNPC.asset` ke field `data` di NPC component.

---

## Step 6 — Idle Animation NPC (Optional)

NPC bisa breathe / idle animation:

- Bikin Animator Controller dengan satu state idle (2-frame loop).
- Attach ke NPC GameObject.

Skip kalau gak penting. NPC statis juga oke.

---

## Step 7 — NPC Schedule (Bonus, Nyimpan untuk Lanjutan)

Stardew NPC pindah-pindah berdasarkan jam. Implementasi simpel:

1. NPC punya array waypoint per `(jam, posisi)`.
2. Subscribe TimeManager.OnTimeChanged → cek apakah jam ini ada waypoint → MoveTo posisi pakai DOTween atau coroutine Lerp.

Skip detail. Latihan untuk Chapter 12.

---

## Step 8 — Test Flow

1. Spawn 50 carrot di inventory player (cheat code di TimeManager Update kalau perlu).
2. Walk ke Pierre → Space → dialog "Selamat datang...".
3. Klik Continue → tombol "Open Shop" muncul.
4. Klik Open Shop → ShopWindow muncul. Tab Buy, list CarrotSeed (30g).
5. Click Buy → coin berkurang, seed masuk inventory.
6. Switch tab Sell → list carrot (35g per).
7. Click Sell → coin naik, carrot berkurang dari inventory.

Close ShopWindow dengan tombol X (tambah tombol close di UI).

---

## Step 9 — Tutorial Hint System (Optional)

Saat first-time interact NPC, kasih tutorial popup. Skip detail.

---

## Troubleshooting

### Dialog gak muncul

- DialogueUI.Instance null? Pastikan komponen ada di scene.
- NPCData asset terisi di NPC component?
- Player tag = `Player`?

### Shop button gak muncul

- NPCData.shop terisi?
- ShowShopOption dipanggil di OnGreetingDone?

### Buy sukses tapi item gak masuk inventory

- InventoryManager.Instance null? Cek scene.
- Inventory penuh? Code refund balik dari WalletManager.

### Sell list kosong

- Item kamu punya `sellPrice > 0`?
- Slot di inventory ada item?

### Coin text gak update

- `WalletManager.Instance.OnCoinsChanged` event subscribed?

---

## Latihan

1. **Multi-page dialog**: extend NPCData jadi `string[] dialogueLines`. DialogueUI tampilkan baris per baris saat continue.
2. **Sell stack**: di sell tab, klik kanan untuk sell semua stack di slot itu.
3. **NPC Heart system**: tambah "friendship" di NPCData (int 0..10). Kasih gift item → naik. Sederhana mirror Stardew heart.
4. **Shop refresh harian**: stock terbatas → restock setiap hari (subscribe OnDayChanged).
5. **Special vendor di hari tertentu**: NPC "Traveling Cart" muncul di Spring Day 14 saja, dengan stock random.

---

## Recap

- [x] NPCData ScriptableObject + NPC component.
- [x] DialogueUI dengan typewriter effect & Continue button.
- [x] ShopInventorySO + ShopUI dengan tab Buy/Sell.
- [x] Beli & jual dengan WalletManager integration.
- [x] Inventory full → refund. Stock decrement opsional.

Player sekarang punya **economy loop**: beli seed → tanam → siram → tunggu → panen → jual → ulang.

Saatnya **save semua progress ke file** supaya gak ilang saat tutup game.

---

## Lanjut

[**Chapter 10 — Save / Load System →**](10-save-load.md)

[← Chapter 8](08-time-day-night.md) | [Daftar Isi](../README.md)
