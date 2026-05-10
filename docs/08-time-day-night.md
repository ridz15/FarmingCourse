# Chapter 8 — Time, Day/Night & Seasons

> "Time turns a sandbox into a story."

Stardew Valley nge-flow karena waktu jalan. Crop tumbuh, NPC pulang malam, hari ganti, musim ganti, festival datang. Di chapter ini kita bikin **TimeManager** yang men-drive semua sistem lain.

---

## Tujuan Chapter

- `TimeManager` singleton: jam, menit, hari, season, year.
- Game time mengalir dengan rasio configurable (1 detik real = 1 menit game default).
- **Day/night cycle visual**: warna ambient berubah dari pagi → siang → senja → malam.
- **Sleep system**: tekan B di rumah → fade to black → next day → CropTileManager.TickDay() → restore.
- Season cycle: 28 hari per musim, 4 musim, year++.
- Crop validity check: gak bisa plant kalau bukan season-nya.

---

## Prasyarat

- [Chapter 7](07-inventory-system.md) selesai (CropTileManager.TickDay() siap).

---

## Step 1 — `TimeManager`

`Assets/_Project/Scripts/Time/TimeManager.cs`:

```csharp
using System;
using UnityEngine;
using FarmingCourse.Farming;

namespace FarmingCourse.Time
{
    public class TimeManager : MonoBehaviour
    {
        public static TimeManager Instance { get; private set; }

        [Header("Time Settings")]
        [Tooltip("Berapa detik real-time = 1 menit dalam game.")]
        [SerializeField] private float realSecondsPerGameMinute = 1f;

        [Tooltip("Mulai jam berapa saat hari baru?")]
        [SerializeField] private int dayStartHour = 6;
        [SerializeField] private int dayEndHour = 26; // 2 AM next day -> akan auto-sleep

        [Header("Season")]
        [SerializeField] private int daysPerSeason = 28;

        [Header("State (Read-Only di runtime)")]
        [SerializeField] private int hour = 6;
        [SerializeField] private int minute = 0;
        [SerializeField] private int day = 1;          // 1..28 in season
        [SerializeField] private Season season = Season.Spring;
        [SerializeField] private int year = 1;

        public int Hour => hour;
        public int Minute => minute;
        public int Day => day;
        public Season Season => season;
        public int Year => year;
        public bool IsTimeFlowing { get; set; } = true;

        // Total minutes elapsed in current day (0..1440).
        public int MinutesIntoDay => hour * 60 + minute;
        public bool IsNight => hour < 6 || hour >= 19;

        // Events.
        public event Action<int, int> OnTimeChanged;       // hour, minute
        public event Action<int, Season, int> OnDayChanged; // day, season, year
        public event Action<Season, int> OnSeasonChanged;
        public event Action OnSleepStart;
        public event Action OnSleepEnd;

        private float minuteAccumulator = 0f;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
        }

        private void Update()
        {
            if (!IsTimeFlowing) return;

            minuteAccumulator += UnityEngine.Time.deltaTime;
            if (minuteAccumulator >= realSecondsPerGameMinute)
            {
                minuteAccumulator -= realSecondsPerGameMinute;
                AdvanceMinute();
            }
        }

        private void AdvanceMinute()
        {
            minute++;
            // Snap ke increments of 10 supaya UI gak update tiap menit
            // (Stardew style: "10:00 -> 10:10 -> 10:20").
            // Comment out kalau mau resolusi 1 menit.
            // if (minute % 10 != 0) return;

            if (minute >= 60)
            {
                minute = 0;
                hour++;
            }

            if (hour >= dayEndHour)
            {
                // Auto-sleep di luar jam normal.
                Debug.Log("[TimeManager] Player passed out at 2 AM!");
                StartSleep();
                return;
            }

            OnTimeChanged?.Invoke(hour, minute);
        }

        // ============================================================
        //  SLEEP
        // ============================================================

        public void StartSleep()
        {
            IsTimeFlowing = false;
            OnSleepStart?.Invoke();
        }

        /// <summary>
        /// Selesaikan sleep -> ganti hari, panggil TickDay di sistem yang relevan.
        /// </summary>
        public void EndSleep()
        {
            // Advance day.
            day++;
            if (day > daysPerSeason)
            {
                day = 1;
                AdvanceSeason();
            }

            hour = dayStartHour;
            minute = 0;
            IsTimeFlowing = true;

            // Notify systems.
            CropTileManager.Instance?.TickDay();
            OnDayChanged?.Invoke(day, season, year);
            OnTimeChanged?.Invoke(hour, minute);
            OnSleepEnd?.Invoke();
        }

        private void AdvanceSeason()
        {
            switch (season)
            {
                case Season.Spring: season = Season.Summer; break;
                case Season.Summer: season = Season.Fall; break;
                case Season.Fall: season = Season.Winter; break;
                case Season.Winter:
                    season = Season.Spring;
                    year++;
                    break;
            }
            OnSeasonChanged?.Invoke(season, year);
        }

        // ============================================================
        //  SAVE/LOAD HELPERS
        // ============================================================

        public TimeSaveData ExportSave() => new TimeSaveData
        {
            hour = hour, minute = minute, day = day, season = season, year = year
        };

        public void ImportSave(TimeSaveData data)
        {
            hour = data.hour; minute = data.minute; day = data.day;
            season = data.season; year = data.year;
            OnTimeChanged?.Invoke(hour, minute);
            OnDayChanged?.Invoke(day, season, year);
            OnSeasonChanged?.Invoke(season, year);
        }
    }

    [System.Serializable]
    public class TimeSaveData
    {
        public int hour, minute, day, year;
        public Season season;
    }
}
```

Drag ke `_Managers` GameObject.

### Test

Pencet Play. Tunggu 60 detik real → sebenarnya 60 menit game = 1 jam. Cek `_Managers` → TimeManager → fields `hour`, `minute` di Inspector berubah.

> Untuk test cepat, ubah `realSecondsPerGameMinute` jadi 0.05 → tiap 3 detik real = 1 jam game.

---

## Step 2 — Time UI

Bikin UI clock di pojok kanan atas Canvas:

1. Canvas → klik kanan → UI → Panel → namain `TimePanel`. Anchor top-right, size 240×100.
2. Anak: TMP Text untuk jam ("6:00 AM"), Day & Season ("Day 1, Spring"), Year ("Year 1").

Script `Assets/_Project/Scripts/UI/TimeUI.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Time;

namespace FarmingCourse.UI
{
    public class TimeUI : MonoBehaviour
    {
        [SerializeField] private TMPro.TextMeshProUGUI clockText;
        [SerializeField] private TMPro.TextMeshProUGUI dateText;

        private void Start()
        {
            if (TimeManager.Instance != null)
            {
                TimeManager.Instance.OnTimeChanged += OnTime;
                TimeManager.Instance.OnDayChanged += OnDay;
                OnTime(TimeManager.Instance.Hour, TimeManager.Instance.Minute);
                OnDay(TimeManager.Instance.Day, TimeManager.Instance.Season, TimeManager.Instance.Year);
            }
        }

        private void OnDestroy()
        {
            if (TimeManager.Instance != null)
            {
                TimeManager.Instance.OnTimeChanged -= OnTime;
                TimeManager.Instance.OnDayChanged -= OnDay;
            }
        }

        private void OnTime(int hour, int minute)
        {
            int displayHour = hour % 12;
            if (displayHour == 0) displayHour = 12;
            string suffix = (hour < 12 || hour >= 24) ? "AM" : "PM";
            clockText.text = $"{displayHour}:{minute:00} {suffix}";
        }

        private void OnDay(int day, FarmingCourse.Farming.Season season, int year)
        {
            dateText.text = $"{season} Day {day}, Year {year}";
        }
    }
}
```

Drag ke TimePanel. Set field-nya.

---

## Step 3 — Day/Night Visual (Color Tint)

Cara paling murah dan efektif: **overlay dark color full-screen**.

### Approach 1: Sprite Renderer Overlay (sederhana)

1. Hierarchy → 2D Object → Sprites → Square → namain `NightOverlay`.
2. Anak dari Main Camera (drag ke camera).
3. Set scale besar (50, 50, 1) supaya cover seluruh viewport.
4. Position relatif (0, 0, 1) — sedikit di depan camera (Z=1 ke depan kamera Z=-10? Justru sebaliknya. Set Z = 5 di depan kamera).
5. Sprite Renderer → Color: hitam dengan alpha = 0 saat siang.
6. Sorting Layer: layer baru `Overlay` dengan order tertinggi.

Script `Assets/_Project/Scripts/Time/DayNightOverlay.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Time;

namespace FarmingCourse.Time
{
    [RequireComponent(typeof(SpriteRenderer))]
    public class DayNightOverlay : MonoBehaviour
    {
        [SerializeField] private Gradient nightGradient;

        private SpriteRenderer sr;

        private void Awake()
        {
            sr = GetComponent<SpriteRenderer>();
        }

        private void Update()
        {
            if (TimeManager.Instance == null) return;
            float t = (TimeManager.Instance.MinutesIntoDay) / 1440f; // 0..1
            sr.color = nightGradient.Evaluate(t);
        }
    }
}
```

Set `nightGradient`:

- Time 0% (00:00): warna hitam, alpha 0.6.
- Time 25% (06:00): warna biru gelap, alpha 0.2.
- Time 33% (08:00): warna transparent (alpha 0).
- Time 70% (16:48): warna oranye soft, alpha 0.1.
- Time 80% (19:12): warna merah-oranye, alpha 0.4.
- Time 90% (21:36): warna ungu gelap, alpha 0.6.
- Time 100% (24:00): warna hitam, alpha 0.6.

(Pakai 7-key gradient untuk smooth transition. Edit di Inspector.)

### Approach 2: 2D Lights (URP Only, Lebih Cantik)

Kalau pakai URP:

- Window → Rendering → Light 2D Setup.
- Bikin Global Light 2D di scene → set color/intensity berdasarkan time of day.

Skip kalau pakai Built-in RP. Approach 1 cukup keren untuk pixel art.

---

## Step 4 — Sleep Trigger di Bed

Bikin GameObject "Bed" di rumah:

1. Hierarchy → 2D Object → Sprites → Square → namain `Bed`.
2. Sprite: drag asset bed dari Sprout Lands (kalau ada).
3. Posisi: di tengah area rumah (atau farm kalau gak ada indoor).
4. Add: BoxCollider2D, **Is Trigger = true**.
5. Layer: `Interactable`.

Script `Assets/_Project/Scripts/Interaction/BedInteraction.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Time;
using FarmingCourse.Player;

namespace FarmingCourse.Interaction
{
    [RequireComponent(typeof(Collider2D))]
    public class BedInteraction : MonoBehaviour
    {
        private bool playerInRange = false;
        private PlayerInputHandler playerInput;

        private void OnTriggerEnter2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            playerInRange = true;
            playerInput = other.GetComponent<PlayerInputHandler>();
            if (playerInput != null) playerInput.OnInteractPressed += TrySleep;
        }

        private void OnTriggerExit2D(Collider2D other)
        {
            if (!other.CompareTag("Player")) return;
            playerInRange = false;
            if (playerInput != null) playerInput.OnInteractPressed -= TrySleep;
        }

        private void TrySleep()
        {
            if (!playerInRange || TimeManager.Instance == null) return;
            // Saat tidur, mulai fade.
            SleepFade.Instance?.Sleep();
        }
    }
}
```

### SleepFade

Bikin GameObject `SleepFade` di Canvas:

- UI → Image → namain `SleepFadeOverlay`. Stretch full canvas.
- Color: hitam.
- Alpha mulai 0 (transparent).
- Disable di start.

Script `Assets/_Project/Scripts/UI/SleepFade.cs`:

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using FarmingCourse.Time;

namespace FarmingCourse.UI
{
    public class SleepFade : MonoBehaviour
    {
        public static SleepFade Instance { get; private set; }

        [SerializeField] private Image fadeImage;
        [SerializeField] private float fadeDuration = 1.2f;

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
            fadeImage.gameObject.SetActive(false);
        }

        public void Sleep()
        {
            StartCoroutine(SleepCoroutine());
        }

        private IEnumerator SleepCoroutine()
        {
            TimeManager.Instance?.StartSleep();

            fadeImage.gameObject.SetActive(true);
            // Fade in (jadi gelap).
            yield return FadeAlpha(0f, 1f, fadeDuration);
            yield return new WaitForSeconds(0.5f);

            // Trigger logical day change.
            TimeManager.Instance?.EndSleep();

            // Fade out (kembali terang).
            yield return FadeAlpha(1f, 0f, fadeDuration);
            fadeImage.gameObject.SetActive(false);
        }

        private IEnumerator FadeAlpha(float from, float to, float duration)
        {
            float t = 0f;
            Color c = fadeImage.color;
            while (t < duration)
            {
                t += UnityEngine.Time.unscaledDeltaTime; // unscaled supaya jalan saat IsTimeFlowing=false
                c.a = Mathf.Lerp(from, to, t / duration);
                fadeImage.color = c;
                yield return null;
            }
            c.a = to;
            fadeImage.color = c;
        }
    }
}
```

Drag ke `_Managers` (atau ke GameObject `SleepFadeOverlay` di canvas). Set field `fadeImage` ke Image.

Test: jalan ke Bed → tekan Space (Interact) → screen fade hitam → ganti hari → fade out.

---

## Step 5 — Crop Plant Validity by Season

Edit `CropTileManager.TryPlant`:

```csharp
public bool TryPlant(Vector3Int cell, SeedSO seed)
{
    var data = GetTile(cell);
    if (data == null || data.state == TileState.Untouched) return false;
    if (data.state == TileState.Planted) return false;
    if (seed == null || seed.cropToPlant == null) return false;

    // Cek season.
    if (TimeManager.Instance != null && seed.cropToPlant.validSeasons != null && seed.cropToPlant.validSeasons.Length > 0)
    {
        bool valid = System.Array.Exists(seed.cropToPlant.validSeasons, s => s == TimeManager.Instance.Season);
        if (!valid)
        {
            Debug.Log($"[CropTileManager] {seed.cropToPlant.displayName} tidak bisa ditanam di {TimeManager.Instance.Season}");
            return false;
        }
    }

    // ... rest sama
}
```

Carrot di-set valid di Spring & Summer. Coba plant saat Fall → fail.

### Crop Mati di End of Season

Kalau tanaman gak panen sampai akhir season (kecuali Winter Trees), mati. Tambah di `TimeManager.AdvanceSeason`:

```csharp
private void AdvanceSeason()
{
    // Kill all crops yang gak valid di season berikutnya (sederhana: kill semua yg bukan di season baru).
    Season newSeason;
    switch (season)
    {
        case Season.Spring: newSeason = Season.Summer; break;
        case Season.Summer: newSeason = Season.Fall; break;
        case Season.Fall: newSeason = Season.Winter; break;
        default: newSeason = Season.Spring; year++; break;
    }
    season = newSeason;
    OnSeasonChanged?.Invoke(season, year);

    // Notify CropTileManager untuk kill crop yang invalid.
    CropTileManager.Instance?.KillInvalidSeasonCrops(season);
}
```

Tambah method di `CropTileManager.cs`:

```csharp
public void KillInvalidSeasonCrops(Season newSeason)
{
    var keys = new List<Vector3Int>(tiles.Keys);
    foreach (var cell in keys)
    {
        var data = tiles[cell];
        if (data.state != TileState.Planted || data.crop == null) continue;
        bool valid = System.Array.Exists(data.crop.validSeasons, s => s == newSeason);
        if (!valid)
        {
            // Crop mati. Sprite jadi sprite "dead" (atau hilang).
            data.state = TileState.Tilled;
            data.crop = null;
            data.growthStage = 0;
            data.watered = false;
            farmableTilemap.SetTile(cell, tilledTile);
            DestroyCropVisual(cell);
            OnTileChanged?.Invoke(cell, data);
        }
    }
}
```

---

## Step 6 — Music Per Season (Opsional, Detail di Chapter 11)

Subscribe to `OnSeasonChanged` → load AudioClip music per season → play. Skip detail di sini.

---

## Step 7 — Skip Day Cheat (Untuk Test)

Sementara TimeManager memerlukan kamu jalan ke bed dan tidur. Tambah cheat untuk dev:

```csharp
private void Update()
{
    // ... existing time flow logic ...

#if UNITY_EDITOR
    if (Input.GetKeyDown(KeyCode.F5))
    {
        Debug.Log("[TimeManager] DEV: Skipping to next day.");
        // Force end day -- tanpa fade.
        EndSleep();
    }
#endif
}
```

Pencet F5 di Editor → ganti hari instan. Berguna saat test growth.

---

## Step 8 — Player Indoor Detection (Opsional)

Stardew player gak bisa tidur sembarang tempat — harus di bed. Sudah cover. Tapi kita bisa lock sleep saat di luar:

- Tambah trigger `IndoorRegion` collider gede di sekitar rumah.
- Player flag `IsIndoor`.
- Bed cek `IsIndoor` sebelum sleep.

Skip detail; opsional.

---

## Step 9 — Camera Brightness (Polish)

Saat malam, kamera bisa zoom-in dikit dan ada vignette. Cinemachine + URP volume profile bisa, tapi advanced. Skip.

---

## Troubleshooting

### Time gak jalan

- `IsTimeFlowing = true`?
- Komponen TimeManager active di scene?
- `realSecondsPerGameMinute` > 0?

### Crop gak tumbuh saat tidur

- `EndSleep` panggil `CropTileManager.Instance?.TickDay()`?
- CropTileManager.Instance null? Pastikan ada di scene.

### Day/night gradient gak smooth

- Gradient mode: `Blend` (bukan `Fixed`).
- Color keys 7+ untuk smooth.

### Bed interact gak respond

- Player tag = `Player`.
- Bed Collider Is Trigger = true.
- Layer Interactable, dan layer collision matrix Player vs Interactable enable.
- `playerInput` di-set saat OnTriggerEnter? Cek log.

### Sleep fade nge-stuck di hitam

- `unscaledDeltaTime` di coroutine, bukan `deltaTime` (karena IsTimeFlowing=false bisa pause Time.timeScale).
- Cek alpha image dapet nilai 0 di akhir.

---

## Latihan

1. **Weather system**: tambah enum `Weather { Sunny, Rainy }` di TimeManager. Random tiap pagi 20% chance rainy. Rainy day → otomatis water semua tile (CropTileManager.WaterAll()).
2. **Energy/stamina decreases**: setiap pakai tool kurangi energy player. Saat sleep, restore. Energy bar UI.
3. **NPC schedule**: NPC gerak ke posisi berbeda berdasarkan jam (pagi di rumah, siang di market, malam balik).
4. **Festival**: di hari tertentu (misal: Spring Day 13 = "Egg Festival"), trigger event special. Skip Time saat event aktif.

---

## Recap

- [x] TimeManager: hour, minute, day, season, year.
- [x] Time mengalir realtime dengan ratio configurable.
- [x] OnTimeChanged, OnDayChanged, OnSeasonChanged event.
- [x] Day/night overlay dengan gradient color.
- [x] Sleep trigger via Bed → fade hitam → TickDay.
- [x] Crop validity per season + auto-kill on season change.
- [x] Cheat F5 untuk skip day.

Game kita sekarang punya **flow waktu**. Saatnya bikin **economy**: shop & NPC.

---

## Lanjut

[**Chapter 9 — Shop & NPC →**](09-shop-npc.md)

[← Chapter 7](07-inventory-system.md) | [Daftar Isi](index.md)
