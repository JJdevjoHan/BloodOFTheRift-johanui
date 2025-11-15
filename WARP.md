# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project overview

This is a small Java Swing RPG-style UI project. The main entry point is `ui.HomeFrame`, which presents a character creation screen for the game "Blood Of The Rift". After character creation, control flows into world UI frames such as `ui.GrassyPlains` (and other world frames like `DesertWorld`, `LavaWorld`, etc.), where exploration and battles occur.

Eclipse metadata files (`.project`, `.classpath`) are present; the project is designed to be opened and run from Eclipse, but it can also be compiled and run from the command line.

## Build, run, and test

### Running from Eclipse

- Import the project into Eclipse as an existing Java project (Eclipse should recognize `.project` / `.classpath`).
- Use `ui.HomeFrame`'s `main` method as the primary run configuration. `ui.GrassyPlains` also has a `main` method useful for direct testing of that world.

### Compiling and running from the command line

From the project root (`AGAINUI`):

- Compile all sources into an `out` directory:
  - On a typical JDK setup:
    ```sh path=null start=null
    mkdir -p out
    javac -d out src/org/eclipse/wb/swing/FocusTraversalOnArray.java src/ui/*.java
    ```
- Run the main UI (character creation and game start):
  ```sh path=null start=null
  java -cp out ui.HomeFrame
  ```
- Alternatively, run directly into the Grassy Plains world (useful for debugging combat and exploration):
  ```sh path=null start=null
  java -cp out ui.GrassyPlains
  ```

### Linting and formatting

- There is no configured linter or formatter in this repository (no Maven/Gradle/Checkstyle configuration is present). If you introduce one (e.g., Checkstyle or Spotless), document the command here.

### Tests

- There are currently no automated test sources or test frameworks configured (no `test` directory, JUnit dependencies, or build tool configuration). There is therefore no standard test command, and no way to run a single test from the CLI at this time.

## High-level architecture

### UI entry and navigation

- `ui.HomeFrame` is the primary entry point.
  - Presents the game title and a class-selection table with images for `Paladin`, `Mage`, and `Warrior`.
  - Collects the player name and validates that it is exactly 5 characters.
  - Lets the player choose a class via `JComboBox` and confirms the choice with descriptive dialogs.
  - On confirmation, it constructs a `Loading` frame and then transitions into `GrassyPlains` (the first explorable world).
- Each world (e.g., `GrassyPlains`, `DesertWorld`, `LavaWorld`, `SnowyIsland`, `FinalWorld`) is implemented as a separate `JFrame`, responsible for:
  - Rendering the local UI layout and theming.
  - Managing world-specific exploration logic.
  - Spawning and managing encounters with local mobs/bosses.

### Core character and enemy model

- `ui.Character` (top-level class) defines the core player model:
  - Public fields for `name`, `className`, `hp`, `mana`, `maxHp`, and `maxMana`.
  - Support for temporary damage buffs (`temporaryDamageBuff`, `damageBuffDuration`) with helper methods to apply and decrement buffs.
  - Abstract methods:
    - `useSkill(int choice, World1Mob target)` — executes a skill and returns a descriptive message.
    - `displaySkillsSwing(JTextArea battleLog)` — writes a description of the class skills to the battle log.
- Concrete subclasses implement class-specific stats and skills:
  - `ui.Warrior` — high-HP melee class with skills like `Stone Slash`, `Flame Strike`, and `Earthquake Blade`.
  - `ui.Mage` — glass-cannon caster with high mana and skills like `Frost Bolt`, `Rune Burst`, and `Lightstorm`.
  - `ui.Paladin` — tank/support with `Shield Bash`, `Radiant Guard`, and `Holy Renewal` (healing).
- `ui.World1Mob` abstracts enemy behavior:
  - Fields: `name`, `hp`, `maxHp`, `damage`.
  - Methods for basic attacks and a `specialSkill(Character)` method for stronger moves.
  - Concrete mob types are defined as inner classes (`Slime`, `Bull`, `Wolf`, `Minotaur`) with different HP/damage profiles, and `Minotaur` overrides `specialSkill` for a boss-style attack.

### Battle flow

- `ui.Battle` encapsulates turn-based combat between a `Character` and a `World1Mob`:
  - Holds references to the `player`, `mob`, the shared `JTextArea battleLog`, and `JLabel` status labels.
  - `start()` runs a simple loop where the player and mob alternate turns until either is defeated.
  - `playerTurn()` uses `JOptionPane` dialogs to ask the player for an action (Attack, Spell, Flee) and applies damage/mana changes.
  - `mobTurn()` applies mob damage to the player and updates the UI.
  - `updateStatus()` and `appendLog()` keep the Swing UI labels and log in sync with the game state.
- In `GrassyPlains` and other worlds, skill buttons (`skill1Btn`, `skill2Btn`, `skill3Btn`) are wired to call `player.useSkill(...)` directly, bypassing `Battle`'s dialog loop for a more button-driven experience.

### World exploration and progression (example: `GrassyPlains`)

- `ui.GrassyPlains` manages both exploration and combat in a single `JFrame`:
  - Maintains `JLabel` status (`playerLabel`, `mobLabel`), a `JTextArea battleLog`, and directional buttons (`North`, `East`, `South`, `West`) plus an `Inspect Class` button.
  - Uses a `Character player` (chosen based on the class from `HomeFrame`) and a `World1Mob currentMob` to represent the active encounter.
  - Tracks world state: steps taken, which directions are cleared, mob assignments per direction, a reserved direction for the `Minotaur` mini-boss, and whether a chest has been found.
  - `explore(direction)` handles movement:
    - Before the chest: increments `stepsTaken`, logs movement text, and eventually triggers a chest find.
    - After the chest: assigns mobs to directions, reserves a direction for the `Minotaur`, and spawns the appropriate enemy when moving in a direction.
  - `doSkill(int choice)` is invoked by skill buttons:
    - Calls `player.useSkill(choice, currentMob)` and logs the result.
    - If the mob dies, it updates counters, may trigger the `Minotaur` and portal to `DesertWorld`, and opens a reward chest via `openRewardChest()`.
    - If the player dies, it disables movement/buttons and shows a Game Over dialog.
  - `openRewardChest()` grants one of several randomized rewards (HP/mana max boosts, temporary damage buff, full heal, full mana) and updates UI.

### Duplication and legacy code in `GrassyPlains`

- `ui.GrassyPlains` contains its own nested definitions of `Character`, `Warrior`, `Mage`, `Paladin`, `World1Mob`, and mobs (`Slime`, `Bull`, `Wolf`, `Minotaur`).
  - These inner classes predate or duplicate the logic in the top-level `ui.Character` and `ui.World1Mob` and their subclasses.
  - When refactoring combat or stats, be aware that there are two separate implementations of the character/enemy model: one shared (`ui.Character`, `ui.Mage`, `ui.Warrior`, `ui.Paladin`, `ui.World1Mob`) and one local to `GrassyPlains`.
  - Any unification or cleanup work should decide which model is canonical and migrate usage accordingly.

### External helper

- `org.eclipse.wb.swing.FocusTraversalOnArray` is a utility class generated/used by Eclipse WindowBuilder for focus traversal. It is standalone, under the Eclipse Public License, and not tightly coupled to the game logic. It can be reused by Swing forms as needed but does not affect gameplay.

