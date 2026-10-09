# Order-aware fork: Brains Off Mixology Helper

## Goal
Keep the plugin's "brains off" flow (fixed station-ordered batches, numbered levers, one station at a time), but make it aware of the three current Mastering Mixology orders, so it knows when to deliver, when to refill, and what extra potions to brew for the current orders.

This is a personal fork, run locally via the dev client (`MixologyBatchPluginTest.main` with `-ea`). It may later be published to the Plugin Hub or offered upstream as a PR.

## Game mechanics the plugin must respect
- 10 potions, each 3 pastes (M = Mox, A = Aga, L = Lye):
  MMM, AAA, LLL, MMA, MML, AAM, ALA, MLL, ALL, MAL.
- 3 processing stations:
  Alembic = Crystallise, Agitator = Homogenise, Retort = Concentrate.
- There are always 3 orders. Each order = (potion type, station/modifier).
- Clicking the conveyor deposits up to 3 potions that match current orders, from anywhere in the inventory.
- **If no order matches, clicking the conveyor deposits the FIRST potion in the inventory** for reduced rewards (10 resin, 1/3 XP). This wastes a prepared potion and is the main thing to prevent.
- Any deposit refreshes all 3 orders.
- Multi-order bonus: 2 orders in one deposit = +20% resin, 3 orders = +40%.

## Current plugin behaviour (from its README)
- The user configures a count (0-28) per potion. Copies are distributed round-robin across stations and flattened into station order.
- Refill trigger: during delivery, a new mixing queue starts when only Mixalots plus at most two other processed potions remain.
- Pulling any lever abandons delivery and starts a rolling refill around the remaining potions.
- Existing potions (finished or not) count toward configured totals.
- The "Guard wrong stations" option swaps the existing Check menu entry into left-click on stations that shouldn't be used. **Do not change this behaviour.**

## Features to add

### 1. Read current orders
- Read the 3 current orders (potion type + station) from game state.
- **Do not guess varbit/widget IDs.** Find how the established open-source Mixology plugins read orders (the "Mastering Mixology Helper" plugin and RuneLite's own Mixology code, also credited in THIRD_PARTY_NOTICES.md) and use the same source.
- Expose orders in the instruction panel, marking for each order whether a matching processed potion is in the inventory.

### 2. Smart deliver signal
- Matching = processed potion of the right type, processed at the right station.
- If >= N orders are fillable from the inventory: highlight the conveyor green (existing behaviour).
- If fewer than N are fillable: do NOT highlight the conveyor. Show a clear "Refill" state and start the refill queue.
- Config: `Deliver threshold` (1-3, default 1).
- Optional config: `Guard conveyor`, which works like Guard wrong stations: swap a harmless menu entry into left-click on the conveyor when 0 orders match. Only reorder existing entries; never remove, add or automate anything.

### 3. Current-order slots in refills
- When building a refill queue, check each current order. If it can't be filled from stock (processed or queued), add that potion to the mixing queue.
- Assign it to the correct station batch so it gets processed in the normal station loop. Mark it visually as an order potion (distinct from stock).
- Order potions use the free slots after the configured stock. If there's no room, show a warning; don't silently drop stock.
- Config: `Brew current orders` (on/off, default on).
- Config: `Skip order potions`, a set of potions never brewed for orders (default: MMM, AAA). These orders are left to be refreshed by a normal deposit.

## Hard constraints
- No automation: no clicks, no mouse movement, no input generation. Overlays, panel text and reordering existing menu entries only (the same scope the original plugin was approved with).
- Java 11, standard Plugin Hub Gradle build, no third-party runtime dependencies.
- Keep existing settings and defaults working. All new behaviour must be configurable, and with the new features off the plugin should behave exactly like the original.
- Keep original attribution and licence notices intact.

## Suggested approach
1. Read the codebase and summarise the architecture: plugin class, plan/queue builder, inventory tracking, overlays, config, menu swapping.
2. Locate the order data source in the reference plugins.
3. Propose an implementation plan with the files to change. **Wait for approval before writing code.**
4. Implement in small steps: (1) order reading + panel display, (2) smart deliver signal, (3) current-order slots, (4) conveyor guard.
5. Add or extend unit tests for the planning logic (order matching, queue building with order slots, threshold edge cases).

## Manual test checklist
- Orders shown in the panel match the in-game order UI.
- No fillable orders → conveyor not highlighted, refill starts.
- Fillable order → conveyor highlighted, deposit works as before.
- A current order not in stock gets brewed, lands in the correct station batch and is processed in the normal loop.
- Skipped potions (MMM/AAA) are never queued for orders.
- With all new options off, behaviour is identical to the original plugin.
