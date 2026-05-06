# Hermes Stack

> My personal AI agent setup. A running record of how I turned chat-window AI into something that actually does my errands.

## How I got here

I've been a heavy user of ChatGPT over the last year or so. Useful for writing, thinking, research — but each conversation was a blank slate. It never *knew* me. Every time I asked about my travel preferences, my projects, or the conversation we'd had last week, I started over.

I wanted more leverage. Not a smarter chatbot — an assistant that compounds. Something that remembers my preferences, knows my history, can execute tasks on my behalf.

Last month I read about [Hermes](https://github.com/NousResearch/hermes-agent) from Nous Research — an open-source, self-improving personal agent. I started using it for a real task that had been sitting on my plate. It worked well enough that I trusted it with more. I moved my inbox triage onto it, then scheduling, then eventually errands.

The first few days were rough. By day four it had learned enough about me that asking "what are my airline preferences?" returned what I'd actually written, not a generic guess. By week two, I was texting a bot to check me in for flights.

This repo is the setup, the two most useful skills I've built so far, and the docs to reproduce it.

## The stack

Three pieces, each independently useful:

| Piece | What it is | What it does for me |
|---|---|---|
| **[Hermes](https://github.com/NousResearch/hermes-agent)** | The agent — the thing I actually talk to (via Telegram or terminal) | Routes my messages, runs tools, maintains a persona that matches how I work |
| **[gbrain](https://github.com/garrytan/gbrain)** | A local knowledge base: markdown files + search | Stores everything Hermes knows about me — preferences, people, projects, meeting notes |
| **Skills** (this repo) | Markdown playbooks that teach Hermes specific tasks | Airline check-in, laundry booking, and more to come |

The shape: you message Hermes → it checks gbrain for context → picks the right skill → runs it → writes results back to gbrain. Skills are just markdown, no compilation, no deploy. Drop a new file in the skills directory and Hermes picks it up.

```
You → Telegram → Hermes → reads gbrain for context
                        → picks a skill
                        → executes (shell, browser, API calls)
                        → writes back to gbrain
```

## The three skills

### Airline check-in

Text the bot the night before a flight. It opens a browser locally (not a cloud browser — airline anti-bot systems block those), logs in, pulls the confirmation number and loyalty profile from my brain, picks my seat preference, handles the hazmat declaration, and texts me the boarding pass.

Hard-won lessons baked in: airlines inject survey popups that silently eat clicks; the KTN field goes invisible if you don't focus it; `networkidle` hangs forever on airline portals because of third-party trackers. Documented in the skill.

[See the full skill →](skills/airline-checkin/SKILL.md)

### Laundry booking

Text the bot when I need a pickup. It confirms the dates with me, logs into the laundry service, books the pickup and drop-off windows, and logs the order.

Asks for explicit confirmation before each critical step. I've had enough automation surprises to prefer "pause and check" over "assume and proceed."

[See the full skill →](skills/laundry-booking/SKILL.md)

### House inventory

Text the bot with a description or photo of items you're storing or cataloging. It correctly indexes objects into a standardized container hierarchy and maintains a searchable home inventory for you.

Helps keep track of storage boxes and move-planning by converting visual data into a structured Markdown inventory in your brain.

[See the full skill →](skills/house-inventory/SKILL.md)

## Install

A Mac with Terminal access, a Telegram account, and about 45 minutes.

### Step 1 — Install Hermes

Follow the [Hermes install guide](https://github.com/NousResearch/hermes-agent). It walks you through installation, picking a model provider (OpenRouter works well — one key, many models), and creating your first profile.

### Step 2 — Set up the Telegram bot

Follow the [Hermes Telegram setup guide](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram):

1. Create a bot via `@BotFather` in Telegram — you'll receive a token
2. Add the token to `~/.hermes/.env` as `TELEGRAM_BOT_TOKEN=<your-token>`
3. Restart Hermes — it'll connect automatically

Verify: send "hi" to your bot. It should reply.

### Step 3 — Let Hermes install the rest

Open Hermes (either in your terminal by typing `hermes`, or by messaging your Telegram bot — either works). Paste this block:

```
Retrieve and follow the instructions at:
https://raw.githubusercontent.com/garrytan/gbrain/master/INSTALL_FOR_AGENTS.md

After gbrain installs, do these additional steps — paste the actual command output for each, don't summarize:

1. Register gbrain as an MCP server so I can query it as a tool:
   hermes mcp add gbrain --command $(which gbrain) --args serve
   Then verify with: hermes mcp list

2. Clone https://github.com/kongaharsha/hermes_stack into ~/linkedin/hermes_stack
   Then verify: ls ~/linkedin/hermes_stack/skills/ should show airline-checkin/ and laundry-booking/

3. Find my active Hermes profile and its skills directory:
   hermes profile
   (Note the Path field. Skills go in Path/skills/. If the profile is "default", skills go in ~/.hermes/skills/.)

4. Copy both skills into that directory:
   cp -r ~/linkedin/hermes_stack/skills/airline-checkin <skills-dir>/
   cp -r ~/linkedin/hermes_stack/skills/laundry-booking <skills-dir>/
   Then verify: ls <skills-dir>/ shows both directories.

5. Copy brain templates into ~/brain/concepts/ (create the directory if it doesn't exist):
   mkdir -p ~/brain/concepts
   cp ~/linkedin/hermes_stack/docs/brain-templates/*.md ~/brain/concepts/
   Then verify: ls ~/brain/concepts/

6. Re-import and embed the brain so it picks up the new template files:
   gbrain import ~/brain/ --no-embed && gbrain embed --stale

When done, tell me which fields I need to fill in to make the airline and laundry skills work. Don't auto-install packages I haven't approved. If any step fails, STOP and report.
```

Hermes walks through each step, asks for API keys and confirmations where needed, and reports back. Budget ~30 minutes for this step.

### Step 4 — Fill in your data

Hermes will point you at three files in `~/brain/concepts/`:

- `loyalty-info.md` — your frequent-flyer numbers, KTN, passport (for the airline skill)
- `airline-preferences.md` — default seat and bag preferences per carrier
- `service-preferences.md` — your laundry provider URL and preferences

Open each, fill in the placeholders. It's just markdown.

### Step 5 — Try it

Text your Telegram bot:

```
what do we know about my airline preferences?
```

It should read from your brain and answer with what you filled in. Then try:

```
check me in for my flight tomorrow
```

or

```
book a laundry pickup for next Tuesday
```

The bot will ask for anything it needs — airline, confirmation number, pickup window. The skills take over from there.

## What's next

I'm adding skills as I find things I do often enough to automate. Next on my list: inbox triage, calendar-aware meeting prep, receipt-to-brain pipeline. The stack compounds — each new skill gets cheaper to build because Hermes and gbrain already know the patterns.

If you set this up, I'd love to see what you build on top.

## More docs

- [Setup walkthrough](docs/setup.md) — detailed installation notes for the technical path
- [Architecture](docs/architecture.md) — how the pieces fit together under the hood
- [Lessons](docs/LESSONS.md) — honest notes on what broke and how I fixed it

## License

MIT. See [LICENSE](LICENSE). Use it, fork it, remix it.

---

**Credits:** Hermes is built by [Nous Research](https://nousresearch.com). gbrain is built by [Garry Tan](https://x.com/garrytan). This repo is my configuration and skills — the engines are theirs.
