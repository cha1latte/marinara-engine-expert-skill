# Marinara Engine Consultant Skill
 
A Claude skill that turns Claude into an expert consultant for [Marinara Engine](https://github.com/Pasta-Devs/Marinara-Engine), the open-source local AI roleplay/chat frontend.
 
## What it does
 
When you describe a project idea — a character, an assistant, a tool integration, a UI mod — this skill makes Claude:
 
1. **Identify which Marinara Engine surfaces your idea touches** (character cards, lorebooks, custom tools, agents, extensions)
2. **Present 2–4 architecture options with honest tradeoffs**
3. **Recommend one**, with rationale
4. **Offer to build it** — character card JSON, tool definitions, webhook backends, extension code
It knows the engine's real schemas, API endpoints, execution model, and limits. It pushes back when you're heading toward an anti-pattern (cramming everything in the system prompt, trying to use `script` tools for network calls, stacking agents unnecessarily).
 
## What's included
 
| File | What it covers |
|---|---|
| `SKILL.md` | Main instructions — how to consult, decision hierarchy, anti-patterns, calibration |
| `references/architecture.md` | Big picture: client/server stack, chat modes, prompt assembly, data storage, API surface |
| `references/character-cards.md` | V2 card spec, field-by-field prompt behavior, the Professor Mari pattern, card templates by use case |
| `references/lorebooks.md` | Entry fields, activation patterns (keyword, regex, selective, grouped, sticky, ephemeral), scoping, token budgets, recursive scanning |
| `references/custom-tools.md` | Execution types (static/webhook/script), parameter schemas, sandbox limits, webhook design, examples |
| `references/extensions.md` | The `marinara` client API (`addStyle`, `addElement`, `apiFetch`, `on`, `observe`), patterns, pitfalls |
| `references/agents.md` | Phases (pre/parallel/post), all 25 built-ins, custom agent schema, result types, tuning advice |
| `references/decision-guide.md` | The 7-question decision tree for picking the right architecture, common stacks, red flags |
| `assets/*.json` / `assets/*.md` | Starter templates for character cards, custom agents, lorebook entries, and webhook tools |
 
## Installation
 
Copy this folder into your Claude skills directory:
 
```
your-skills-folder/
└── marinara-engine-consultant/
    ├── SKILL.md
    ├── assets/
    └── references/
```
 
The skill triggers automatically when you mention Marinara Engine, Professor Mari, or describe building characters, tools, extensions, or agents for an AI chat frontend.
 
## Example prompts
 
- "I want to build an assistant that knows everything about my band's gear and tour schedule"
- "How do I add tool calling to my character so it can look up WordPress site configs?"
- "What's the difference between a custom tool and a custom agent?"
- "I want to build an extension that shows a token counter in the chat UI"
- "Help me migrate my SillyTavern characters to Marinara"
## Note on currency
 
Marinara Engine is under active development. The reference files are a snapshot. The skill is configured to fetch from the GitHub repo when it's uncertain about current behavior — but the references may lag behind the latest release. PRs welcome if something's out of date.
 
## License
 
MIT
