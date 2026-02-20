# Clawventure schema reference

This document defines the file formats for all game data. Both the world builder (`/adventure:create`) and the game engine (`/adventure:play`) reference this schema.

A game consists of a directory containing several files and subdirectories. The directory structure provides progressive disclosure: the engine loads only the files it needs each turn, keeping context usage low regardless of world size.

---

## Directory structure

```
adventure-directory/
  world.json              Index: meta, trackables, location graph, NPC roster, quest list, item list, map, starting conditions
  locations/
    location_id.json      One file per location: description, details, events, local items and NPCs
  npcs/
    npc_id.json           One file per NPC: personality, dialogue prompt, knowledge, secrets, disposition
  quests/
    quest_id.json         One file per quest: stages, completion conditions, rewards
  items.json              All items in a single file
  state.json              Mutable player state (written every turn)
  memory.json             Narrative memory for session resume and NPC conversation continuity
  transcript.md           Growing prose transcript of the adventure (append-only)
```

File names use the same identifiers as the IDs in world.json. For example, if `location_index` contains a key `relay_square`, the location file is `locations/relay_square.json`.

---

## world.json

The index file. Loaded every turn. Contains only summary data -- enough to navigate the world, determine what files to load, and display status.

```json
{
  "meta": {
    "title": "string -- evocative title for the adventure",
    "theme": "string -- genre/theme",
    "tone": "string -- tonal description",
    "description": "string -- 2-3 paragraph world overview for the game engine's context",
    "created": "ISO 8601 date string"
  },

  "trackables": {
    "trackable_id": {
      "name": "string -- display name, e.g. 'Health', 'Sanity'",
      "description": "string -- what this quantity represents narratively",
      "min": 0,
      "max": 100,
      "start": 100,
      "on_zero": "death | madness | capture | collapse | custom",
      "on_zero_narrative": "string -- only if on_zero is 'custom'. Describes what happens narratively.",
      "show_in_status": true
    }
  },

  "location_index": {
    "location_id": {
      "name": "string -- display name",
      "connections": {
        "direction_or_label": "target_location_id"
      },
      "npcs": ["npc_id -- NPCs stationed here by default"],
      "has_conditions": false
    }
  },

  "npc_index": {
    "npc_id": {
      "name": "string -- full name",
      "title": "string -- role or epithet",
      "default_location": "location_id"
    }
  },

  "quest_index": {
    "quest_id": {
      "name": "string -- quest title",
      "given_by": "npc_id or 'world'"
    }
  },

  "item_index": {
    "item_id": {
      "name": "string -- display name",
      "type": "key | weapon | quest | consumable | treasure | tool | document"
    }
  },

  "map": {
    "type": "spatial | network | none",
    "display": "string -- ASCII art showing location connectivity or faction relationships. Use newlines for multiple lines."
  },

  "starting_conditions": {
    "location": "location_id",
    "inventory": ["item_id"],
    "active_quests": ["quest_id"],
    "flags": {},
    "prologue": "string -- 3-5 paragraphs of opening narrative. Second person. Set the scene, establish the player's situation, end with a hook."
  }
}
```

### Trackables

Trackables replace the hardcoded `health` field. They are setting-specific quantities that matter to the game. Examples by theme:

| Theme | Suggested trackables |
|-------|---------------------|
| D&D / high fantasy | health, mana |
| Cyberpunk | health, street_cred |
| Cthulhu / eldritch | health, sanity |
| Cold War / espionage | cover, suspicion |
| Post-apocalyptic / survival | health, hunger, radiation |
| Political / Foundation-style | influence, fleet_strength, loyalty |
| Prehistoric | health, hunger, warmth |

The `on_zero` field determines what happens when a trackable hits its minimum:
- `death`: standard death/failure with checkpoint reload offered
- `madness`: the character's mind breaks -- narrate the descent, offer checkpoint reload
- `capture`: the character is caught/exposed -- narrate capture, offer checkpoint reload
- `collapse`: a resource or faction is exhausted -- narrate the collapse and its consequences
- `custom`: use the `on_zero_narrative` string for a setting-specific outcome

Pick 1-3 trackables per setting. Every setting needs at least one.

### Map

The map field provides a visual aid for the player. Set the type based on the setting:

- `spatial`: physical exploration settings (dungeons, settlements, wilderness). Show an ASCII connectivity diagram of locations.
- `network`: abstract/political settings (court intrigue, espionage). Show faction or relationship diagrams.
- `none`: settings where neither applies. Set `map` to `null`.

During play, the engine overlays visited/unvisited status and hides locations the player hasn't discovered yet.

---

## Location files (locations/location_id.json)

One file per location. Loaded when the player enters that location.

```json
{
  "name": "string -- evocative proper name",
  "description": "string -- 3-5 sentences of atmospheric description shown on first entry. Literary, sensory, vivid.",
  "on_revisit": "string -- shorter 1-2 sentence description for return visits",
  "details": "string -- additional details revealed when the player examines the location more closely",
  "connections": {
    "direction_or_label": "target_location_id"
  },
  "items": ["item_id -- items present here at start"],
  "npcs": ["npc_id -- NPCs stationed here by default"],
  "conditions": {
    "locked": {
      "requires_item": "item_id or null",
      "requires_flag": "flag_name or null",
      "description": "string -- what blocks entry and what the player sees"
    }
  },
  "events": [
    {
      "trigger": "on_enter | on_examine | on_use_item | on_talk_npc",
      "trigger_detail": "item_id or npc_id if applicable, or null",
      "condition": {
        "has_flag": "flag_name or null",
        "has_item": "item_id or null",
        "quest_at_stage": { "quest_id": "stage_number or null" }
      },
      "once": true,
      "narrative": "string -- what happens, written in second person",
      "effects": {
        "set_flag": "flag_name or null",
        "add_item": "item_id or null",
        "remove_item": "item_id or null",
        "advance_quest": { "quest_id": "new_stage or null" },
        "move_npc": { "npc_id": "new_location_id or null" },
        "unlock_location": "location_id or null",
        "adjust_disposition": { "npc_id": "delta or null" },
        "adjust_trackable": { "trackable_id": "delta or null" }
      }
    }
  ]
}
```

### Event rules

- Events at the same location with the same `trigger` and `trigger_detail` must have mutually exclusive conditions, or be merged into a single event.
- `once: true` events are recorded in `state.json`'s `fired_events` array and never fire again.
- If multiple events match a trigger in the same turn, process them in array order, apply effects cumulatively, and narrate them as a connected sequence.
- Every `condition` field can be partially filled -- null fields are treated as "no requirement".

---

## NPC files (npcs/npc_id.json)

One file per NPC. Loaded when the player is at a location where that NPC is present, or when the NPC is referenced in dialogue.

```json
{
  "name": "string -- full name",
  "title": "string -- role or epithet, e.g. 'Innkeeper', 'The Blind Seer'",
  "description": "string -- physical appearance and first impression, 2-3 sentences",
  "personality": "string -- 3-5 key personality traits with brief explanations",
  "background": "string -- backstory and motivations, 2-4 sentences",
  "knowledge": [
    "string -- things this NPC knows and can share"
  ],
  "secrets": [
    "string -- things this NPC will not reveal easily (high disposition or quest progress required)"
  ],
  "dialogue_prompt": "string -- FULL CHARACTER PROMPT for generating in-character dialogue. This is the most important field. Include: speech patterns, vocabulary level, verbal tics, accent notes, characteristic phrases, emotional default, how they address strangers vs friends. Minimum 4 sentences.",
  "default_location": "location_id",
  "inventory": ["item_id -- items they can give, trade, or sell"],
  "quest_roles": {
    "quest_id": "string -- brief description of their role in this quest"
  },
  "disposition_start": 0,
  "disposition_modifiers": {
    "positive": ["topics or actions that raise disposition"],
    "negative": ["topics or actions that lower disposition"]
  }
}
```

### Dialogue prompt guidance

The `dialogue_prompt` is the single most important field in the game. It must contain enough character information for the engine to generate consistent, distinctive dialogue without reading any other field. Include:

- Speech patterns and sentence structure (clipped and terse? flowing and ornate? peppered with jargon?)
- Vocabulary level and register
- Verbal tics and characteristic phrases (recurring words, habitual greetings, pet expressions)
- How they address strangers vs people they trust
- Emotional default and how they express different emotions
- What topics make them animated or shut down

---

## Quest files (quests/quest_id.json)

One file per quest. Loaded when checking active quest status or when a quest-relevant action occurs.

```json
{
  "name": "string -- quest title shown to player",
  "description": "string -- player-facing quest summary, 2-3 sentences",
  "given_by": "npc_id or 'world'",
  "stages": [
    {
      "stage": 0,
      "description": "string -- internal description of this stage",
      "objective": "string -- player-visible objective text",
      "hints": [
        "string -- progressive hints if player seems stuck, from vague to specific"
      ],
      "completion": {
        "type": "arrive_at | talk_to | use_item | has_item | flag_set | give_item_to_npc",
        "target": "relevant id",
        "detail": "additional context if needed",
        "next_stage": 1
      }
    }
  ],
  "completion_stage": "number -- the stage number that means the quest is complete",
  "rewards": {
    "items": ["item_id"],
    "flags": ["flag_name"],
    "unlocks": ["location_id or quest_id"],
    "narrative": "string -- what happens narratively when quest completes"
  }
}
```

---

## items.json

All items in a single file. Loaded when items are relevant to the current turn (pickup, use, examination, inventory check).

```json
{
  "item_id": {
    "name": "string -- display name",
    "description": "string -- 1-2 sentences when examined",
    "type": "key | weapon | quest | consumable | treasure | tool | document",
    "portable": true,
    "usable_at": {
      "location_id or npc_id": "string -- what happens when used here/on this NPC"
    },
    "hidden": false,
    "revealed_by": "flag_name or null -- if hidden, what flag reveals it"
  }
}
```

### Item visibility

Items with `hidden: true` are not visible to the player until the flag named in `revealed_by` is set. Every `revealed_by` flag must have a corresponding event or NPC interaction convention that sets it (see flag conventions below).

---

## state.json

Mutable player state. Read at the start of each turn, written at the end. This file contains only mechanical state -- no narrative content.

```json
{
  "setting_directory": ".",
  "player": {
    "name": "",
    "description": "",
    "location": "starting_location_id",
    "inventory": ["item_id"]
  },
  "trackables": {
    "trackable_id": "current_value (number)"
  },
  "quests": {
    "quest_id": {
      "status": "available | active | completed",
      "current_stage": 0,
      "log": ["string -- brief log entries"]
    }
  },
  "npc_state": {
    "npc_id": {
      "disposition": 0,
      "met": false,
      "conversation_count": 0,
      "gave_items": [],
      "location_override": null
    }
  },
  "world_flags": {},
  "visited_locations": [],
  "turn": 0,
  "history": [
    {
      "turn": 1,
      "action": "string -- brief description",
      "location": "location_id",
      "summary": "string -- 1-sentence narrative summary"
    }
  ],
  "fired_events": []
}
```

### Flag conventions

The engine sets these flags automatically:
- `[npc_id]_talked`: set when the player talks to an NPC for the first time
- `asked_[npc_id]_[topic]`: set when an NPC shares knowledge on a specific topic
- `taken_[item_id]`: set when the player picks up an item from a location

These conventions ensure that `revealed_by` fields in items and `condition.has_flag` fields in events work reliably without requiring explicit events for every interaction.

---

## memory.json

Narrative memory for session persistence and NPC conversational continuity. Written only when something significant happens -- not every turn. Never modified during checkpoint rollback (the player remembers even if the game rewinds).

```json
{
  "version": 1,

  "sessions": [
    {
      "session_id": 1,
      "started_at_turn": 0,
      "ended_at_turn": 15,
      "started_at": "ISO 8601",
      "ended_at": "ISO 8601 or null if still active",
      "summary": "string -- 2-3 sentence summary of what happened this session. Written at session end."
    }
  ],

  "narrative_log": [
    {
      "turn": 5,
      "session_id": 1,
      "type": "discovery | npc_encounter | quest_advance | player_choice | world_change | trackable_threshold",
      "location": "location_id",
      "summary": "string -- 1-2 sentence narrative summary in past tense.",
      "significance": "high | medium"
    }
  ],

  "npc_conversations": {
    "npc_id": [
      {
        "turn": 8,
        "session_id": 1,
        "disposition_before": 5,
        "disposition_after": 10,
        "topics_discussed": ["short string identifiers mapping to the NPC's knowledge/secrets"],
        "player_approach": "string -- brief characterisation: friendly, aggressive, inquisitive, traded for info, etc.",
        "npc_revealed": ["string -- knowledge items the NPC shared"],
        "npc_withheld": ["string -- topics the NPC deflected or refused to share"],
        "emotional_tone": "string -- how the conversation went emotionally",
        "memorable_line": "string -- one standout NPC line from the conversation"
      }
    ]
  },

  "checkpoints": [
    {
      "turn": 10,
      "reason": "auto | pre_danger | manual",
      "state_snapshot": { "...full copy of state.json at this turn..." }
    }
  ],

  "player_choices": [
    {
      "turn": 15,
      "session_id": 1,
      "choice": "string -- what the player chose",
      "consequence_note": "string -- what happened as a result"
    }
  ]
}
```

### Logging rules

**Always log (significance: high):**
- First meeting with any NPC
- Quest stage advances
- Locations unlocked
- One-time events fired
- Trackable hitting a threshold (below 25% of max)
- Player death/reload

**Log if noteworthy (significance: medium):**
- NPC conversation where disposition changed by more than 10 points
- NPC revealed a secret
- Quest-critical item obtained
- Meaningful custom action via "Other"

**Do not log:**
- Routine movement between locations
- Checking inventory or map
- Return visits with no new developments
- NPC conversations that are purely social with no information exchange

### Checkpoint strategy

- Automatic checkpoint every 5 turns
- Keep only the most recent 3 checkpoints (discard oldest when adding a new one)
- On any trackable reaching its `on_zero` threshold, load the nearest checkpoint

### NPC conversation records

Write a record after every non-trivial conversation (at least one exchange of information, one disposition change, or one item exchange). Use `topics_discussed` identifiers that map to the NPC's `knowledge` and `secrets` arrays from their NPC file. The `memorable_line` is one sentence maximum -- the most characterful or plot-relevant line the NPC said.

---

## transcript.md

A growing markdown file recording the prose narrative. Append-only -- never read by the engine during play, never modified or truncated. Exists purely as an export for the player.

### Format

```markdown
# [Adventure Title]

*[Theme] -- [Tone]*

## Prologue

[Prologue text from starting_conditions]

---

## Turn 1

[Scene description prose]

> *[Player's chosen action]*

[Action outcome prose: NPC dialogue, item discovery, event narration]

---

## Turn 2

[...]
```

Written after each turn's narration and action resolution. On game completion, append the epilogue and final stats.

---

## Backwards compatibility

If the engine finds a monolithic `setting.json` instead of a `world.json`, it should load the entire file and operate as before. The progressive-disclosure directory structure is the preferred format for new settings but the engine must handle both.
