# ClawText Ingest

Version: 1.0.0

ClawText Ingest is a lightweight, agent-friendly ingestion tool that converts multi-source data (files, web pages, chat exports, APIs, and raw text) into structured memory entries for the ClawText RAG layer. It automatically generates YAML frontmatter (date, project, type, entities, source, keywords) and writes memory files to your OpenClaw workspace so ClawText can index and inject them into sessions.

Why this exists
- Reduce friction when adding new knowledge to memory.
- Make ingestion agent-executable and repeatable.
- Produce consistent metadata so clustering and BM25 scoring work better.

Key features
- Multi-source ingestion: files (glob), URLs, JSON (chat/API), raw text
- Automatic YAML frontmatter generation
- Batch processing with per-item error handling
- Writes to your OpenClaw memory directory (~/.openclaw/workspace/memory)
- Trigger cluster rebuild marker so ClawText will re-index
- Simple, small codebase intended for easy extension and agent automation

Quick Install (local)

```bash
# from workspace root
cd ~/.openclaw/workspace/skills/clawtext-ingest
npm install
```

Quick start (example)

```javascript
import ClawTextIngest from './src/index.js';

const ingest = new ClawTextIngest();

// Ingest doc files
await ingest.fromFiles(['docs/**/*.md', 'notes/*.txt'], { project: 'docs', type: 'fact' });

// Ingest a website (simple fetch)
await ingest.fromUrls(['https://example.com/article'], { project: 'external', type: 'fact' });

// Ingest a chat export (array of messages)
await ingest.fromJSON(chatExportArray, { project: 'team', type: 'decision' }, {
  keyMap: { contentKey: 'message', dateKey: 'timestamp', authorKey: 'user' }
});

// Run batch ingestion
await ingest.ingestAll([
  { type: 'files', data: ['docs/**/*.md'], metadata: { project: 'docs' } },
  { type: 'json', data: chatExportArray, metadata: { project: 'team' }, options: { keyMap: { contentKey: 'message', dateKey: 'timestamp', authorKey: 'user' } } }
]);

// Signal clusters to rebuild (delete stale cluster files so ClawText rebuilds)
await ingest.rebuildClusters();
```

API reference

- new ClawTextIngest(memoryDir?)
  - memoryDir defaults to `~/.openclaw/workspace/memory`
- fromFiles(patterns, metadata)
  - Glob patterns or array. Returns { imported, errors }
- fromUrls(urls, metadata)
  - Fetches URL content and writes to memory files. Returns { imported, errors }
- fromJSON(data, metadata, options)
  - Accepts chat exports or API arrays. options.keyMap maps content/date/author keys. Returns { imported, errors }
- fromText(text, metadata)
  - Writes raw text with frontmatter to a memory file. Returns { imported, errors }
- ingestAll(sources)
  - Batch multiple source objects: { type, data, metadata, options }
  - Returns { totalImported, results, errors }
- rebuildClusters()
  - Marks clusters for rebuild by removing cluster jsons. Returns { success, message }
- getReport()
  - Returns ingestion summary and errors

Best practices
- Use YAML frontmatter where possible to capture: date, project, type, entities, keywords
- Prefer small daily memory files rather than one giant file — improves BM25 relevance
- Run `rebuildClusters()` after large ingestions (backfills) to refresh ClawText indexes
- Add `entities` for agent names and canonical project fields for better project-weighting

Example: Ingesting a Discord export

1. Export channel to JSON or CSV via your preferred exporter.
2. Map keys: content => message, timestamp => timestamp, author => user
3. Run fromJSON with the keyMap option and a transform function if you need to clean content.

Extensibility ideas
- Add deduplication (hash-based or canonical ID) to avoid re-ingesting the same content
- Add HTML cleaning and readability parsing (readability, strip scripts) for Web ingestion
- Add connectors for Slack, Notion, Confluence, GitHub issues/pull requests
- Add embedding generation step (optional) so new memories are semantically indexed immediately

Testing

Run the included tests:

```
npm test
```

License

MIT — see LICENSE file for details.

Contributing

PRs welcome. Keep changes small and include tests for parsing/transform cases. Follow the existing code style (ESM modules, Node 18+).

Contact

If you want me to push this README and create a release/tag, say "push README + tag v1.0.0" and I will commit and push it to origin/main and create the tag locally. Alternatively, say "just commit README" and I'll commit and push without tagging.
