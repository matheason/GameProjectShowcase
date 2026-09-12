# Project [Codename] — Technical Showcase

## 🛡️ Repository Access & Privacy Notice
The main source code and proprietary assets for this indie game project are maintained in a **private repository** for intellectual property (IP) protection prior to release. This public showcase repository highlights system architecture, component design, and software engineering practices used during development.

---

## 📖 Overview & Mechanics
- **Engine & Language**: Unity (6.x) | C#
- **Genre**: Action-Adventure / Puzzle-Platformer
- **Core Concept**: Environmental traversal and puzzle-solving built around dynamic material properties (Gel, Mud, Rock) and wave-sensing mechanics.

---

## 🛠️ System Architecture & Engineering

### 1. Component-Based & Decoupled Architecture
To ensure scalability and maintainability, entity logic is decoupled into single-responsibility C# modules rather than monolithic manager scripts.

![Unity Editor Architecture](./Media/unity_editor_inspector.png)

* **Single-Responsibility Modules**: Component separation allows independent handling of core systems (e.g., `Player Movement`, `Body Mass Scale`, `Puddle Manager`, `Beat Attack Router`).
* **Clean Workspace Hierarchy**: Organized script directories separating `Combat`, `Core`, `DataScripts`, `Enemies`, and `AudioScripts` for modular codebase maintenance.

---

### 💻 Code Architecture Sample (Event-Driven Communication)

Below is an abstract snippet demonstrating how environmental triggers broadcast data to game entities via dynamic C# events without creating hard dependencies:

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// Central event hub for systems that should not directly reference each other.
///
/// Example: a UI button can request a save without knowing where SaveManager lives.
/// Call the public methods instead of invoking the Action fields directly; the
/// methods are null-safe and make the intended game message clear.
/// </summary>
// HOW THIS WORKS (read 02_Architecture.md for the full picture):
//   * Each "channel" is a static Action field. Listeners do `Channel += MyMethod;` in
//     OnEnable and `Channel -= MyMethod;` in OnDisable to start/stop receiving it.
//   * Each channel has a matching "fire" method (e.g. PlayerStatsChanged) that calls
//     Invoke with `?.` so firing is safe even when nobody is listening.
// Always FIRE through the methods; only SUBSCRIBE to the Action fields.
public static class GameEvents
{
    /// <summary>
    /// Player stat and stat-refresh messages used by HUD and status panels.
    /// </summary>
    public static class Player
    {
        public static Action OnPlayerStatsChanged;
        public static Action OnRequestStatsRefresh;

        // Fired exactly once per death by PlayerLifecycle (the single HP-zero detector).
        // Listeners must not detect death themselves by polling IsDead off the stats
        // event — that pattern fires per-consumer and re-triggers on every stat change.
        public static Action OnPlayerDied;
        public static Action OnPlayerRespawned;

        public static void PlayerStatsChanged()
        {
            // The "?." means: only invoke if at least one listener is subscribed.
            OnPlayerStatsChanged?.Invoke();
        }

        public static void RequestStatsRefresh()
        {
            OnRequestStatsRefresh?.Invoke();
        }

        public static void PlayerDied()
        {
            OnPlayerDied?.Invoke();
        }

        public static void PlayerRespawned()
        {
            OnPlayerRespawned?.Invoke();
        }
    }

    /// <summary>
    /// UI messages such as button clicks, anchor hover/touch, dialogue text, and canvas lock state.
    /// </summary>
    public static class UI
    {
        public static Action OnButtonClicked;
        public static Action<string> OnTouchedAnchor;
        public static Action<string, List<DialogueLine>> OnRequestDialogueTextRefresh;
        public static Action<bool> OnUICanvasPermissionChanged;

        public static void ButtonClicked()
        {
            OnButtonClicked?.Invoke();
        }

        public static void TouchedAnchor(string id)
        {
            OnTouchedAnchor?.Invoke(id);
        }

        public static void RequestDialogueTextRefresh(string storyID, List<DialogueLine> dialogueLines)
        {
            OnRequestDialogueTextRefresh?.Invoke(storyID, dialogueLines);
        }

        public static void UICanvasPermissionChanged(bool permission)
        {
            OnUICanvasPermissionChanged?.Invoke(permission);
        }
    }

    /// <summary>
    /// World-progression messages, such as unlocking or moving to a new level.
    /// </summary>
    public static class World
    {
        public static Action<WorldLevels> OnWorldStateChanged;

        public static void WorldStateChanged(WorldLevels levelUnlocked)
        {
            OnWorldStateChanged?.Invoke(levelUnlocked);
        }
    }

    /// <summary>
    /// High-level commands requested by menus, buttons, and gameplay triggers.
    /// GlobalActionsManager is the main listener that performs the work.
    /// </summary>
    public static class GlobalActions
    {
        public static Action<GlobalActionTypes, string> OnGlobalActionApplied;

        // info carries the command's argument (a panel id, scene name, etc.); it defaults
        // to "" for commands that need no argument, like QuitApp.
        public static void GlobalActionApplied(GlobalActionTypes functionType, string info = "")
        {
            OnGlobalActionApplied?.Invoke(functionType, info);
        }
    }

    /// <summary>
    /// Trigger messages for modifying player stats from world objects.
    /// </summary>
    public static class Trigger
    {
        public static Action<StatsType, TriggerSignal, TriggerName, float> OnTriggerChangedStats;

        public static void TriggerChangedStats(
            StatsType statsType,
            TriggerSignal signal,
            TriggerName source,
            float value
        )
        {
            OnTriggerChangedStats?.Invoke(statsType, signal, source, value);
        }
    }

    /// <summary>
    /// Global-variable change messages fired by GlobalVariables whenever a value is set.
    /// </summary>
    public static class Variables
    {
        public static Action<string, float> OnGlobalVariableChanged;

        public static void GlobalVariableChanged(string key, float value)
        {
            OnGlobalVariableChanged?.Invoke(key, value);
        }
    }

    /// <summary>
    /// Long-term progress messages: plot flags, inventory, achievements, and map
    /// discovery. Fired by the stores in Assets/Scripts/SaveScripts, which are the
    /// same things SaveManager persists — so anything that listens here stays correct
    /// after a load too, because loading replays every value through these events.
    /// </summary>
    public static class Progress
    {
        public static Action<string, bool> OnFlagChanged;
        public static Action<string, int> OnInventoryChanged;
        public static Action<string, string> OnEquipmentChanged;
        public static Action<string> OnAchievementUnlocked;
        public static Action<string, int> OnAchievementProgressChanged;
        public static Action<string, float> OnRegionDiscoveryChanged;

        public static void FlagChanged(string flag, bool raised)
        {
            OnFlagChanged?.Invoke(flag, raised);
        }

        public static void InventoryChanged(string itemId, int newCount)
        {
            OnInventoryChanged?.Invoke(itemId, newCount);
        }

        public static void EquipmentChanged(string slot, string itemId)
        {
            OnEquipmentChanged?.Invoke(slot, itemId);
        }

        public static void AchievementUnlocked(string achievementId)
        {
            OnAchievementUnlocked?.Invoke(achievementId);
        }

        public static void AchievementProgressChanged(string achievementId, int value)
        {
            OnAchievementProgressChanged?.Invoke(achievementId, value);
        }

        /// <summary>percent01 is 0..1, not 0..100 — multiply where it is displayed.</summary>
        public static void RegionDiscoveryChanged(string regionId, float percent01)
        {
            OnRegionDiscoveryChanged?.Invoke(regionId, percent01);
        }
    }

    /// <summary>
    /// Localization messages used when the active language changes.
    /// </summary>
    public static class Localization
    {
        public static Action OnLanguageChanged;

        public static void LanguageChanged()
        {
            OnLanguageChanged?.Invoke();
        }
    }

    /// <summary>
    /// Save/load lifecycle messages. SaveManager listens to these and performs disk IO.
    /// </summary>
    public static class System
    {
        public static Action OnRequestSave;
        public static Action OnRequestLoad;
        public static Action OnHasSavedSuccessfully;
        /// <summary>Start a new game in the current flow (SaveManager.StartNewGame handles it).</summary>
        public static Action OnRequestNewGame;

        public static void RequestNewGame()
        {
            OnRequestNewGame?.Invoke();
        }

        public static void RequestSave()
        {
            OnRequestSave?.Invoke();
        }

        public static void RequestLoad()
        {
            OnRequestLoad?.Invoke();
        }

        public static void HasSavedSuccessfully()
        {
            OnHasSavedSuccessfully?.Invoke();
        }
    }
}

