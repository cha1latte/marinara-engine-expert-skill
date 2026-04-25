# Agents

Agents are **autonomous LLM sub-systems that run during message generation**. They handle side tasks like state tracking, prose quality enforcement, continuity checking, image generation, and music control — running before, alongside, or after the main response.

All agents are **disabled by default**. Users enable only what they need. Each enabled agent adds latency and token cost per turn, so the right recommendation is usually the minimum viable set.

**Source of truth:** `packages/server/src/services/agents/` (agent-executor.ts, agent-pipeline.ts, knowledge-retrieval.ts), `packages/shared/src/schemas/agent.schema.ts`, `packages/shared/src/constants/agent-prompts.ts` (default prompt templates for built-ins).

## Agent Phases

An agent runs in one of three phases, which determine when it fires relative to the main response generation:

### `pre_generation`
Runs **before** the main model is called. Can inject context, review the prompt, or rewrite directives that will be included in generation.

**Good for:**
- Context injection (semantic lorebook retrieval, knowledge sources)
- Prompt review and quality scoring
- Prose directives ("use these devices this turn, avoid these words")
- Scheduling (generating character schedules for conversation mode)
- Narrative pacing directives (Director-style injections)

**Cost:** Adds latency to every turn before the user sees the response starting.

### `parallel`
Runs **at the same time as** the main model. Doesn't block the main response.

**Good for:**
- Image generation based on the current scene
- Music/Spotify suggestions
- Echo messages (absent characters reacting to the current scene)
- Combat mechanics that are computed separately from narration
- Autonomous messaging triggers in conversation mode

**Cost:** Token cost adds up but doesn't delay user-visible output.

### `post_processing`
Runs **after** the main model finishes. Receives the generated message and can extract, update, or even rewrite it.

**Good for:**
- State extraction (world state, character status, quest progress)
- Continuity checking and correction
- Sprite/expression picking based on emotional content
- Background selection based on scene
- Copy-editing and grammar passes
- Rolling summaries for long chats
- Auto-generating lorebook entries from the story

**Cost:** Adds latency after generation finishes but before results fully settle.

## Built-In Agents (as of recent versions)

There are ~25 built-in agents. The README groups them by role; here they are by phase.

### Pre-Generation
- **prose-guardian** — Analyzes recent messages for repetition, picks rhetorical devices to use/avoid, enforces sentence variety, rotates sensory channels. Heavy but powerful for literary quality.
- **narrative-director** — Injects pacing directives, dramatic beats, cliffhangers, scene transitions.
- **continuity-checker** — Flags contradictions with established lore and facts. Often run as a reviewer, can be post-processing too.
- **prompt-reviewer** — Scores and reviews the assembled prompt, suggests improvements.
- **knowledge-retrieval** — Semantic search across lorebook entries; injects matches pre-generation. This is the closest thing to traditional RAG in the engine.
- **schedule-planner** — Generates/maintains character weekly schedules for conversation mode.
- **immersive-html** — Renders custom HTML/CSS widgets in messages (for creative formatting).
- **response-orchestrator** — For group chats, decides which character speaks next.

### Parallel
- **echo-chamber** — Simulates a live-stream chat or sidebar reactions; absent characters react to the scene.
- **illustrator** — Generates images based on story scenes via an image provider.
- **combat** — Handles turn-based combat mechanics, dice rolls, encounters.
- **autonomous-messenger** — Manages characters sending messages unprompted in conversation mode.

### Post-Processing
- **consistency-editor** — Edits the generated response to fix factual errors and tracker contradictions. Can make surgical corrections.
- **world-state** — Extracts date, time, location, weather, temperature from narration.
- **expression-engine** — Picks character sprite expressions based on emotional content.
- **quest-tracker** — Manages quest objectives, completion, rewards.
- **background** — Picks the appropriate background image for the current scene.
- **character-tracker** — Tracks which characters are present and their moods, relationships, appearance, outfit, stats.
- **persona-stats** — Updates player and character RPG stats.
- **custom-tracker** — User-defined tracking (any JSON data the user wants to monitor).
- **lorebook-keeper** — Auto-generates lorebook entries from ongoing story (writes back to the lorebook).
- **chat-summary** — Rolling conversation summaries for long-term context.
- **spotify-dj** — Suggests thematic music/playlists for the scene mood.
- **cyoa-choices** — Generates 2–4 in-character choices after each response.
- **love-toys-control** — Controls Buttplug.io haptic devices (yes, really).

Each has a default prompt template in `packages/shared/src/constants/agent-prompts.ts`. **Users can override any template** via the Agent Editor.

## Custom Agents

Users can create their own agents from scratch. The schema:

```typescript
{
  type: string,              // any string identifier
  name: string,              // display name
  description: string,
  phase: "pre_generation" | "parallel" | "post_processing",
  enabled: boolean,
  connectionId: string | null,  // separate LLM connection, optional
  promptTemplate: string,    // the system prompt for this agent
  settings: object,          // arbitrary config
}
```

**A custom agent is essentially a scoped LLM call with its own prompt, running in a specific phase.** The agent gets context about the current chat and is expected to return output in a structured form (depending on its `agentResultType`).

### Result types (what the agent returns)

From the schema:
- `game_state_update` — JSON update to world state
- `text_rewrite` — replacement text for the main message
- `sprite_change` — sprite expression update
- `echo_message` — a reaction message shown elsewhere
- `quest_update` — quest state changes
- `image_prompt` — a prompt for image generation
- `context_injection` — plain text to inject into the next generation's context
- `continuity_check` — flags for continuity issues
- `director_event` — narrative event directive
- `lorebook_update` — new/updated lorebook entries
- `prompt_review` — score + feedback on the assembled prompt
- `background_change` — change the current background
- `character_tracker_update` — update tracked character state
- `persona_stats_update` — update persona stats
- `chat_summary` — rolling summary text
- `spotify_control` — Spotify playback action
- `secret_plot` — hidden plot thread

**The default (and most common) for custom agents is `context_injection`** — the agent returns text, and that text is injected into the next turn's context. This is the flexible option and works for most user-defined agents.

### Tool-using agents

Agents can call tools too — the agent executor supports tool-calling loops. An agent with `toolContext` set can make tool calls, receive results, and continue until it's done. This allows custom agents to do things like "look up current weather, then inject that as world state context."

See `packages/server/src/services/agents/agent-executor.ts` for the loop implementation.

## When to Use an Agent vs. Other Surfaces

**Agent** — automatic, per-turn, background.
**Tool** — model-invoked, on-demand, when the model decides it's needed.
**Lorebook** — keyword-triggered, no LLM call needed beyond pattern matching.

Rule of thumb:
- **Needs to happen every turn without being prompted?** → Agent.
- **Needs to happen when a topic comes up?** → Lorebook.
- **Needs to happen when the user asks for it?** → Tool.
- **Always relevant static info?** → Character card description.

### Good custom agent candidates
- A "historical accuracy checker" for a period RP (runs post-processing, flags anachronisms)
- A "tone enforcer" for a specific writing style (runs pre-generation, injects style directives)
- A "relationship tracker" that updates a custom JSON state each turn
- A "foreshadowing director" that injects subtle plot hints when specific conditions are met

### Bad custom agent candidates
- "Call my API to look up X" — use a tool instead; agents shouldn't be doing user-requested actions.
- "Remember things the user says" — use semantic memory (built-in) or lorebook-keeper, not a custom agent.
- "Rewrite every message to be more dramatic" — prose-guardian already does this flavor of work, probably better than your custom agent will.
- "Respond as a different character" — that's not an agent, that's a group chat.

## Tuning and Pitfalls

### Stacking too many agents
Every enabled agent costs a separate LLM call. A chat with 8 agents enabled will make 9 total LLM calls per turn (8 agents + the main response). Token cost and latency add up fast.

**Recommendation for most characters:** 0–3 agents. More only if the project specifically benefits.

### Choosing the agent's LLM
Each agent can have its own `connectionId`. Useful patterns:
- Use a **cheap/fast model** (Gemma sidecar, GPT-4o mini, Haiku) for trackers and extractors.
- Use the **main model** for prose-guardian, continuity, and anything that needs literary sensitivity.
- Use a **vision-capable model** for agents that need to see generated images (rare).

The built-in Gemma 4 E2B sidecar is specifically designed to handle tracker agents locally, keeping tokens off your main provider bill.

### Conflicting agents
If you enable both `world-state` and a custom tracker that also manages location/weather, they can step on each other. Disable one, or use the `custom-tracker` with locked fields.

### Agent prompt templates
Default templates are decent but generic. For production use, customize the `promptTemplate` to match your specific style, genre, or constraints. The default prompts are in `packages/shared/src/constants/agent-prompts.ts` — reading them gives a good starting point for custom variants.

### Debugging agents
- The Agent Editor shows the agent's last run and output.
- Enable verbose logging if the agent seems to misbehave.
- Run the agent with `agent-executor`'s built-in retry mechanism if needed.

## Example: A Custom "Historical Accuracy" Agent

For a Regency-era roleplay:

```json
{
  "type": "historical_accuracy_regency",
  "name": "Regency Accuracy Check",
  "description": "Flags anachronisms in the generated message (post-1820 references, modern slang, wrong social conventions).",
  "phase": "post_processing",
  "enabled": true,
  "connectionId": null,
  "promptTemplate": "You are a Regency-era historian (1811-1820, English). You'll receive a passage of narration/dialogue. Your ONLY job is to flag anachronisms:\n- References to events, inventions, or people after 1820\n- Modern slang or phrasing that's out of period\n- Incorrect social conventions (dress, titles, conduct)\n- Wrong geography or currency\n\nReturn a JSON object:\n{ \"issues\": [\"description of issue 1\", \"...\"], \"clean\": true|false }\n\nIf no issues, return { \"issues\": [], \"clean\": true }.\n\nThe passage:\n{{message}}",
  "settings": {}
}
```

This would run after every generation, return a structured check, and (if wired to the consistency-editor) trigger rewrites on issues.

## API Endpoints

- `GET /api/agents` — list
- `POST /api/agents` — create
- `PATCH /api/agents/:id` — update
- `PUT /api/agents/:id` — replace
- `DELETE /api/agents/:id` — delete
- `POST /api/agents/:id/toggle` — enable/disable
- Echo message endpoints for managing echo-chamber output

## UI Location

**Agents Panel** (right sidebar → sparkles icon). Shows built-in agents (with toggles and editable prompt templates) and a Custom Tools subsection. Each agent has a full editor for its prompt, connection override, and settings.
