# Architecture

A plain-language walk through how the three pieces fit together.

## The picture

```
                ┌──────────────┐
                │   Telegram   │   You text the bot.
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │Hermes Gateway│   Runs in background (LaunchAgent on Mac).
                │  ~/.hermes/  │   Receives messages from Telegram or terminal.
                └──────┬───────┘   Boots an agent turn per incoming message.
                       │
           ┌───────────┴───────────┐
           ▼                       ▼
   ┌──────────────┐        ┌──────────────┐
   │   Skills     │        │    gbrain    │   A local markdown knowledge base.
   │  (markdown)  │        │ (MCP server) │   Exposes search/query/read/write
   │              │        │              │   as tools the agent can call.
   │ airline-     │        │              │
   │ checkin      │        └──────┬───────┘
   │ laundry-     │               │
   │ booking      │               ▼
   │  ...         │        ┌──────────────┐
   └──────────────┘        │   ~/brain/   │   Plain markdown. One page per
                           │   (git repo) │   person, company, concept,
                           │              │   project. You can edit directly.
                           └──────────────┘
```

## What each piece does

### Hermes

The runtime. It's a Python gateway that:

- **Listens** on Telegram (via bot API), CLI, or any registered messaging platform
- **Boots an agent** per incoming message — loads SOUL.md (your persona), AGENTS.md (conventions), the skills index, and the MCP tool list
- **Routes** to a model provider (OpenRouter, Anthropic, OpenAI, local models)
- **Manages** its own session memory, approval workflow, and cron jobs

Hermes doesn't know your preferences or facts about the world. It knows how to *talk* and how to *use tools*.

### gbrain

Your personal knowledge base, exposed as tools.

- Stores markdown files in `~/brain/` (you can edit them by hand at any time)
- Indexes them into an embedded Postgres + pgvector database at `~/.gbrain/brain.pglite`
- Exposes hybrid search (vector + keyword), graph traversal, page read/write via a standalone `gbrain serve` process speaking MCP
- Hermes calls those tools like any other (alongside shell, browser, filesystem)

The market-standard name for this is "retrieval-augmented generation" — but the framing matters: gbrain isn't a database you feed the LLM. It's a memory layer the LLM reads and writes to, like a human flipping through a notebook.

### Skills

Markdown playbooks that tell Hermes how to do specific tasks. Each skill is a directory with `SKILL.md` inside:

```markdown
---
name: airline-checkin
description: Automate airline check-in via local Chrome (CDP).
---

# Airline Check-in

[plain English workflow and resilience patterns]
```

Hermes loads every `SKILL.md` it can find under its configured skills directories (per-profile, plus `external_dirs` in config.yaml pointing at extra locations like `~/gbrain/skills/`). The frontmatter's `name` and `description` go into the model's system prompt. The body is loaded when the LLM decides to "run" that skill — so you can write arbitrarily long workflows without bloating the system prompt.

## The turn lifecycle

What happens when you text "check me in for my flight":

1. **Telegram → Hermes gateway** delivers the message.
2. **Agent turn starts.** Hermes assembles the system prompt: SOUL.md (who you are), AGENTS.md (conventions), every discovered `SKILL.md`'s frontmatter, and the live tool list including the gbrain MCP tools.
3. **Model thinks.** It sees the `airline-checkin` skill description in the prompt and a fresh user message about a flight. It decides to load that skill.
4. **Skill body loads.** The full `airline-checkin/SKILL.md` content comes into context. It says "read `~/brain/concepts/loyalty-info.md` for identity" and "open Chrome via CDP."
5. **Model calls `gbrain.get_page("concepts/loyalty-info")`** — your confirmation number, KTN, loyalty status flow back.
6. **Model calls a shell tool** — opens Chrome for Testing on port 9222, navigates to the airline site.
7. **Browser automation runs.** The model issues `click`, `fill`, `wait` commands one by one. Each gets a DOM snapshot back. Proceeds through the wizard.
8. **Boarding pass appears.** Model captures the link, texts it to you, and writes a timeline entry to `~/brain/logs/airline-checkins.md`.
9. **gbrain re-indexes.** The new log entry becomes searchable next time.

The point: none of this requires custom code. Skills are prose. Data is markdown. Tools are MCP. Everything composes.

## Where state lives

| State | Location | Editable? |
|---|---|---|
| Your preferences, people, projects | `~/brain/**/*.md` | Yes, directly — gbrain watches for changes |
| Agent persona | `~/.hermes/profiles/<p>/SOUL.md` | Yes |
| Session transcripts | `~/.hermes/profiles/<p>/sessions/*.json` | Read-only in practice |
| Skill playbooks | `~/.hermes/profiles/<p>/skills/` and `~/gbrain/skills/` | Yes, drop in new SKILL.md files |
| Gbrain index | `~/.gbrain/brain.pglite/` | Rebuildable from `~/brain/` |
| Telegram token, API keys | `~/.hermes/.env` | Yes (secrets) |
| Cron jobs | `~/.hermes/cron/jobs.json` | Yes (or via `hermes cron` CLI) |

Losing the gbrain index is survivable — `gbrain init && gbrain import ~/brain/ --no-embed && gbrain embed --stale` rebuilds it from the markdown source of truth. Losing `~/brain/` is not — back it up to a private git remote.

## Why this shape

Three design choices worth calling out:

**Markdown is the source of truth, not the database.** I can edit a person's brain page in my editor, commit to git, push to a remote. Gbrain's index is rebuildable. The data outlives any particular tool.

**Skills are prose, not code.** A skill is a workflow description the model reads. New skills don't require a deploy, a build, or even a restart — drop in a markdown file, it shows up on the next turn. This makes the system easy to iterate on and hard to lock in.

**Tools are MCP-based.** Anything the agent can do is exposed as a tool it calls. Hermes has shell, browser, filesystem. Gbrain adds search/read/write for your brain. If you want calendar access, install a calendar MCP server. The surface grows by composition.
