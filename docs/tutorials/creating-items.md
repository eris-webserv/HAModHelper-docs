# Creating Custom Items

This tutorial guides you through creating, registering, rendering, and modifying custom items in *Hybrid Animals* using the `HAModHelper` API.

---

## Overview

Items live in the `HAModHelper.GamePlugin.Items.Systems` namespace, split across a few focused pieces:

* **`ItemManager`:** Registers items, looks them up, and keeps them in sync with the game's live item cache.
* **`WorldPrefabManager`:** Links custom 3D models (from your own Unity `AssetBundle`) to items that can be dropped or placed in the world.

> [!TIP]
> Injecting your item into crafting menus is covered separately in [Injecting Crafting Recipes](crafting-recipes.md) — it uses a different manager (`CraftingInjectionManager`) and isn't required just to make an item exist.

---

## Step 1: Define & Register an Item

Items are created by instantiating an `Item` model and registering it with `ItemManager.Instance`.

### Item Properties

| Property | Type | Description |
| :--- | :--- | :--- |
| `ModId` | `string` | Your mod's unique identifier (e.g., `"MyMod"`). |
| `ItemId` | `string` | The item's identifier (e.g., `"EtheriteOre"`). |
| `Id` | `string` | Read-only. Computed as `"{ModId}:{ItemId}"` — this is the full ID you pass to `GetItem`, crafting injection, etc. |
| `Name` | `string` | In-game display name visible in menus and inventories. |
| `Description` | `string?` | Optional item tooltip description text. |
| `StackLimit` | `int` | Maximum amount allowed in a single inventory slot (defaults to `1`). |
| `Actions` | `ItemActions` | Flags enum (`IsTool`, `IsUsable`, `IsConsumable`, `IsPlaceable`). **Not wired up to any game behavior yet** — setting these currently has no in-game effect. Safe to leave at the default. |
| `SpritePath` | `string?` | Inventory icon. See [Sprite resolution](#sprite-resolution) below — this is not a filesystem path. |
| `ExtraFields` | `Dictionary<string, string>` | Escape hatch for any raw game field HAModHelper doesn't model yet, including keys with spaces (e.g. `World_obj_path`, used in [Step 2](#step-2-registering-3d-world-prefabs)). |

<p align="center">
    <img src="../../resources/item_inventory_preview.png" alt="Custom Item Inventory Preview"/>
</p>

---

### Registration Example: Creating "Etherite Ore"

Two ways to get an item fully working end to end, depending on whether you have your own art:

#### Example A: Using Vanilla Assets (No Custom AssetBundle)

Reuses an existing vanilla sprite and world prefab (Titanium Ore's) — zero extra registration
steps beyond `AddItem`.

```csharp
using HAModHelper.GamePlugin.Items.Systems;

// 1. Instantiate and configure the Etherite Ore item
Item etheriteOre = new Item
{
    ModId = "MyMod",
    ItemId = "EtheriteOre",
    Name = "Etherite Ore",
    Description = "A glowing, mysterious ore pulsating with mystical energy.",
    StackLimit = 64,
    SpritePath = "item titanium ore"
};

// 2. Link a world prefab path via ExtraFields
etheriteOre.ExtraFields["World_obj_path"] = "Buildables/BiomeBuildables/Titanium Vein";
etheriteOre.ExtraFields["Type"] = "Place_in_world";

// 3. Register the item with ItemManager
ItemManager.Instance.AddItem(etheriteOre);
```

#### Example B: Using Your Own Custom Assets

Same item, but pointing at a custom sprite and a custom 3D model from your own Unity `AssetBundle`
instead of reusing vanilla ones. See [Loading Custom Assets](loading-assets.md) for how to build the
`AssetBundle` itself, [Sprite Resolution](#sprite-resolution) below for why the sprite line alone
isn't enough to actually show an icon yet, and [Step 2](#step-2-registering-3d-world-prefabs) below
for what `WorldPrefabManager.Register` is doing for the prefab.

```csharp
using System.IO;
using System.Reflection;
using UnityEngine;
using HAModHelper.GamePlugin.Items.Systems;

// 1. Instantiate and configure the Etherite Ore item
Item etheriteOre = new Item
{
    ModId = "MyMod",
    ItemId = "EtheriteOre",
    Name = "Etherite Ore",
    Description = "A glowing, mysterious ore pulsating with mystical energy.",
    StackLimit = 64
};

// 2. Raw field names via ExtraFields. Inventory_sprite_path here is identical to setting
//    Item.SpritePath -- ExtraFields always wins (see ItemConverter.ToGameFields) -- shown
//    this way so it's clear this is just a game field, not special AssetBundle-aware behavior.
etheriteOre.ExtraFields["Inventory_sprite_path"] = "item etherite ore";
etheriteOre.ExtraFields["World_obj_path"] = "Buildables/BiomeBuildables/Etherite Vein";
etheriteOre.ExtraFields["Type"] = "Place_in_world";

// 3. Register the item with ItemManager
ItemManager.Instance.AddItem(etheriteOre);

// 4. Load your own embedded AssetBundle (see Loading Custom Assets for how to embed it) and
//    register its prefab for this item's World_obj_path. "Etherite Vein" is the GameObject's
//    name inside the bundle -- read it straight from the bundle's own container manifest if
//    you're not sure what you named it in Unity.
//
// AssetBundle.LoadFromMemory is stripped from this IL2CPP build, and LoadFromStream needs an
// Il2CppSystem.IO.Stream rather than a regular .NET one, so the embedded resource is read fully
// and handed to Il2CppSystem.IO.MemoryStream instead.
Assembly assembly = Assembly.GetExecutingAssembly();
using Stream stream = assembly.GetManifestResourceStream("MyMod.mymod.bundle")!;
using var memory = new MemoryStream();
stream.CopyTo(memory);
AssetBundle bundle = AssetBundle.LoadFromStream(new Il2CppSystem.IO.MemoryStream(memory.ToArray()));

WorldPrefabManager.Instance.Register(
    itemFullId: etheriteOre.Id,
    worldObjPath: etheriteOre.ExtraFields["World_obj_path"],
    bundle: bundle,
    assetName: "Etherite Vein"
);
```

Call `ItemManager.Instance.AddItem(...)` from your plugin's `Load()` method — unlike perks (see [Creating Perks](creating-perks.md)), items are safe to register at startup: `AddItem` queues the item internally and HAModHelper automatically flushes that queue the first time the game asks for it, so you don't need to delay or retry anything.

### Sprite Resolution

`SpritePath` (or the equivalent raw `ExtraFields["Inventory_sprite_path"]`, as in Example B above) is passed straight through to the game as-is — HAModHelper does **not** resolve it to a file on disk. In practice this means:

* **Vanilla sprite names work out of the box** (e.g. `"item titanium ore"`, as used in Example A above, or `"item egg"` as used in this project's own [`Debug.cs`](../../src/HAModHelper.GamePlugin/Base/Debug.cs)) — the base game already knows how to find those.
* **A custom key pointing at your own `AssetBundle`-loaded sprite will not resolve**, as in Example B above (`"item etherite ore"` renders blank in-game). There is currently no `SpriteManager`-style hook (unlike world prefabs, see below) that lets HAModHelper serve a custom sprite when the base game can't find one in its own catalog. The world prefab in the same example works fine regardless, since `WorldPrefabManager` *does* exist for that case.

---

## Step 2: Registering 3D World Prefabs

Example A above reused a vanilla `World_obj_path` ("Buildables/BiomeBuildables/Titanium Vein"), so it
needed nothing further — the base game already knows how to place it. This step is only for items
that need their **own** 3D model, as shown in Example B: register it from an `AssetBundle` using
`WorldPrefabManager.Instance`. See [Loading Custom Assets](loading-assets.md) for how to build and
embed the `AssetBundle` itself.

```csharp
using System.IO;
using System.Reflection;
using UnityEngine;
using HAModHelper.GamePlugin.Items.Systems;

// Load your own embedded Unity AssetBundle (see Loading Custom Assets for how to embed it)
Assembly assembly = Assembly.GetExecutingAssembly();
using Stream stream = assembly.GetManifestResourceStream("MyMod.mymod.bundle")!;
using var memory = new MemoryStream();
stream.CopyTo(memory);
AssetBundle myModBundle = AssetBundle.LoadFromStream(new Il2CppSystem.IO.MemoryStream(memory.ToArray()));

// Register the 3D model asset for Etherite Ore
WorldPrefabManager.Instance.Register(
    itemFullId: "MyMod:EtheriteOre",
    worldObjPath: "Buildables/BiomeBuildables/Etherite Vein",
    bundle: myModBundle,
    assetName: "Etherite Vein"
);
```

* `worldObjPath` must exactly match the value you set on `item.ExtraFields["World_obj_path"]` in Step 1 — that's how `WorldPrefabManager` knows which prefab belongs to which item.
* `bundle` needs to stay loaded for the lifetime of your mod. Keep a static/instance reference to it in your plugin — if the `AssetBundle` gets garbage collected, the prefab lookup will fail the next time it's needed.
* This exists specifically for items whose `World_obj_path` **isn't** already in the base game's Addressables catalog (i.e. any item you made up yourself). Vanilla `World_obj_path` values resolve normally without registering anything here.

<p align="center">
    <img src="../../resources/item_world_prefab.gif" alt="Custom Item Dropped in World" width="340"/>
</p>

---

## Step 3: Modifying or Deleting Existing Items

You can also use `ItemManager` to edit runtime properties of vanilla or modded items, or remove unwanted items completely.

```csharp
using HAModHelper.GamePlugin.Items.Systems;

// 1. Fetch an existing item (e.g., vanilla Wood)
Item? wood = ItemManager.Instance.GetItem("base:Wood");

if (wood != null)
{
    // Modify stack limit and apply updates at runtime
    wood.StackLimit = 999;
    wood.UpdateItem(); // Calls ItemManager.Instance.PatchItem(wood)
}

// 2. Remove an item entirely from the game registry
Item? unwantedItem = ItemManager.Instance.GetItem("base:UnwantedItem");
if (unwantedItem != null)
{
    ItemManager.Instance.DeleteItem(unwantedItem);
}
```

> [!WARNING]
> `GetItem` can return `null` even for a valid ID if the game's underlying `ResourceControl` object isn't loaded yet (e.g. very early in startup, before the main scene exists). Always null-check the result, and prefer calling `GetItem` from a point after the game has finished loading rather than directly in `Load()`.

---

## Next Steps

* [Injecting Crafting Recipes](crafting-recipes.md) — make your new item craftable at vanilla stations.
* [Creating Perks](creating-perks.md) — register abilities, which can reference your items via `ExtraFields`.
* [Using the Event Bus](event-bus.md) — react to or fire your own custom mod events.
