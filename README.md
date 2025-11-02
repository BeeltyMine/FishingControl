FishingControl plugin — Events and usage
========================================

Overview
--------
FishingControl is a small example plugin that demonstrates how to use the server's fishing-related events to control casting, reeling, and loot generation. This README documents the three events the plugin demonstrates and gives short code examples for plugin authors.

Events
------
All event classes are under the `pocketmine\event` namespace.

1) PlayerFishCastEvent
   - Fired when a player attempts to cast a fishing hook (i.e. uses a fishing rod while no hook is already present).
   - Type: `pocketmine\event\player\PlayerFishCastEvent`
   - Cancellable: yes. If cancelled, the cast will not happen and no hook projectile is spawned.
   - Typical constructor: `new PlayerFishCastEvent(Player $player, FishingRod $rod)`
   - Use case: prevent casting in certain worlds/regions, implement per-world fishing rules.

   Example listener:
   ```php
   use pocketmine\event\player\PlayerFishCastEvent;

   public function onCast(PlayerFishCastEvent $e): void{
       if($e->getPlayer()->getWorld()->getFolderName() === "no-fishing-world"){
           $e->setCancelled(true);
           $e->getPlayer()->sendMessage("Fishing is disabled here.");
       }
   }
   ```

2) PlayerFishReelEvent
   - Fired when a player reels in an existing hook that they own.
   - Type: `pocketmine\event\player\PlayerFishReelEvent`
   - Cancellable: yes. If cancelled, the reeling action will be aborted (the hook remains).
   - Key methods:
     - `getHook(): ?Entity` — the FishHook entity being reeled, if present.
     - `getTargetEntity(): ?Entity` — an entity that was hooked (if any).
     - `getLoot(): ?Item` — the loot item the server intends to give (may be null).
     - `setLoot(?Item $item): void` — modify the loot (set to a different item) or set to null to drop/skip loot.
   - Use case: change what the player receives, prevent reeling in certain entities, add special behaviour based on player permissions.

   Example listener (force every catch to a diamond):
   ```php
   use pocketmine\event\player\PlayerFishReelEvent;
   use pocketmine\item\VanillaItems;

   public function onReel(PlayerFishReelEvent $e): void{
       $e->setLoot(VanillaItems::DIAMOND());
   }
   ```

   Notes:
   - The server will use the loot returned from this event when spawning/delivering the item. If you set a loot item here, it should be an instance of `pocketmine\item\Item` (for vanilla items use `VanillaItems::...()`).

3) FishHookLootEvent
   - Fired right before the server spawns the `ItemEntity` for loot from a fish hook.
   - Type: `pocketmine\event\entity\FishHookLootEvent`
   - Cancellable: yes. If cancelled, the item entity will not be spawned.
   - Key methods:
     - `getLoot(): Item` — the item that will be spawned.
     - `setLoot(Item $item): void` — replace the spawned item.
   - Use case: final chance to transform or prevent the item from appearing, or to add special metadata to the spawned item.

   Example listener:
   ```php
   use pocketmine\event\entity\FishHookLootEvent;
   use pocketmine\item\VanillaItems;

   public function onLoot(FishHookLootEvent $e): void{
       // Log and change the loot
       $this->getLogger()->info("Hook loot about to spawn: " . $e->getLoot()->getName());
       $e->setLoot(VanillaItems::DIAMOND());
   }
   ```

Notes and tips
--------------
- Event priorities and cancellation: handlers may declare `@priority` and `@handleCancelled` in their docblocks if you need fine-grained control. Handlers that check `isCancelled()` can decide whether to run or ignore.
- Registration: register your listener in `onEnable()` using `$this->getServer()->getPluginManager()->registerEvents($listener, $this);`.
- Ordering: `PlayerFishReelEvent` is fired before the server spawns the loot. `FishHookLootEvent` is fired immediately before the item entity is spawned — use whichever fits your needs.
- Testing: Use a combination of server console logs and in-game testing (cast and reel). The example plugin provides fallback closure registrations for the events, and also logs when the server creates and calls the events.

Multi-loot and append behavior
-----------------------------
The server now supports plugins providing multiple loot items when a player reels in a hook. There are two modes plugins can use:

1) Replace default loot (default): plugin calls `setLoot(...)` or `setLoots([...])`. The server will use exactly the items you provide instead of the vanilla loot.

2) Append to default loot: plugin calls `addLoot(...)` or `setLoots([...])` and then `setAppendLoot(true)` on the `PlayerFishReelEvent`. The server will keep the vanilla-generated loot as the first item and append your items after it.

Examples
--------
Replace vanilla loot with a single diamond:

```php
use pocketmine\event\player\PlayerFishReelEvent;
use pocketmine\item\VanillaItems;

public function onReel(PlayerFishReelEvent $e): void{
  $e->setLoot(VanillaItems::DIAMOND());
}
```

Provide multiple items and replace vanilla loot:

```php
public function onReel(PlayerFishReelEvent $e): void{
  $e->setLoots([VanillaItems::DIAMOND(), VanillaItems::GOLD_INGOT()]);
}
```

Append items to vanilla loot (vanilla loot kept first):

```php
public function onReel(PlayerFishReelEvent $e): void{
  $e->addLoot(VanillaItems::DIAMOND());
  $e->addLoot(VanillaItems::GOLD_INGOT());
  $e->setAppendLoot(true);
}
```

Notes about item spawning
------------------------
- For each loot item (vanilla or plugin-provided) the server fires a `FishHookLootEvent` immediately before spawning the `ItemEntity`. Handlers for `FishHookLootEvent` may `setLoot(...)` or `setLoots(...)` or `setCancelled(true)` to modify or cancel individual spawns.
- If you want to completely handle spawning yourself, you can call `setLoots([])` or `setLoot(null)` (or `setCancelled(true)` on the `FishHookLootEvent`) and then spawn your own `ItemEntity` instances at the desired location.

Testing instructions
--------------------
1) Start the server from the repository root (PowerShell):

```powershell
.\start.ps1
# or
.\start.cmd
```

2) Confirm the plugin loads: look for `[FishingControl] FishingControl enabled` in the console logs.

3) Join the server with a Bedrock client and equip a fishing rod.

4) Cast and reel several times. Watch the server console:
   - The server logs debug messages when it calls events (e.g. `FishingRod: calling PlayerFishReelEvent for <player>`).
   - The plugin logs `FishingControl: onReel fired for <player> loot=...` when your listener runs.
   - `FishHookLootEvent` logs appear as `--------FishHook spawning loot: <item>`.

5) To test append behavior specifically:
   - Modify your plugin listener to call `addLoot(...)` and `setAppendLoot(true)` as in the example above.
   - Reload/restart the server or re-enable the plugin, then cast and reel. You should see the vanilla loot plus your appended items spawn (and corresponding logs).

If anything doesn't behave as expected, paste the relevant console lines (the debug lines and plugin logs) and I'll help trace it further.

Compatibility
-------------
This plugin is intended to run on BeeltyMine (stable v5.x) and is known to work with the APIs in this repository. API class names and method availability may differ in older or custom builds.

Runs on BeeltyMine
------------------
Bu eklenti BeeltyMine üzerinde çalışır (BeeltyMine v5.x uyumludur).

Source / GitHub
----------------
GitHub: https://github.com/BeeltyMine/BeeltyMine

License
-------
FishingControl example plugin and this README are provided as examples; adapt and reuse them under the same license as the main project.
