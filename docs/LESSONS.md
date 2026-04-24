# Lessons

Real issues I hit setting this up, and how I fixed them. Written as a reference for anyone hitting the same walls.

## Duplicate gateway services fight over the Telegram bot

**Symptom:** Messages to the bot go unanswered, or answered by a different profile than you expected. Logs show `409 Conflict` from Telegram's `getUpdates`.

**Cause:** Telegram's bot API allows one connection per bot token. If you have both `ai.hermes.gateway.plist` and `ai.hermes.gateway-<profile>.plist` loaded in launchd, they race for the connection. Whichever polls first wins; the other gets kicked.

**Fix:** Keep only one gateway LaunchAgent loaded. The default (`ai.hermes.gateway.plist`) reads `~/.hermes/active_profile` to decide which profile to serve, so you rarely need a sub-profile LaunchAgent at all.

```bash
launchctl unload ~/Library/LaunchAgents/ai.hermes.gateway-<profile>.plist
```

## Stale PID file crashes the gateway in a restart loop

**Symptom:** After unloading a sub-profile LaunchAgent, the remaining gateway crash-loops with "PID file race lost to another gateway instance."

**Cause:** The dead gateway left `~/.hermes/profiles/<profile>/gateway.pid` pointing at a PID that no longer exists. The surviving gateway reads it, thinks another gateway is active, exits. KeepAlive restarts it. Cycle repeats.

**Fix:** Remove the stale PID file. KeepAlive respawns cleanly.

```bash
rm ~/.hermes/profiles/<profile>/gateway.pid
```

## OPENAI_API_KEY is shell-inherited, not profile-local

**Symptom:** `gbrain embed` from the terminal works fine, but when Hermes runs the same command it gets a 401 from OpenAI.

**Cause:** Your shell rc (`~/.zshrc` or `~/.env`) has the key. The Hermes LaunchAgent inherits login env at startup, but subprocesses Hermes spawns may not get it — depends on how the tool runs. Worse: if you have an old/revoked key in `~/.zshrc` and a fresh one in `~/.env`, shell order picks the stale one first.

**Fix:** Put the key in `~/.hermes/.env` (gateway-level, loaded by Hermes on startup, propagated to all agent subprocesses). Reconcile any duplicate declarations in shell rc files.

```bash
echo "OPENAI_API_KEY=<your-real-key>" >> ~/.hermes/.env
launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway
```

## gbrain schema migration fails on upgrade

**Symptom:** `gbrain init` errors with `column "source_id" does not exist` (or similar column mismatch) after upgrading gbrain.

**Cause:** Gbrain's `initSchema()` runs the embedded schema SQL (which assumes the new schema) *before* migrations that add missing columns. On an old brain, `CREATE INDEX IF NOT EXISTS ... ON pages(source_id)` fires against a table without that column yet.

**Fix:** Nuke and rebuild. Your `~/brain/` markdown is the source of truth — the index is rebuildable.

```bash
cp -r ~/.gbrain ~/.gbrain.backup-$(date +%Y%m%d-%H%M%S)   # backup first
rm -rf ~/.gbrain/brain.pglite
gbrain init
gbrain import ~/brain/ --no-embed
gbrain embed --stale
gbrain extract links --source db
gbrain extract timeline --source db
```

This takes ~2 minutes for a small brain. Longer for bigger ones.

## PGLite corruption after concurrent access

**Symptom:** `gbrain jobs list` and other queue reads crash with `PGLite failed to initialize its WASM runtime... Aborted()`. Even after the autopilot process stops.

**Cause:** Two processes wrote to the PGLite file at the same time. `postmaster.pid` gets set to a sentinel value (`-42`), PGLite can't recover.

**Fix:** If `postmaster.pid` is the only corruption, removing it sometimes lets PGLite recover. In my case it didn't — I had to restore from backup.

```bash
# Try the cheap fix first
rm ~/.gbrain/brain.pglite/postmaster.pid
gbrain doctor

# If that doesn't work, rebuild from backup
mv ~/.gbrain/brain.pglite ~/.gbrain/brain.pglite.broken-$(date +%s)
cp -r ~/.gbrain.backup-YYYYMMDD-HHMMSS/brain.pglite ~/.gbrain/brain.pglite
# or nuke and rebuild from ~/brain/ markdown as above
```

**Prevention:** don't let autopilot run at the same time as manual `gbrain` commands or agent-driven gbrain calls. Either leave autopilot off and rely on cron-driven syncs, or scope autopilot to quiet hours when nothing else is writing.

## Agent hallucinates setup completion

**Symptom:** You ask Hermes to set up gbrain. It reports "successfully registered 4 jobs, health 75/100, schema v13" — but nothing actually happened. Config unchanged. Jobs queue empty.

**Cause:** The agent ran commands, but the commands failed silently or returned garbage (e.g., invalid flags like `--job` and `--schedule` to `gbrain jobs submit`). Instead of reporting errors, the agent wrote a plausible-sounding summary based on what it *intended* to do.

**Fix:** When asking an agent to do setup work, include an explicit verification instruction in the prompt:

> After each command, paste the actual output. Do not summarize. If any step fails, STOP and report.

This eliminates the "plausible summary" failure mode.

## Hermes loads skills but doesn't reach for them

**Symptom:** You ask "what do we know about Pedro?" — Hermes answers from session memory instead of running `gbrain query`. The gbrain skills are discoverable in `~/gbrain/skills/` but the model isn't invoking them.

**Cause:** The skills are in the prompt but Hermes's native tools (`session_search`, file-read) are faster and cheaper. The model takes the path of least resistance.

**Fix:** Register gbrain as an MCP server so its operations become first-class tools in the tool list, not just workflow descriptions:

```bash
hermes mcp add gbrain --command $(which gbrain) --args serve
```

Once `mcp__gbrain__query` and `mcp__gbrain__get_page` show up as tools alongside `session_search`, the model starts using them — especially with SOUL.md's "brain-first lookup" directive pointing at them.

## The macOS 26.3 WASM bug warning is misleading

**Symptom:** Any gbrain command fails with `PGLite failed to initialize its WASM runtime. This is most commonly the macOS 26.3 WASM bug: https://github.com/garrytan/gbrain/issues/223`.

**Cause:** The error message is a heuristic. It names the most common culprit even if your OS is different. The real cause is almost always concurrent-writer corruption (see above) or a stale lock file.

**Fix:** Don't trust the OS version in the error. Check the actual state — `ls ~/.gbrain/brain.pglite/` for stale `postmaster.pid` or lock files, try the cheap fix, fall back to rebuild.

---

The meta-lesson: **every new piece you add to this stack is a new coordination point**. Two gateways, two DBs, three env-file locations, an autopilot that wants exclusive access — the failures are rarely in any single component, they're at the seams. When something breaks, check what's racing what.
