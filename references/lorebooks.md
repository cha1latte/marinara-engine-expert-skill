# Lorebooks

Lorebooks are **keyword-triggered knowledge injection** for characters and chats. Each lorebook contains entries; each entry has trigger keywords and content. When recent chat messages contain a keyword, the entry's content is injected into the prompt.

This is the right tool for **large, structured, but stable** reference knowledge that would waste tokens if always-on.

**Source of truth:** `packages/shared/src/schemas/lorebook.schema.ts`.

## Concepts

A **lorebook** is a container with:
- A name, description, category (`world` / `character` / `npc` / `spellbook` / `uncategorized`)
- A `tokenBudget` — maximum tokens of entries that can be injected per turn (default 2048)
- A `scanDepth` — how many recent messages to scan for keywords (default 2)
- `recursiveScanning` — if true, activated entries' content is itself scanned for more triggers
- `maxRecursionDepth` — cap on recursion (default 3, max 10)
- Scope: global, per-character (`characterId`), or per-chat (`chatId`)

An **entry** is the actual knowledge chunk with:
- `name` — human-readable label
- `content` — what gets injected into the prompt
- `keys` — primary trigger words/regexes
- `secondaryKeys` — additional triggers
- Activation rules (constant, selective, probability, conditions, schedule)
- Timing (sticky, cooldown, delay, ephemeral)
- Grouping and priority (group, groupWeight, order)
- Position (before or after character description; depth within history)
- Matching options (whole-word, case-sensitive, regex)

## Entry Fields (Full Reference)

From `createLorebookEntrySchema`:

| Field | Type | Default | What it does |
|---|---|---|---|
| `name` | string | required | Display name in the UI. Not sent to model. |
| `content` | string | `""` | The text injected when triggered. |
| `keys` | string[] | `[]` | Primary trigger keywords. Matching any triggers the entry. |
| `secondaryKeys` | string[] | `[]` | Additional triggers, used with `selective` + `selectiveLogic`. |
| `enabled` | bool | `true` | Master on/off. |
| `constant` | bool | `false` | **If true, ALWAYS injected** (no keyword needed). Use sparingly. |
| `selective` | bool | `false` | If true, combines primary and secondary keys via `selectiveLogic`. |
| `selectiveLogic` | `"and" | "or" | "not"` | `"and"` | How primary and secondary keys combine when `selective`. |
| `probability` | number | `null` | 0-1 chance of firing even when triggered. |
| `scanDepth` | number | `null` | Overrides the lorebook's scan depth for this entry. |
| `matchWholeWords` | bool | `false` | If true, `king` won't match `kingdom`. |
| `caseSensitive` | bool | `false` | If true, `King` ≠ `king`. |
| `useRegex` | bool | `false` | Treat `keys` as regex patterns instead of plain strings. |
| `position` | 0 or 1 | `0` | 0 = before character description, 1 = after. |
| `depth` | number | `4` | How many messages deep to inject (for `@depth` positioning). |
| `order` | number | `100` | Insertion order within same position/group. Lower = earlier. |
| `role` | `"system" | "user" | "assistant"` | `"system"` | What role to attribute the injection to. |
| `sticky` | number | `null` | Stay active for N messages after last trigger. |
| `cooldown` | number | `null` | Minimum messages between activations. |
| `delay` | number | `null` | Wait N messages before first activation in a chat. |
| `ephemeral` | number | `null` | Only active for N total activations per chat. |
| `group` | string | `""` | Group name. Entries in same group compete by weighted lottery. |
| `groupWeight` | number | `null` | Weight for intra-group competition. |
| `preventRecursion` | bool | `false` | Don't trigger further entries from this one's content. |
| `locked` | bool | `false` | Protect from agent edits (e.g., Lorebook Keeper). |
| `tag` | string | `""` | Freeform category tag. |
| `relationships` | object | `{}` | Cross-entry references (for graph-style lore). |
| `dynamicState` | object | `{}` | Per-chat mutable state. |
| `activationConditions` | array | `[]` | Game-state gates (`field`, `operator`, `value`). |
| `schedule` | object | `null` | Time/date/location gating for game mode. |

## Common Entry Patterns

### Always-on entry (use sparingly)
```json
{
  "name": "Setting Overview",
  "constant": true,
  "content": "The story is set in a cyberpunk Seoul, 2087. Megacorps rule; AI is illegal."
}
```
Constant entries burn tokens every turn. Keep them short.

### Keyword-triggered entry (the default case)
```json
{
  "name": "Dr. Kim",
  "keys": ["Dr. Kim", "doctor Kim", "Kim Ji-ho"],
  "content": "Dr. Kim Ji-ho, 42, black-market cyberneticist. Works out of a basement clinic in Gangnam."
}
```
Injected only when any of the keys match recent messages.

### Regex entry for flexible matching
```json
{
  "name": "Dragon Lore",
  "keys": ["dragon\\w*", "\\bwyrm\\b"],
  "useRegex": true,
  "matchWholeWords": false,
  "content": "Dragons in this world are extinct except for five known survivors..."
}
```

### Selective entry (both primary AND secondary must match)
```json
{
  "name": "Combat with Kim",
  "keys": ["Dr. Kim"],
  "secondaryKeys": ["fight", "attack", "combat", "gun"],
  "selective": true,
  "selectiveLogic": "and",
  "content": "In combat, Dr. Kim prefers non-lethal neural disruptors; he won't kill unless cornered."
}
```

### Grouped entries (one-of-N weighted lottery)
```json
[
  { "name": "Random Weather: Rain", "group": "weather", "groupWeight": 3, "constant": true, "content": "It's raining." },
  { "name": "Random Weather: Fog", "group": "weather", "groupWeight": 1, "constant": true, "content": "Heavy fog." },
  { "name": "Random Weather: Clear", "group": "weather", "groupWeight": 6, "constant": true, "content": "Clear skies." }
]
```
Of the group members whose triggers fire, only one is chosen, weighted by `groupWeight`.

### Sticky entry (stays around after trigger)
```json
{
  "name": "In the cave",
  "keys": ["enter the cave", "cave mouth"],
  "content": "The party is inside the Echoing Cave. It's dark and damp. Every sound echoes.",
  "sticky": 10
}
```
Once triggered, stays active for 10 more messages even without the keyword repeating.

### Ephemeral entry (limited uses)
```json
{
  "name": "First meeting surprise",
  "keys": ["Marcus"],
  "content": "Marcus is surprised to see the player — he thought they were dead.",
  "ephemeral": 1
}
```
Fires at most once per chat, then disables itself.

## Scope: Global vs. Per-Character vs. Per-Chat

Lorebooks have three scopes, controlled by `characterId` and `chatId`:
- `characterId = null, chatId = null` — **global**. Attached to all chats where enabled in prompt settings.
- `characterId = "xyz", chatId = null` — **character-scoped**. Only active in chats that include this character.
- `chatId = "abc"` — **chat-scoped**. Only active in one specific chat.

**When to use which:**
- World lore, shared universes → global.
- Character's personal memories, backstory depth → character-scoped.
- Current scene state, one-session-specific plot flags → chat-scoped.

## Token Budget Management

The lorebook's `tokenBudget` caps total injected content per turn. If more entries match than fit, higher-priority entries win (ordered by `order` first, then entry length / internal logic).

**Practical sizing:**
- For casual characters: 500–1000 tokens budget.
- For rich worldbuilding: 2000–4000 tokens budget.
- Past ~4000 is usually a sign you should split into multiple lorebooks or use semantic memory (RAG over messages).

## Recursive Scanning

If `recursiveScanning` is true, the content of activated entries is itself scanned for triggers. This enables "chained" lore — entry A triggers, mentions B, B triggers, mentions C, etc. — up to `maxRecursionDepth`.

**When to use:**
- Complex fictional worlds where entries cross-reference each other.
- When you want mentioning one character to bring in their faction, their location, their history.

**When to avoid:**
- Small lorebooks (wastes compute, no benefit).
- Performance-sensitive chats (recursion has real cost).
- When entries are deliberately isolated (e.g., separate factions that shouldn't leak into each other).

**Mitigation:** use `preventRecursion: true` on specific entries to stop them from triggering further chains.

## Knowledge Retrieval Agent (Semantic Matching)

The `knowledge-retrieval` agent (in `packages/server/src/services/agents/knowledge-retrieval.ts`) does **embedding-based semantic search** across lorebook entries. This is complementary to keyword matching — it catches cases where the user talks about a concept without using the exact trigger word.

Enable via: create a `knowledge_retrieval` agent in the Agents panel. It runs pre-generation and injects semantically matched entries.

**Requirements:** entries need to be embedded (happens automatically when the agent is configured).

## Activation Conditions (Game-State Gating)

Entries can require game-state conditions to fire:
```json
{
  "name": "Forest Encounter",
  "keys": ["forest"],
  "content": "...",
  "activationConditions": [
    { "field": "time", "operator": "equals", "value": "night" },
    { "field": "location", "operator": "contains", "value": "forest" }
  ]
}
```
Both conditions must match (AND logic) for the entry to fire. Useful for Game Mode where the World State agent tracks live game state.

## Schedule (Time/Date/Location)

For time-based gating:
```json
{
  "schedule": {
    "activeTimes": ["night", "midnight"],
    "activeDates": ["2024-12-25"],
    "activeLocations": ["castle"]
  }
}
```

## AI Lorebook Maker

Marinara has a built-in `lorebook-maker` that generates structured entries from a topic prompt. Use the `POST /api/lorebook-maker/generate` endpoint (SSE stream). In the UI: Lorebooks panel → AI generator button.

**When to use:** to bootstrap a lorebook from a summary of a setting/world/topic. Don't expect the output to be final — treat it as a draft to edit.

## Best Practices

### Keyword choice
- Include variations: full names, first names, nicknames, titles.
- For common words, turn on `matchWholeWords` to avoid false positives.
- For proper nouns with unusual capitalization, consider `caseSensitive`.

### Entry length
- 1–3 short paragraphs is ideal. Entries over ~300 words tend to dominate context.
- If an entity has a lot of lore, split into multiple entries with overlapping keywords (general info + specific deep-dives).

### When NOT to use lorebooks
- **Small stable knowledge** — just put it in the character card.
- **Data that's always relevant** — put it in the card, skip the keyword machinery.
- **Fast-changing data** — lorebooks are for stable knowledge. Use webhook tools for live data.
- **Knowledge that's too big for any prompt** — at some point you need actual RAG. Marinara has `knowledge-retrieval` for semantic matching over lorebook entries; past that, you'd need a webhook to an external vector DB.

### When to use multiple lorebooks
- Separating world lore (global) from character-specific memories (character-scoped).
- Different settings/campaigns where only one should be active.
- Spellbooks (a Marinara-specific category) for combat ability lists.

## Migration from SillyTavern

Marinara imports SillyTavern lorebooks/world-info directly via Settings → Import. The schemas are mostly compatible; Marinara extends them with fields like `ephemeral`, `group`/`groupWeight`, `activationConditions`, `schedule`, and richer recursion controls.

## API Endpoints

- `GET /api/lorebooks` — list
- `GET /api/lorebooks/:id` — one lorebook (with entries)
- `POST /api/lorebooks` — create
- `PATCH /api/lorebooks/:id` — update
- `DELETE /api/lorebooks/:id` — delete
- `POST /api/lorebooks/:id/entries` — create entry
- `PATCH /api/lorebooks/:id/entries/:entryId` — update entry
- `DELETE /api/lorebooks/:id/entries/:entryId` — delete entry
- `POST /api/lorebook-maker/generate` — AI-generate (SSE)
- `GET /api/lorebooks/:id/export` — export JSON

## UI Location

- **Lorebooks panel** (right sidebar) — create, edit, attach to characters/chats.
- **World Info Inspector** — live view of which entries are active in the current chat, with token usage and keyword reasons.
