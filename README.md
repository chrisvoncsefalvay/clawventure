# Clawventure -- interactive fiction engine for Claude Code

                                                                           
                                                                           
    ▄█████ ▄▄     ▄▄▄  ▄▄   ▄▄ ▄▄ ▄▄ ▄▄▄▄▄ ▄▄  ▄▄ ▄▄▄▄▄▄ ▄▄ ▄▄ ▄▄▄▄  ▄▄▄▄▄ 
    ██     ██    ██▀██ ██ ▄ ██ ██▄██ ██▄▄  ███▄██   ██   ██ ██ ██▄█▄ ██▄▄  
    ▀█████ ██▄▄▄ ██▀██  ▀█▀█▀   ▀█▀  ██▄▄▄ ██ ▀██   ██   ▀███▀ ██ ██ ██▄▄▄ 
                                                                           

_A completely idiotic idea by [Chris von Csefalvay](https://chrisvoncsefalvay.com)._

Clawventure is a plugin that turns Claude Code into a choose-your-adventure game engine. Create richly detailed worlds with interconnected locations, NPCs with distinct personalities, items and multi-stage quests, then play through them with AI-driven dialogue and dynamic narration.

The key trick: every NPC carries a full character prompt, so Claude generates unique, contextual dialogue rather than following scripted trees. The setting files provide constraints and world data. Claude provides the narrative intelligence.

## Commands

| Command | Purpose |
|---------|---------|
| `/adventure:create` | Build a new adventure setting interactively |
| `/adventure:play` | Play an existing adventure |

### `/adventure:create`

Walks you through world creation via interactive prompts:

1. Choose theme, tone, scale and flavour elements
2. Select setting-specific trackable quantities (health, sanity, influence, etc.)
3. Generates a complete world directory (locations, NPCs, items, quests)
4. Writes initial state, memory and transcript files

### `/adventure:play`

Runs the game engine:

1. Loads setting files from the adventure directory
2. Character creation on first run
3. Continuous game loop: scene narration, choices via interactive prompts, NPC dialogue, state persistence
4. Saves automatically every turn -- quit and resume any time with a narrative recap
5. Generates a prose transcript you can keep after the game

## Installation

### Quick install (copy commands)

Copy the `commands/` directory contents into your Claude Code commands directory:

```bash
# Linux / macOS
mkdir -p ~/.claude/commands/adventure
cp commands/*.md ~/.claude/commands/adventure/

# Windows (PowerShell)
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\commands\adventure"
Copy-Item commands\*.md "$env:USERPROFILE\.claude\commands\adventure\"
```

Restart Claude Code. The commands `/adventure:create` and `/adventure:play` will be available.

### Plugin install (if using the plugin system)

If your Claude Code setup supports plugin installation, clone this repo and register it:

```bash
git clone <repo-url> ~/.claude/plugins/cache/adventure-plugin/adventure/2.0.0
```

Then add an entry to `~/.claude/plugins/installed_plugins.json` under the `plugins` key:

```json
"adventure@adventure-plugin": [
  {
    "scope": "user",
    "installPath": "<full-path-to-cloned-directory>",
    "version": "2.0.0",
    "installedAt": "<ISO-timestamp>",
    "lastUpdated": "<ISO-timestamp>"
  }
]
```

## How it works

### Architecture

The system has two halves:

**World builder** (`/adventure:create`) generates a directory of files:

```
my-adventure/
  world.json          Index: meta, trackables, location graph, NPC roster, map
  locations/*.json    One file per location with full descriptions and events
  npcs/*.json         One file per NPC with dialogue prompts and personality
  quests/*.json       One file per quest with stages and conditions
  items.json          All items
  state.json          Player state (updated each turn)
  memory.json         Narrative memory for resume and NPC continuity
  transcript.md       Growing prose transcript of the adventure
```

This progressive-disclosure structure means the engine loads only what it needs each turn -- the current location, present NPCs, active quests -- rather than the entire world. A sprawling 20-location setting uses the same per-turn context as a compact 6-location one.

**Game engine** (`/adventure:play`) reads the setting files and maintains the mutable files:

- `state.json` tracks player location, inventory, quest progress, NPC dispositions, world flags, and turn-by-turn history
- `memory.json` tracks narrative events, NPC conversation summaries, session boundaries, and rolling checkpoints for save/reload
- `transcript.md` records the full prose narrative of the adventure

### Trackable quantities

Each setting defines what quantities matter to the game. Instead of a hardcoded health bar, trackables are chosen to fit the theme:

- A Cthulhu setting tracks health and sanity
- A Cold War spy thriller tracks cover integrity and suspicion
- A survival setting tracks health, hunger and radiation
- A political intrigue setting tracks influence and loyalty

Trackables are displayed in the status line and change through events, NPC interactions and environmental effects. When any trackable hits zero, the setting defines what happens: death, madness, capture, or something custom.

### NPC dialogue system

Each NPC has a `dialogue_prompt` field -- a full character description including speech patterns, verbal tics, vocabulary, emotional defaults and how they address people. When the player talks to an NPC, Claude adopts that persona, considering:

- The NPC's current disposition toward the player (-100 hostile to +100 devoted)
- Whether they have met before and what was discussed (tracked in memory.json)
- What the NPC knows and is willing to share at the current trust level
- Active quest states relevant to the NPC
- Topics the NPC likes or dislikes

No dialogue trees. Every conversation is dynamically generated, and NPCs remember prior interactions.

### State persistence

State saves after every turn. Players can quit any time and resume with `/adventure:play`. On resume, the engine builds a narrative "Previously..." recap from the memory file -- not just "you were at location X" but a prose summary of what happened, who you spoke to and what matters.

Rolling checkpoints (every 5 turns, keeping the last 3) enable save-point reloading on failure.

### Map and navigation

Settings that involve physical exploration include an ASCII map in `world.json`. Players can view the map at any time, with visited and unvisited locations clearly marked. Quest objectives include location hints so players always know where to go.

### Transcript

The game writes a prose transcript to `transcript.md` as you play -- scene descriptions, dialogue, quest events. When the adventure ends, you have a readable story of your playthrough.

## File reference

```
.claude-plugin/
  plugin.json           Plugin manifest
commands/
  create.md             World builder command
  play.md               Game engine command
  schema.md             Complete schema reference for all game files
examples/
  setting.json          Example setting (legacy monolithic format)
  state.json            Matching initial state
README.md               This file
```

## Creating settings by hand

You can write setting files manually or with other tools. The complete schema is documented in `commands/schema.md`. The minimum viable setting needs:

- A `world.json` with at least 2 connected locations, 1 NPC, 1 trackable and starting conditions
- Location files in `locations/` for each location in the index
- At least 1 NPC file in `npcs/` with a dialogue prompt
- An `items.json` (can be empty)
- At least 1 quest file in `quests/` (or the game has no objective)

The engine also supports the legacy monolithic `setting.json` format (as shown in `examples/`).

## Backwards compatibility

v2.0.0 introduces the directory-based setting format. The engine still supports the v1.0.0 monolithic `setting.json` format -- if it finds a `setting.json` instead of a `world.json`, it loads the entire file and plays as before. New settings created with `/adventure:create` use the directory format.

## Licence

Oh, for the love of fuck. Just do whatever.


---

_Made with ❤️ by 👨‍🔬 Chris and 🐕‍🦺 Oliver in the ⛰️ Mile High City._
