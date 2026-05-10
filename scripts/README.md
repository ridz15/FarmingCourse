# scripts/

This folder is reserved for the **complete, drop-in C# scripts** used in the
course chapters. They mirror the code blocks in `chapters/` so students can copy
working files directly when they get stuck.

## Status

The course currently embeds **all code inline inside the markdown chapters**.
This folder will be populated over time with structured `.cs` files matching
the namespaces used in the course:

```
scripts/
├── Player/
│   ├── PlayerInputHandler.cs
│   ├── PlayerMovement.cs
│   ├── PlayerAnimator.cs
│   ├── PlayerToolHandler.cs
│   └── PlayerFootsteps.cs
├── Inventory/
│   ├── ItemSO.cs
│   ├── BasicItemSO.cs
│   ├── ToolSO.cs
│   ├── SeedSO.cs
│   ├── InventoryManager.cs
│   ├── ItemDatabase.cs
│   ├── ItemPickup.cs
│   ├── WalletManager.cs
│   └── Tools/
│       ├── HoeToolSO.cs
│       ├── WateringCanToolSO.cs
│       └── ScytheToolSO.cs
├── Farming/
│   ├── CropSO.cs
│   ├── CropTileManager.cs
│   ├── CropDatabase.cs
│   └── WaterRefillTrigger.cs
├── Time/
│   ├── TimeManager.cs
│   └── DayNightOverlay.cs
├── NPC/
│   ├── NPCData.cs
│   └── NPC.cs
├── Shop/
│   └── ShopInventorySO.cs
├── Save/
│   ├── SaveData.cs
│   ├── SaveManager.cs
│   └── BootLoader.cs
├── Audio/
│   ├── AudioManager.cs
│   └── MusicSeasonController.cs
├── UI/
│   ├── HotbarUI.cs
│   ├── InventoryWindowUI.cs
│   ├── InventorySlotUI.cs
│   ├── TooltipUI.cs
│   ├── DialogueUI.cs
│   ├── ShopUI.cs
│   ├── ShopRowUI.cs
│   ├── TimeUI.cs
│   ├── CoinUI.cs
│   ├── SleepFade.cs
│   └── PauseMenu.cs
├── Utility/
│   ├── ScreenManager.cs
│   └── ImpulseHelper.cs
└── Interaction/
    └── BedInteraction.cs
```

## Want to help populate this folder?

See [CONTRIBUTING.md](../CONTRIBUTING.md). Pull requests welcome — copy the code
out of the chapter, add `.meta` files only if you also commit the matching
`.asset` files (we generally don't, since asset GUIDs differ per project).

## Why mirror code in two places?

- **Markdown** is great for *learning* — code is right next to the explanation.
- **Standalone files** are great for *reuse* — students with a working Unity
  project can drop these into `Assets/_Project/Scripts/` directly.

The chapter markdown remains the source of truth. If a chapter changes, this
folder needs to be updated to match.
