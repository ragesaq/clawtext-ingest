# clawtext-ingest

> ⚠️ **This package has been merged into [ClawText](https://github.com/ragesaq/clawtext).**
>
> As of **ClawText v1.4.0**, the full ingest engine is bundled into the main ClawText package. You no longer need to install `clawtext-ingest` separately.

---

## Migration

**Before (clawtext-ingest as separate package):**
```bash
npm install clawtext-ingest
clawtext-ingest ingest-files --input="docs/**/*.md" --project=myproject
```

**After (ClawText 1.4+, everything in one place):**
```bash
# Install ClawText (ingest is now bundled)
# Add to openclaw.json and restart gateway — see ClawText install docs

# Same CLI commands work as before
clawtext-ingest ingest-files --input="docs/**/*.md" --project=myproject
clawtext-ingest-discord fetch-discord --forum-id 123456789
```

All CLI commands, adapters, and the `ClawTextIngest` API are identical — they just ship from the main ClawText package now.

---

## Go here instead

**→ [github.com/ragesaq/clawtext](https://github.com/ragesaq/clawtext)**

- Install docs: [AGENT_INSTALL.md](https://github.com/ragesaq/clawtext/blob/main/AGENT_INSTALL.md)
- Ingest docs: [docs/INGEST.md](https://github.com/ragesaq/clawtext/blob/main/docs/INGEST.md)
- Discord ingest: `npm run ingest:discord` or `clawtext-ingest-discord`

---

## Why merged?

Ingest is one of three core lanes in ClawText's memory platform:

| Lane | What it does |
|------|-------------|
| **Working Memory** | Conversational capture, tiered retrieval, hot cache injection |
| **Knowledge Ingest** | Bulk-load repos, docs, Discord threads, exports ← *this was clawtext-ingest* |
| **Operational Learning** | Capture failures, review patterns, promote durable guidance |

Keeping ingest as a separate install created an unnecessary split. Everything belongs in one place with one install story, one version, one docs story.

---

*This repo is archived. No further updates.*
