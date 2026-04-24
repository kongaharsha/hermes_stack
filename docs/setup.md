# Setup Walkthrough

The README covers the non-technical path — paste one block into Hermes, let the agent do the work. This doc is for readers who want to see every step explicitly before running it.

## Prerequisites

- **macOS 14+ or Linux.** I'm on macOS 14.3.1; the instructions are Mac-first. Linux users can swap LaunchAgent steps for systemd.
- **A Telegram account** for the bot interface.
- **A model API key** — OpenRouter is the easiest (one key, many models). OpenAI or Anthropic work too.

## Step 1 — Install Hermes

```bash
git clone https://github.com/NousResearch/hermes-agent ~/.hermes/hermes-agent
cd ~/.hermes/hermes-agent
# follow the installer — it sets up the ~/.hermes profile structure
# and the LaunchAgent for the gateway
```

After installation, verify:

```bash
hermes --version
hermes profile
```

You should see one profile (probably `default` or whatever you named it) and your model provider.

## Step 2 — Install gbrain

Install Bun (gbrain's runtime):

```bash
curl -fsSL https://bun.sh/install | bash
export PATH="$HOME/.bun/bin:$PATH"
```

Then clone and link:

```bash
git clone https://github.com/garrytan/gbrain.git ~/gbrain
cd ~/gbrain && bun install && bun link
gbrain --version    # should print 0.18.2 or later
gbrain init         # creates ~/.gbrain/brain.pglite (embedded Postgres)
```

## Step 3 — Create your brain repo

The brain is a plain git repo full of markdown. Create one:

```bash
mkdir -p ~/brain/{people,companies,concepts,projects,logs}
cd ~/brain && git init
```

Drop the three brain-concept templates from this repo into it:

```bash
cp ~/linkedin/hermes_stack/docs/brain-templates/*.md ~/brain/concepts/
gbrain import ~/brain/ --no-embed
gbrain embed --stale
```

You now have three concept pages in the brain (loyalty, airline prefs, service prefs) with placeholder fields.

## Step 4 — Set up the Telegram bot

1. Open Telegram, search for `@BotFather`
2. Send `/newbot`, pick a display name and username (must end in `bot`)
3. Copy the API token BotFather gives you
4. Add it to your profile's `.env`:

   ```bash
   echo "TELEGRAM_BOT_TOKEN=<paste-token-here>" >> ~/.hermes/.env
   # or, if you're using a named profile:
   # echo "TELEGRAM_BOT_TOKEN=..." >> ~/.hermes/profiles/<profile>/.env
   ```

5. Restart the gateway:

   ```bash
   launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway
   ```

6. Send "hi" to your new bot. It should reply.

Full details: [Hermes Telegram setup guide](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram).

## Step 5 — Register gbrain as an MCP server in Hermes

This is the critical wiring step. It gives Hermes direct tool access to gbrain (search, query, read pages, write pages) instead of shelling out.

```bash
hermes mcp add gbrain --command $(which gbrain) --args serve
hermes mcp list    # should show gbrain enabled
```

The gateway hot-reloads automatically after `mcp add`. Your next Hermes turn includes 30+ gbrain tools in the tool list.

**Note on env:** if your OpenAI key lives in a profile-scoped `.env` that the subprocess doesn't inherit, you may need to edit `~/.hermes/profiles/<profile>/config.yaml` directly and set `mcp_servers.gbrain.env.OPENAI_API_KEY` to the key value. Embeddings require it.

## Step 6 — Install the two skills

```bash
git clone https://github.com/kongaharsha/hermes_stack.git ~/linkedin/hermes_stack

PROFILE=$(cat ~/.hermes/active_profile 2>/dev/null || echo default)
if [ "$PROFILE" = "default" ]; then
  SKILLS_DIR="$HOME/.hermes/skills"
else
  SKILLS_DIR="$HOME/.hermes/profiles/$PROFILE/skills"
fi

cp -r ~/linkedin/hermes_stack/skills/airline-checkin "$SKILLS_DIR/"
cp -r ~/linkedin/hermes_stack/skills/laundry-booking "$SKILLS_DIR/"
ls "$SKILLS_DIR"
```

Hermes discovers `SKILL.md` files under its skill directories on every turn. No restart needed.

## Step 7 — Fill in your data

Open these three files in your editor and replace placeholders with your actual info:

- `~/brain/concepts/loyalty-info.md` — airline loyalty numbers, KTN, passport
- `~/brain/concepts/airline-preferences.md` — seat/bag/hazmat defaults
- `~/brain/concepts/service-preferences.md` — laundry provider URL and prefs

After editing, re-index:

```bash
gbrain sync --repo ~/brain
gbrain embed --stale
```

## Step 8 — Test

Text your Telegram bot:

```
what do we know about my airline preferences?
```

If Hermes replies with what you filled in (not a generic answer), the full stack is wired up.

Try the skills:

```
check me in for my United flight, confirmation ABC123
```

```
book a laundry pickup for next Tuesday morning
```

## Gotchas

See [LESSONS.md](LESSONS.md) for the real issues I hit during setup and how I fixed them.
