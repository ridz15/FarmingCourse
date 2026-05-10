# Chapter 11 — Audio, Polish & Build

> "Polish adalah hal-hal kecil yang gak ada player notice — sampai mereka gak ada."

Game-nya playable. Tapi belum **mantap**. Di chapter ini kita kasih audio, screenshake, particles, juice, dan akhirnya **build** ke file executable.

---

## Tujuan Chapter

- Audio Manager dengan music + SFX system, volume control.
- BGM per season.
- SFX trigger-based: footstep, hoe swing, water splash, harvest, coin.
- Camera shake saat impact (hoe, harvest).
- Particle effects: dirt explosion saat till, water drips, harvest sparkle.
- Build setting & build executable (Windows / Mac / WebGL).

---

## Prasyarat

- [Chapter 10](10-save-load.md) selesai (game complete).

---

## Step 1 — Bikin AudioManager

`Assets/_Project/Scripts/Audio/AudioManager.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace FarmingCourse.Audio
{
    public class AudioManager : MonoBehaviour
    {
        public static AudioManager Instance { get; private set; }

        [Header("Audio Sources")]
        [SerializeField] private AudioSource musicSource;
        [SerializeField] private AudioSource sfxSource;
        [SerializeField] private AudioSource ambientSource;

        [Header("Volumes")]
        [Range(0, 1)] [SerializeField] private float musicVolume = 0.5f;
        [Range(0, 1)] [SerializeField] private float sfxVolume = 0.8f;

        public float MusicVolume
        {
            get => musicVolume;
            set { musicVolume = Mathf.Clamp01(value); musicSource.volume = musicVolume; }
        }

        public float SfxVolume
        {
            get => sfxVolume;
            set { sfxVolume = Mathf.Clamp01(value); sfxSource.volume = sfxVolume; }
        }

        private void Awake()
        {
            if (Instance != null && Instance != this) { Destroy(gameObject); return; }
            Instance = this;
            DontDestroyOnLoad(gameObject);
            ApplyVolumes();
        }

        private void ApplyVolumes()
        {
            if (musicSource != null) musicSource.volume = musicVolume;
            if (sfxSource != null) sfxSource.volume = sfxVolume;
            if (ambientSource != null) ambientSource.volume = sfxVolume * 0.6f;
        }

        public void PlayMusic(AudioClip clip, bool loop = true)
        {
            if (musicSource == null || clip == null) return;
            if (musicSource.clip == clip && musicSource.isPlaying) return;
            musicSource.clip = clip;
            musicSource.loop = loop;
            musicSource.Play();
        }

        public void StopMusic() => musicSource?.Stop();

        public void PlaySfx(AudioClip clip, float volumeScale = 1f, float pitch = 1f)
        {
            if (sfxSource == null || clip == null) return;
            sfxSource.pitch = pitch;
            sfxSource.PlayOneShot(clip, volumeScale * sfxVolume);
        }

        /// <summary>
        /// Play SFX dengan random pitch (untuk variation).
        /// </summary>
        public void PlaySfxRandomPitch(AudioClip clip, float minPitch = 0.9f, float maxPitch = 1.1f, float volumeScale = 1f)
        {
            float pitch = Random.Range(minPitch, maxPitch);
            PlaySfx(clip, volumeScale, pitch);
        }
    }
}
```

Drag ke `_Managers`. Tambah 3 child AudioSource GameObject (atau langsung 3 AudioSource component di same GameObject):

- **MusicSource** — Loop ✅, Spatial Blend = 0 (2D).
- **SfxSource** — Loop ❌, Spatial Blend = 0.
- **AmbientSource** — Loop ✅, Spatial Blend = 0, untuk birds chirping dll.

Drag references di Inspector AudioManager.

---

## Step 2 — `MusicSeasonController`

Music ganti per season:

`Assets/_Project/Scripts/Audio/MusicSeasonController.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Time;
using FarmingCourse.Farming;

namespace FarmingCourse.Audio
{
    public class MusicSeasonController : MonoBehaviour
    {
        [SerializeField] private AudioClip springMusic;
        [SerializeField] private AudioClip summerMusic;
        [SerializeField] private AudioClip fallMusic;
        [SerializeField] private AudioClip winterMusic;
        [SerializeField] private AudioClip nightMusic;

        private void Start()
        {
            if (TimeManager.Instance != null)
            {
                TimeManager.Instance.OnSeasonChanged += OnSeason;
                TimeManager.Instance.OnTimeChanged += OnTime;
                OnSeason(TimeManager.Instance.Season, TimeManager.Instance.Year);
            }
        }

        private void OnDestroy()
        {
            if (TimeManager.Instance != null)
            {
                TimeManager.Instance.OnSeasonChanged -= OnSeason;
                TimeManager.Instance.OnTimeChanged -= OnTime;
            }
        }

        private void OnSeason(Season season, int year)
        {
            if (TimeManager.Instance.IsNight) return;
            AudioClip clip = season switch
            {
                Season.Spring => springMusic,
                Season.Summer => summerMusic,
                Season.Fall => fallMusic,
                Season.Winter => winterMusic,
                _ => springMusic,
            };
            AudioManager.Instance?.PlayMusic(clip);
        }

        private void OnTime(int hour, int minute)
        {
            // Switch ke night music di malam.
            bool nowNight = TimeManager.Instance.IsNight;
            // (Logic untuk smooth transition bisa ditambah dengan FadeMusic coroutine.)
        }
    }
}
```

---

## Step 3 — SFX Triggers

Tambah `AudioClip` references di `ToolSO` & subclass-nya. Lalu di `ToolSO.Use` panggil `AudioManager.Instance?.PlaySfx(useSfx)`.

Sample SFX (cari di freesound.org):

- **Footstep on grass**: subscribe di Player movement, play tiap 0.4 detik saat moving.
- **Hoe thud**: dirt impact.
- **Water splash**: pour sound.
- **Plant seed**: light rustle.
- **Harvest**: snap.
- **Coin**: ka-ching.
- **Sleep**: zzz / fade.

### Footstep Implementation

`Assets/_Project/Scripts/Player/PlayerFootsteps.cs`:

```csharp
using UnityEngine;
using FarmingCourse.Audio;

namespace FarmingCourse.Player
{
    [RequireComponent(typeof(PlayerInputHandler))]
    public class PlayerFootsteps : MonoBehaviour
    {
        [SerializeField] private AudioClip[] grassFootsteps;
        [SerializeField] private float stepInterval = 0.4f;

        private PlayerInputHandler input;
        private float timer;

        private void Awake() { input = GetComponent<PlayerInputHandler>(); }

        private void Update()
        {
            if (!input.IsMoving) { timer = 0; return; }
            timer += Time.deltaTime;
            if (timer >= stepInterval)
            {
                timer = 0;
                if (grassFootsteps != null && grassFootsteps.Length > 0)
                {
                    var clip = grassFootsteps[Random.Range(0, grassFootsteps.Length)];
                    AudioManager.Instance?.PlaySfxRandomPitch(clip, 0.9f, 1.1f, 0.4f);
                }
            }
        }
    }
}
```

Drag ke Player. Set 4-5 grass-footstep clips.

---

## Step 4 — Particles

### Dirt Explosion saat Till

1. GameObject → Effects → Particle System → namain `DirtParticle`.
2. Customize:
   - Duration: 0.3
   - Start Lifetime: 0.4-0.6
   - Start Speed: 2
   - Start Size: 0.1
   - Start Color: warna coklat tanah (varied)
   - Emission: 0 (we'll burst manually)
   - Burst: 1 burst di time 0, count 8.
   - Shape: Cone, angle 25, radius 0.2.
   - Velocity over Lifetime: y-positive (terbang ke atas) lalu turun karena gravity.
   - Renderer: Material default 2D, Sorting Layer Default, order 4.

3. Save sebagai prefab `DirtBurstParticle.prefab`.

Spawn dari CropTileManager.TryTillSingle setelah berhasil:

```csharp
[SerializeField] private GameObject dirtBurstPrefab;

private bool TryTillSingle(Vector3Int cell)
{
    // ... existing logic ...
    if (dirtBurstPrefab != null)
    {
        Vector3 worldPos = farmableTilemap.GetCellCenterWorld(cell);
        var go = Instantiate(dirtBurstPrefab, worldPos, Quaternion.identity);
        Destroy(go, 1.5f);
    }
    // ...
}
```

### Water Splash saat Watering

Same pattern: prefab `WaterSplashParticle` dengan blue color, smaller.

### Harvest Sparkle

Cyan/yellow stars burst.

---

## Step 5 — Camera Shake (Cinemachine Impulse)

### Setup di Vcam

1. Pilih CM vcam1 → Add Extension → **Cinemachine Impulse Listener**.

### Trigger Impulse

Di tool action (e.g., HoeToolSO.Use after success):

```csharp
[SerializeField] private float impulseStrength = 0.05f;

public override bool Use(Vector3 userPosition, Vector3Int targetTile)
{
    bool success = CropTileManager.Instance.TryTill(targetTile, tillRadius);
    if (success) AudioManager.Instance?.PlaySfx(useSfx);
    if (success) ImpulseHelper.Generate(impulseStrength);
    return success;
}
```

`Assets/_Project/Scripts/Utility/ImpulseHelper.cs`:

```csharp
using Cinemachine;
using UnityEngine;

namespace FarmingCourse.Utility
{
    public class ImpulseHelper : MonoBehaviour
    {
        private static CinemachineImpulseSource source;

        public static void Generate(float strength)
        {
            if (source == null)
            {
                var go = new GameObject("ImpulseSource");
                source = go.AddComponent<CinemachineImpulseSource>();
                source.m_DefaultVelocity = Vector3.one;
            }
            source.GenerateImpulse(strength);
        }
    }
}
```

> **Note**: Kalau pakai Cinemachine 3.x (Unity 6), namespace berbeda (`Unity.Cinemachine`). Nama class juga: `CinemachineImpulseSource`.

---

## Step 6 — Hit Stop / Time Slow (Polish Lanjut)

Trigger `Time.timeScale = 0.05f` selama 0.05 detik saat aksi penting (harvest crop final stage). Ngasih feedback "punch".

Skip detail. Cari "hit stop unity" di YouTube untuk implementation.

---

## Step 7 — Settings UI Volume Slider

Tambahkan ke PauseMenu: 2 slider untuk Music & SFX.

```csharp
[SerializeField] private UnityEngine.UI.Slider musicSlider;
[SerializeField] private UnityEngine.UI.Slider sfxSlider;

private void Awake()
{
    // ... existing
    musicSlider.value = AudioManager.Instance?.MusicVolume ?? 0.5f;
    sfxSlider.value = AudioManager.Instance?.SfxVolume ?? 0.8f;
    musicSlider.onValueChanged.AddListener(v => { if (AudioManager.Instance != null) AudioManager.Instance.MusicVolume = v; });
    sfxSlider.onValueChanged.AddListener(v => { if (AudioManager.Instance != null) AudioManager.Instance.SfxVolume = v; });
}
```

Persistent: simpan ke PlayerPrefs:

```csharp
PlayerPrefs.SetFloat("musicVolume", v);
PlayerPrefs.Save();
```

Load di AudioManager.Awake:

```csharp
musicVolume = PlayerPrefs.GetFloat("musicVolume", 0.5f);
sfxVolume = PlayerPrefs.GetFloat("sfxVolume", 0.8f);
```

---

## Step 8 — Sprite Outline / Hover Effect (Polish)

Saat mouse hover di interactable (NPC, crop matang), tambah sparkle / glow.

Sederhana: Material outline shader. Setup pakai URP shader graph atau import asset gratis "Sprite Outline" dari Asset Store.

Skip detail.

---

## Step 9 — Build Settings

Saatnya export ke executable.

### File → Build Settings

- **Scenes In Build**: drag MainScene. Index 0.
- **Platform**: pilih Windows / Mac / Linux Standalone, atau WebGL.

### Player Settings (klik "Player Settings")

#### Resolution and Presentation

- **Default Screen Width**: 1920
- **Default Screen Height**: 1080
- **Fullscreen Mode**: Fullscreen Window.
- **Resolution Dialog**: Hidden by Default.

#### Splash Image (Personal / Unity Pro)

- Hilangkan Unity logo splash kalau punya Unity Pro. Free tier tetap nampil.

#### Other Settings

- **Color Space**: Linear (atau Gamma — Linear lebih akurat tapi sedikit beda look).
- **Auto Graphics API for Windows**: ✅
- **Static Batching / Dynamic Batching**: ✅

#### Publishing Settings

- **Configuration**: Release.
- **Compression Method**: LZ4HC (build size lebih kecil, build time sedikit lama).

---

## Step 10 — Build!

1. **File → Build And Run** (atau Build saja kalau gak mau auto-launch).
2. Pilih folder output: `Builds/Windows/`.
3. Tunggu 1-5 menit.
4. Setelah selesai, executable muncul: `StardewClone.exe`.

Test:

- Run executable.
- Verify game jalan, save bisa di-load.
- Check resolution, fullscreen toggle (Alt+Enter atau setting).

### WebGL Build

1. **Platform**: WebGL.
2. Player Settings → **Publishing Settings → Compression Format**: Brotli (lebih kecil) atau Gzip (kompatibel server lebih luas).
3. Build → output folder `Builds/WebGL/`.
4. Run via local web server: `cd Builds/WebGL && python -m http.server 8000` → buka `http://localhost:8000`.

> **WebGL gotchas**:
> - File access (Application.persistentDataPath) pakai IndexedDB. Save akan persist di browser.
> - First load lama (download 30-50 MB).
> - PostProcessing & beberapa shader gak full support. Test dulu.

### Mac & Linux

Pilih platform di Build Settings → install module via Unity Hub kalau belum ada → Build.

---

## Step 11 — Itch.io Publish (Optional)

Cara cepat share game:

1. Bikin akun di https://itch.io.
2. Dashboard → Create new project.
3. Upload zip dari `Builds/WebGL/` (untuk web playable) atau zip windows build (downloadable).
4. Set kind: HTML (kalau WebGL) atau Downloadable.
5. Set viewport size 1280×720 (atau sesuai).
6. Publish (atau set as draft).

URL itch.io kamu sekarang punya halaman game. Share ke teman.

---

## Step 12 — Performance Profiling

Sebelum release, profile dulu:

- Window → Analysis → **Profiler**.
- Pencet Play, biarkan profile running 30 detik.
- Lihat panel CPU, GPU, Memory.
- Spike >1ms? Klik untuk dive ke breakdown.

Common culprits:

- `GetComponent` di Update tiap frame.
- `Find / FindWithTag` per frame.
- Particle system overload.
- Many GameObjects with Update method.

Optimize: cache reference, pool particle system, use `IUpdate` manager pattern.

---

## Step 13 — Bug Hunt Final

Sebelum publish:

- [ ] Save & reload — pastikan semua state pulih.
- [ ] Crop loop full season — Spring → Summer → Fall → Winter → Spring (year 2). Tidak crash.
- [ ] Inventory full → harvest → item drop ke ground (tidak hilang).
- [ ] Buy item → coin terkurangi → item masuk inv.
- [ ] Sell item → item keluar → coin nambah.
- [ ] Sleep multiple kali — gak crash.
- [ ] Tekan tombol UI saat dialog terbuka — tidak konflik.
- [ ] Resize window → UI tidak rusak.
- [ ] Build executable jalan tanpa Unity Editor.

---

## Latihan

1. **Particle pooling**: instead of Instantiate/Destroy tiap till, pool 20 prefab dirt particle, reuse.
2. **Volume mixer**: pakai `AudioMixer` Unity untuk routing music/sfx ke groups, set volume via `audioMixer.SetFloat("MusicVolume", Mathf.Log10(v) * 20)` (logarithmic).
3. **Shader effect**: rain droplet shader saat `Weather.Rainy`. (Bonus dari Chapter 8.)
4. **Day intro card**: setiap pagi, tampilkan card "Day 5 - Spring" fade in/out 2 detik.
5. **Achievements**: tambah `AchievementManager`. Achievement: "First Harvest", "100 Carrots", "Reach Year 2", "Stardew (50 of one crop)".

---

## Recap

- [x] AudioManager dengan music + sfx + ambient channel.
- [x] MusicSeasonController auto-switch BGM per season.
- [x] Footstep, hoe, water, harvest SFX.
- [x] Particle effect dirt burst, water splash, sparkle.
- [x] Camera shake via Cinemachine Impulse.
- [x] Settings UI dengan volume slider + PlayerPrefs persist.
- [x] Build executable Windows / WebGL / Mac.

**Game kamu sudah jadi. Selamat!** 🎉

Tinggal satu chapter terakhir: arah lanjutan & resources.

---

## Lanjut

[**Chapter 12 — Next Steps & Resources →**](12-next-steps.md)

[← Chapter 10](10-save-load.md) | [Daftar Isi](../README.md)
