# ClawText-Ingest

**Version:** 1.3.0 | **Status:** Production — Active

Multi-source ingestion library for the ClawText RAG system. Fetches messages from Discord (forums, channels, threads), filters text-only content, deduplicates via SHA1, and writes structured markdown cluster files that ClawText indexes for automatic context injection.

---

## What It Does

ClawText-Ingest is the data pipeline half of the ClawText system. It:

1. **Fetches** — Pulls messages from Discord via bot API (handles pagination, rate limits)
2. **Filters** — Text-only: strips attachments and embeds, skips empty messages
3. **Deduplicates** — SHA1 hash per message; already-seen messages are silently skipped
4. **Writes** — Per-thread markdown files with YAML frontmatter to `~/memory/clusters/`
5. **Triggers rebuild** — Calls `build-clusters.js` to refresh the RAG search index

Safe to run repeatedly. Running the same ingest 100 times produces the same result.

---

## How It Fits Into the Pipeline

```
 Discord Forums & Channels
         │
         ▼
 ┌───────────────────────┐
 │   ingest-all.mjs      │  Master script — runs all sources
 │   (uses this library) │  Nightly cron: 3 AM
 └──────────┬────────────┘
            │
            ▼
 ┌───────────────────────┐
 │   ~/memory/clusters/  │  Per-thread .md files with YAML headers
 │   *.md                │  ~9,000+ messages across 56+ threads
 └──────────┬────────────┘
            │
            ▼
 ┌───────────────────────┐
 │   build-clusters.js   │  Chunks all .md files → _index.json
 │   (cron: */30 min)    │  5,879 searchable chunks
 └──────────┬────────────┘
            │
            ▼
 ┌───────────────────────┐
 │   ClawText RAG        │  BM25 search on every prompt
 │   before_prompt_build │  Injects relevant context automatically
 └───────────────────────┘
```

---

## Installation

```bash
# Already bundled with OpenClaw workspace at:
~/.openclaw/workspace/skills/clawtext-ingest/

# Or clone separately:
git clone https://github.com/ragesaq/clawtext-ingest.git
cd clawtext-ingest
npm install
```

No separate Discord token setup required — uses the token stored by OpenClaw at  
`~/.openclaw/credentials/discord.token.json`.

---

## Making Ingestion Automatic

The recommended approach is the master ingest script with crons. No manual steps after initial setup.

### Step 1 — Configure Sources

Edit `~/.openclaw/workspace/scripts/ingest-all.mjs` and set your channels:

```javascript
const SOURCES = [
  // Forum channels (have threads — each thread becomes a cluster file)
  { id: '1476018965284261908', type: 'forum', name: 'rgcs-dev' },
  { id: '1475021817168134144', type: 'forum', name: 'ai-projects' },
  { id: '1477543809905721365', type: 'forum', name: 'moltmud-projects' },

  // Regular channels (ingested as single cluster file)
  { id: '1474997928056590339', type: 'channel', name: 'general' },
  { id: '1475019186563448852', type: 'channel', name: 'status' },
];
```

### Step 2 — Run Initial Ingest

```bash
node ~/.openclaw/workspace/scripts/ingest-all.mjs
```

This fetches everything available today. Output:

```
═══════════════════════════════════════════════
✅ ingest-all COMPLETE  (87.3s)
═══════════════════════════════════════════════
  Sources:   5
  Threads:   56
  Fetched:   9,105
  Imported:  8,968
  Skipped:   137 (dupes)
═══════════════════════════════════════════════
```

### Step 3 — Wire the Crons

```bash
crontab -e
```

Add:

```cron
# Rebuild RAG chunk index every 30 min (picks up session extractions)
*/30 * * * * /usr/bin/node ~/.openclaw/workspace/hooks/build-clusters.js \
  >> ~/memory/.cluster-rebuild.log 2>&1

# Full Discord re-ingest nightly at 3 AM (only new messages ingested)
0 3 * * * /usr/bin/node ~/.openclaw/workspace/scripts/ingest-all.mjs \
  >> ~/memory/.ingest-nightly.log 2>&1
```

That's it. Memory now grows automatically.

---

## Programmatic API

For custom ingestion workflows or agent-driven tasks:

```javascript
import { ClawTextIngest } from '~/.openclaw/workspace/skills/clawtext-ingest/src/index.js';

const ingest = new ClawTextIngest();

// Ingest an array of message objects
await ingest.fromJSON(messages, { checkDedupe: true });

console.log(`Imported: ${ingest.importedCount}`);
console.log(`Skipped:  ${ingest.skippedCount}`);
```

### fromJSON(messages, options)

```javascript
const messages = [
  {
    content: "The smoothing algorithm was updated in v1.2.324",
    author: "ragesaq",
    timestamp: "2026-03-05T06:00:00Z",
    channel: "rgcs-dev",
    source: "Discord"
  }
];

await ingest.fromJSON(messages, {
  checkDedupe: true,   // Skip already-seen messages (default: true)
  outputDir: '/home/lumadmin/memory/clusters',
  tags: ['rgcs', 'development']
});
```

### Message Object Fields

| Field | Required | Description |
|-------|----------|-------------|
| `content` | ✅ | Message text (empty messages skipped) |
| `author` | — | Author username |
| `timestamp` | — | ISO 8601 timestamp |
| `channel` | — | Source channel name |
| `source` | — | Source identifier |

### Deduplication

SHA1 hashes stored in `~/memory/.ingest_hashes.json`. The hash is computed from:
`sha1("${channelId}:${messageId}:${content}")`

This means:
- Same message re-ingested → skipped
- Same text in different channel → treated as new (safe)
- Editing a message → treated as new (safe)

To reset deduplication (force full re-ingest):
```bash
rm ~/memory/.ingest_hashes.json
```

---

## Direct Discord API Usage

For ingesting specific channels or threads on demand:

```javascript
import { readFileSync } from 'fs';
import { createHash } from 'crypto';

const token = JSON.parse(
  readFileSync('/home/lumadmin/.openclaw/credentials/discord.token.json')
).token;

async function fetchThread(channelId) {
  const messages = [];
  let before = null;
  
  while (true) {
    const url = `https://discord.com/api/v10/channels/${channelId}/messages?limit=100${before ? `&before=${before}` : ''}`;
    const res = await fetch(url, { headers: { Authorization: `Bot ${token}` } });
    
    if (res.status === 429) {
      const data = await res.json();
      await new Promise(r => setTimeout(r, data.retry_after * 1000 + 200));
      continue;
    }
    
    const batch = await res.json();
    if (!Array.isArray(batch) || batch.length === 0) break;
    messages.push(...batch);
    before = batch[batch.length - 1].id;
    if (batch.length < 100) break;
    await new Promise(r => setTimeout(r, 250));
  }
  
  // Filter text-only, no attachments
  return messages.filter(m => m.content?.trim() && !m.attachments?.length);
}
```

---

## Cluster File Format

Each ingested thread produces a `.md` file in `~/memory/clusters/`:

```markdown
---
source: discord
forum: "rgcs-dev"
thread_id: "1478240533557022730"
thread_name: "RGCS Smoothing Development v1.2.324"
message_count: 672
ingested_at: "2026-03-05T07:19:00.000Z"
tags: [discord, rgcs-dev]
---

# RGCS Smoothing Development v1.2.324

**[2026-03-03T04:00] ragesaq:** Starting fresh thread for v1.2.324...

**[2026-03-03T04:01] lumbot:** Confirmed — smoothing coefficients updated...
```

Filenames follow the pattern:
```
{forum-name}-{thread-id}-{slug}.md
```

---

## Logs

| File | Contents |
|------|---------|
| `~/memory/.ingest-nightly.log` | Nightly ingest run output |
| `~/memory/.cluster-rebuild.log` | 30-min cluster rebuild output |
| `~/memory/.ingest-log.jsonl` | Structured JSON log of every ingest run |
| `~/memory/.ingest_hashes.json` | SHA1 dedup hash store |

Check nightly log:
```bash
tail -50 ~/memory/.ingest-nightly.log
```

Check last ingest run stats:
```bash
tail -1 ~/memory/.ingest-log.jsonl | python3 -m json.tool
```

---

## Current Coverage (March 5, 2026)

| Forum / Channel | Threads | Messages |
|----------------|---------|----------|
| `#rgcs-dev` | 6 | ~2,470 |
| `#ai-projects` | 35 | ~4,300 |
| `#moltmud-projects` | 15 | ~1,400 |
| `#general` | 2 | ~4 |
| `#status` | — (direct) | ~836 |
| **Total** | **56+** | **~9,000+** |

RAG index: **59 cluster files, 5,879 chunks**

---

## Troubleshooting

**Rate limited during ingest:**
The ingest script handles 429s automatically with backoff. If you see warnings, they're recovered — check the final summary for actual imported count.

**New thread not appearing in RAG:**
1. Check the thread is in a configured forum channel
2. Run `ingest-all.mjs` manually — new threads in existing forums are picked up automatically
3. Run `build-clusters.js` to refresh the index

**Dedup blocking messages it shouldn't:**
If you believe messages are being incorrectly skipped, check:
```bash
cat ~/memory/.ingest_hashes.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'{len(d)} hashes stored')"
```

To force a clean re-ingest of everything:
```bash
rm ~/memory/.ingest_hashes.json
node ~/.openclaw/workspace/scripts/ingest-all.mjs
```

**`build-clusters.js` not found:**
```bash
ls ~/.openclaw/workspace/hooks/build-clusters.js
# If missing:
# Copy from clawtext repo or recreate — see ClawText README
```

---

## Related

- **[ClawText](../../extensions/clawtext/)** — RAG plugin that consumes the cluster index
- **[ingest-all.mjs](../scripts/ingest-all.mjs)** — Master ingest script
- **[build-clusters.js](../hooks/build-clusters.js)** — Cluster index builder

---

## License

MIT
