# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A RuneLite Plugin Hub plugin ("Brains Off Mixology Helper") that guides repeatable Mastering Mixology inventories. It is advisory only: it draws overlays and reorders existing menu entries, but never clicks, moves the mouse, or performs actions. Keep it that way — automation would make the plugin ineligible for the Plugin Hub.

Plugin Hub `build=standard` with no third-party runtime dependencies. Don't add runtime deps.

## Commands

```bash
./gradlew clean build                                   # compile, test, jar
./gradlew test                                          # tests only
./gradlew test --tests 'com.mixologybatch.BatchPlanTest'                       # one class
./gradlew test --tests 'com.mixologybatch.BatchPlanTest.defaultPlanMatchesRequestedInventory'   # one method
```

Dev client: run `MixologyBatchPluginTest.main` from an IDE. It is a launcher, not a test — it calls `ExternalPluginManager.loadBuiltin` then `RuneLite.main`. **Assertions must be enabled (`-ea`)** or RuneLite refuses to start.

### Windows environment constraint

Gradle's test worker cannot load classes from a path containing non-ASCII characters. On this machine the repo must stay at `C:\Repos\RuneLite` (not the `C:\Users\André…` profile), and `GRADLE_USER_HOME` is set to `C:\gradle-home`. Symptom if violated: every test fails with `ClassNotFoundException` on its own class, or `Could not find or load main class ...GradleWorkerMain`, while `compileJava` and `jar` still succeed. Check the path before debugging the code.

Toolchain is pinned to Java 11 in `build.gradle` to match CI (`ubuntu-latest`, Temurin 11).

## Architecture

Single package `com.mixologybatch`. Types are package-private and `final` by default; only `MixologyBatchPlugin`, `MixologyBatchConfig`, and `StationOrder` are public (RuneLite needs those).

### Two-layer planning

The central idea, and the thing that requires reading several files to see:

- **`BatchPlan`** — the *logical* plan, derived purely from config. Potion copies are distributed round-robin across stations (`copy % stations.length`), then flattened in station order into slots `0..N-1`. This is what makes each station's potions contiguous in the inventory, which is the whole point of the plugin: the minigame consumes the inventory top-left to bottom-right, so one station's batch finishes before the next begins. Immutable; rebuilt only on config change.
- **`CyclePlan`** — the *physical* mapping of that logical plan onto real inventory slots. Existing potions keep their slots; the lowest free slots are reserved for what's mixed next. Carries mutable observation state (`blockedSlots`, `persistentBlockedSlots`, `observedDigweedSlots`) updated by `observeInventory`.
- **`BatchEntry.remap(potion, slot)`** is the bridge — it rebinds a logical entry to an actual slot and actual potion, preserving station metadata.

So `plan.get(rank)` is "what the config wants", `cycle.entryAtRank(rank, actualPotion)` is "where it actually lives". Confusing the two is the easiest bug to introduce here.

### Resolution pipeline

```
Client varbits + inventory  →  MixologyBatchPlugin.updateState()
                            →  BatchStateResolver.resolve(...)
                            →  Guidance  →  4 overlays
```

`BatchStateResolver` takes plan, cycle, inventory, vessel potion, mixer slots, and active stations, and returns an immutable `Guidance` (Phase + Action + entry + component + step number + message). **It never touches `Client`** — that is why it carries the bulk of the test coverage. Keep new game-state logic there rather than in the plugin class, and add tests alongside.

`MixologyBatchPlugin` is the only class holding a `Client` reference. It owns event subscriptions and the `BatchMode` state machine.

### BatchMode state machine

`REFILLING` → `PROCESSING` → `FINISHING_PARTIAL`, with transitions concentrated in `updateState()`. Notable behaviors:

- Clicking the one still-available station during a partial inventory is read as "finish this partial batch" and sets `FINISHING_PARTIAL`.
- Pulling any lever while processing abandons the delivery and restarts a rolling refill.
- `shouldStartRollingRefill` deliberately ignores MAL (Mixalot) so leftover Mixalots carry into the next batch; the refill starts once only two non-MAL finished potions remain.

### Optimistic mix prediction

`MixPrediction` makes the queue advance on the **Mix** click rather than waiting for the animation. On click, the plugin assumes the tracked potion lands in the next empty slot and records baseline potion counts; `reconcileMixPrediction` later confirms against the real inventory, falling back to a count diff if the potion landed elsewhere, and expires after 8 game ticks. While a prediction is outstanding, `updateState` plans against a *projected* inventory with the predicted potion written in and the vessel/mixer cleared.

A wrong lever pull must not advance or rewrite the intended recipe — both `handleLeverClick` and `resolveMixing` deliberately keep showing the correct sequence instead of surfacing an error.

### Menu guard

`onPostMenuSort` + `StationMenuGuard` swap the station's **existing** `Check` entry into the left-click position when a station shouldn't be used. The processing entry is swapped, never deleted or duplicated, so it stays reachable via right-click. `MenuScan` groups entries by exact `(objectId, sceneX, sceneY)` so two nearby machines can't have their entries crossed.

### Game state sources

State comes from varbits (`VarbitID.MM_LAB_*`) — vessel readiness, the three mixer slots, and per-station potion contents — not from widget scraping. The only widget use is `InterfaceID.MM_OVERLAY` presence, which together with region `5521` and plane `0` defines "in the lab". `LabObject` hardcodes `WorldPoint`s because the lab is a fixed region.

## Conventions

- Tabs for indentation, Allman braces (RuneLite house style).
- **JUnit 4** (`org.junit.Test`, `org.junit.Assert.*`) — not JUnit 5.
- `Potion` enum **order is load-bearing**: `fromVarbit` maps the game's potion-type varbit as `ordinal + 1`. Reordering the enum silently breaks potion detection.
- `Potion` recipe codes follow the guided lever order (`ALA`, `MLL`, `ALL`), which intentionally differs from the underlying RuneLite item names (`AAL`, `LLM`, `LLA`). Don't "fix" this mismatch.
- Overlays are registered in `startUp` and must be removed in `shutDown`.

## Before Plugin Hub submission

Per the README: set the public author in `runelite-plugin.properties`, publish the repository, and add a Plugin Hub manifest with its URL and a full commit hash.
