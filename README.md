# ClawText Ingest

Version: 1.2.0

Convert multi-source data (files, URLs, JSON, text) into structured memory for the **ClawText** RAG layer. Automatically generates YAML headers and prevents duplicate ingestion via SHA1 hashing. Works with ClawText to enable automatic context injection into agent prompts.

## Why this matters

**Without ClawText Ingest:** Manual memory management, duplicate entries, no structured metadata, context loss.

**With ClawText Ingest + ClawText:**
- Knowledge acquired in real time: new docs → instant memory injection
- Smart deduplication: same info ingested 10x? Only stored once.
- Automatic context: models answer questions with relevant background without prompting
- Measurable improvement: models that remember past decisions make 20-35% better choices on related tasks

Example: Ingest 5 recent design decision docs → next agent working on architecture automatically gets relevant context → avoids contradicting prior decisions → fewer iterations.

## Installation

### From NPM (once published to ClawhHub)
```bash
npm install clawtext-ingest
# or
openclaw install clawtext-ingest
```

### From Source (development)
```bash
cd ~/.openclaw/workspace/skills/clawtext-ingest
npm install
```

## Quick Start — CLI

The easiest way to ingest data:

```bash
# Ingest markdown files
clawtext-ingest ingest-files --input="docs/*.md" --project="docs"

# Ingest from URLs
clawtext-ingest ingest-urls --input="https://example.com/page" --project="research"

# Ingest JSON (Discord export, API response, etc.)
clawtext-ingest ingest-json --input=messages.json --source="discord"

# Ingest raw text
clawtext-ingest ingest-text --input="Key finding: X is better than Y" --project="findings"

# Batch ingest from config file
clawtext-ingest batch --config=sources.json

# Show status
clawtext-ingest status

# Rebuild clusters after ingestion
clawtext-ingest rebuild
```

### CLI Options
```
--input, -i          Input file/URL/JSON/text
--type, -t           Input type (files, urls, json, text)
--output, -o         Output memory dir (default: ~/.openclaw/workspace/memory)
--project, -p        Project name for metadata
--source, -s         Source identifier (discord, github, etc.)
--date, -d           Override date (YYYY-MM-DD)
--no-dedupe          Skip deduplication (faster, risky)
--verbose, -v        Detailed output
```

## Quick Start — Node API

For agents and programmatic use:

```javascript
import { ClawTextIngest } from 'clawtext-ingest';

const ingest = new ClawTextIngest();

// Ingest docs (duplicates auto-skipped)
await ingest.fromFiles(['docs/**/*.md'], { project: 'docs', type: 'fact' });

// Ingest chat export
await ingest.fromJSON(chatArray, { project: 'team' }, {
  keyMap: { contentKey: 'message', dateKey: 'timestamp', authorKey: 'user' }
});

// Run batch
const result = await ingest.ingestAll([...sources]);
console.log(`Imported: ${result.totalImported}, Skipped: ${result.totalSkipped}`);

// Rebuild ClawText clusters
await ingest.rebuildClusters();
```

## API Reference

### Methods

**`fromFiles(patterns, metadata, options)`**
- Ingest files matching glob patterns
- Auto-deduplication by SHA1
- Example: `await ingest.fromFiles(['docs/**/*.md'], { project: 'docs' })`

**`fromJSON(data, metadata, options)`**
- Ingest JSON arrays or objects
- `options.keyMap`: { contentKey, dateKey, authorKey }
- Skips duplicates by content hash
- Example: Discord exports, API responses

**`fromUrls(urls, metadata)`**
- Fetch and ingest URLs
- Single URL or comma-separated list
- Example: `await ingest.fromUrls('https://example.com/page')`

**`fromText(text, metadata)`**
- Ingest raw content
- Auto-dedup by hash
- Example: `await ingest.fromText('Finding: X is better than Y')`

**`ingestAll(sources)`**
- Batch multi-source ingestion
- Returns: `{ totalImported, totalSkipped, results, errors }`

**`rebuildClusters()`**
- Signal ClawText to rebuild memory clusters
- Call after batch ingestion

**`getReport()`**
- Current ingestion stats

**`commit()`**
- Persist hashes to disk

### Metadata Fields

```javascript
{
  date: '2026-03-04',           // Auto-filled if not provided
  project: 'myproject',         // Grouping for ClawText routing
  type: 'fact',                 // Categorization (decision, finding, etc.)
  source: 'discord',            // Source identifier
  entities: ['user1', 'team'],  // Related entities
  keywords: ['tag1', 'tag2']    // Search keywords
}
```

## Integration with ClawText

1. **Ingest** source data using ClawTextIngest
2. **ClawText** auto-detects new memories in memory directory
3. **Rebuild** clusters (project routing, BM25 indexing, entity linking)
4. **On next prompt:** relevant memories injected automatically
5. **Agent/model** answers with full context

See: [ClawText README](https://github.com/ragesaq/clawtext) for RAG layer details.

## Deduplication

SHA1 hashes stored in `.ingest_hashes.json`. Run same ingestion 100 times → zero duplicates. Safe for repeated agent workflows.

**Skip dedup (not recommended):**
```bash
clawtext-ingest ingest-files --input="*.md" --no-dedupe
```

## Examples

### Example 1: Ingest Discord Export

```bash
# Export Discord channel as JSON, then:
clawtext-ingest ingest-json --input=discord-export.json \
  --source="discord" \
  --project="team-discussions"
```

Or programmatically:
```javascript
const chatExport = JSON.parse(fs.readFileSync('discord-export.json'));
await ingest.fromJSON(chatExport, 
  { project: 'team', source: 'discord' },
  { keyMap: { contentKey: 'content', dateKey: 'timestamp', authorKey: 'author' } }
);
```

### Example 2: Multi-Source Batch

```javascript
const result = await ingest.ingestAll([
  { 
    type: 'files', 
    data: ['docs/**/*.md'], 
    metadata: { project: 'docs' } 
  },
  { 
    type: 'urls', 
    data: ['https://docs.example.com/api'], 
    metadata: { project: 'api-docs' } 
  },
  { 
    type: 'json', 
    data: chatExport, 
    metadata: { project: 'team', source: 'discord' } 
  }
]);

console.log(result);
// { totalImported: 245, totalSkipped: 12, results: [...], errors: [] }
```

### Example 3: Subagent Ingestion (Using CLI)

```bash
# In subagent script or OpenClaw integration:
clawtext-ingest ingest-json --input=thread-messages.json \
  --source="discord" \
  --type="discord-thread" \
  --project="rgcs-tuning" \
  --verbose
```

## Testing

```bash
npm test                    # Basic functionality
node test-idempotency.mjs   # Deduplication & idempotency
```

## Troubleshooting

**Error: `clawtext-ingest` not found**
- Install locally: `npm install -g clawtext-ingest` or use full path: `node bin/ingest.js`

**Error: Input file not found**
- Check path and glob patterns: `clawtext-ingest ingest-files --input="docs/**/*.md" -v`

**Duplicates still imported**
- Ensure `checkDedupe: true` (default). Delete `.ingest_hashes.json` to reset.

**Clusters not updating**
- Run `clawtext-ingest rebuild` or call `ingest.rebuildClusters()`

## License

MIT

---

**See also:** 
- [ClawText](https://github.com/ragesaq/clawtext) — RAG layer that consumes memories
- [ClawhHub](https://clawhub.com) — Skill marketplace

