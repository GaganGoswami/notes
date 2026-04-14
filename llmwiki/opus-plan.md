## Plan: LLM Wiki Second Brain System

**TL;DR:** Build a local Obsidian vault + VS Code workspace maintained by GitHub Copilot as a disciplined wiki agent. Raw sources (articles, YouTube transcripts, GitHub data, web clips) are ingested and compiled into a persistent, interlinked wiki with confidence-scored knowledge, tiered memory, a typed knowledge graph, and hybrid search via QMD MCP. Python scripts handle fetching/automation; Copilot does the knowledge work.

**Core constraint:** Copilot is an interactive agent, not a background service. Hooks and proactive features need external scripts. The plan front-loads the interactive wiki pattern (where Copilot reaches parity with Claude Code) and defers autonomous features.

---

### Phase 1: Foundation — Vault Structure & Schema *(Low complexity)*

Create the directory skeleton, identity files, and the schema that turns Copilot into a wiki maintainer.

1. **Create vault directory structure** — `_soul/`, `_index/`, `_templates/`, `_scripts/`, `raw/` (articles, papers, transcripts, github, books, data, assets), `wiki/` (entities/people/orgs/projects/tools, concepts, sources, syntheses, decisions, maps), `memory/` (working, episodic, semantic, procedural), `daily/YYYY/MM/`, `output/`
2. **Create `_soul/SOUL.md`** — Agent personality, tone (precise, factual), epistemic humility rules, boundaries
3. **Create `_soul/USER.md`** — Your profile: name, role, timezone, accounts, preferences (you fill this in)
4. **Create `_soul/SCHEMA.md`** — Entity types, relationship types (uses, depends-on, contradicts, supersedes, etc.), frontmatter field spec for all page types with confidence scoring, timestamps, source tracking
5. **Create `.github/copilot-instructions.md`** — The schema layer: directory conventions, full 10-step ingest workflow, confidence rules (new claim 0.5–0.7, 2+ sources → 0.7–0.9, decay at 90 days), linking rules (wikilinks, no orphans, bidirectional), lint checklist, consolidation schedule, forbidden actions
6. **Create seed navigation files** — `_index/index.md` (master catalog), `_index/log.md` (ops log), `_index/inbox.md` (triage queue), `_index/graph-schema.md` (relationship registry)
7. **Create page templates** in `_templates/` — entity, concept, source-summary, daily-log, comparison, crystallization — each with full frontmatter skeleton

---

### Phase 2: Ingest Pipeline — Source Processing Tools *(Medium complexity)*

Python scripts to fetch/convert sources into `raw/`; reusable `.prompt.md` files for Copilot to process them into the wiki.

1. **`_scripts/requirements.txt`** — `youtube-transcript-api`, `markdownify`, `beautifulsoup4`, `requests`, `readability-lxml`
2. **`_scripts/yt_transcript.py`** — YouTube URL → markdown in `raw/transcripts/` with frontmatter. Uses `youtube-transcript-api`
3. **`_scripts/web_clip.py`** — URL → article markdown in `raw/articles/` via `readability-lxml` + `markdownify`. Downloads images to `raw/assets/`
4. **`_scripts/github_fetch.py`** — Repo/issue/PR → markdown snapshot in `raw/github/` via GitHub API or `gh` CLI
5. **Reusable prompts** — `_prompts/ingest-source.prompt.md`, `query-wiki.prompt.md`, `lint-wiki.prompt.md`, `crystallize.prompt.md` for standardized Copilot workflows

---

### Phase 3: Search — QMD MCP Integration *(Low-Medium complexity)*

Enable hybrid search (BM25 + vector + LLM reranking) so Copilot can search beyond `index.md` at scale.

1. **Install QMD** — `npm install -g @tobilu/qmd` (needs Node ≥ 22, auto-downloads ~2GB of local models)
2. **Configure `.vscode/mcp.json`** — QMD as MCP server with tools: `query`, `get`, `multi_get`, `status`
3. **Initial indexing** — `qmd index .` from vault root
4. **Update `copilot-instructions.md`** — "prefer QMD `query` tool for searching; fall back to `index.md` for browsing"
5. **`_scripts/reindex.sh`** — convenience script for re-indexing after bulk ingests

---

### Phase 4: Memory Lifecycle — Confidence, Decay & Consolidation *(Medium complexity)*

Implement v2 knowledge lifecycle: confidence scoring, supersession, retention decay, tier promotion.

1. **`_scripts/consolidate.py`** — Promotes working → episodic (daily) and flags episodic patterns for semantic promotion
2. **`_scripts/decay.py`** — Ebbinghaus-inspired confidence decay. Pages not reinforced in 90+ days lose 0.1 confidence; marked `stale` below 0.3. Retention classes: architecture = slow-decay (180d), transient bugs = fast-decay (30d)
3. **`_scripts/lint.py`** — Detects orphan pages, broken wikilinks, missing frontmatter, unresolved contradictions, entities mentioned without own page
4. **`_scripts/graph_query.py`** — Parses `related_entities` frontmatter → builds in-memory graph → answers "what depends on X?", "impact of changing X?" via BFS/DFS

---

### Phase 5: Automation — Make, Git Hooks & Session Prompts *(Medium complexity)*

Bridge the gap from Claude Code's lifecycle hooks with manual prompts + automated scripts.

1. **`Makefile`** — targets: `ingest URL=...`, `lint`, `decay`, `consolidate`, `reindex`, `daily`
2. **`.githooks/pre-commit`** — auto-reindex changed markdown, validate frontmatter
3. **Manual "hooks" via `.prompt.md`** — `session-start.prompt.md` (loads MEMORY.md + daily log + inbox + recent log entries) and `session-end.prompt.md` (captures decisions, insights, action items → daily log + MEMORY.md)
4. **Optional: launchd plist** for daily scheduled consolidation + decay + reindex

---

### Phase 6: Knowledge Graph & Entity System *(Medium-High complexity)*

Typed knowledge graph with dual representation: wikilinks for Obsidian graph view + structured frontmatter for programmatic queries.

1. **Populate `graph-schema.md`** with full relationship registry (types, inverses, weights)
2. **Update ingest workflow** — entity extraction → create/update entity pages → add relationships to both pages' frontmatter AND inline `## Relationships` section
3. **`_scripts/graph_export.py`** — Export full graph as JSON + optional DOT/Graphviz
4. **Dashboard page** (`wiki/maps/dashboard.md`) with Dataview queries: stale entities, low-confidence claims, most-connected nodes

---

### Phase 7 (Optional): Proactive System & Chat Interface *(High complexity)*

Builds OUTSIDE Copilot using direct LLM API calls. Only if you want autonomous monitoring.

1. **Standalone heartbeat** — Python + Anthropic/OpenAI API, reads wiki, checks GitHub notifications, sends summaries every 30 min via cron (~$0.05/run)
2. **Chat interface** — Slack/Discord bot backed by LLM API + QMD search over the wiki

---

### Relevant Files

| Path | Purpose |
|------|---------|
| `.github/copilot-instructions.md` | Schema layer — makes Copilot a wiki maintainer |
| `.vscode/mcp.json` | QMD search MCP server config |
| `_soul/SOUL.md`, `USER.md`, `SCHEMA.md` | Agent identity, owner profile, domain ontology |
| `_index/index.md`, `log.md`, `inbox.md`, `graph-schema.md` | Navigation & meta files |
| `_templates/*.md` | Page templates with frontmatter skeletons |
| `_scripts/*.py` | Ingest, lint, decay, consolidation, graph tools |
| `_prompts/*.prompt.md` | Reusable Copilot workflow prompts |
| `Makefile` | Convenience automation targets |
| `.githooks/pre-commit` | Auto-reindex on commit |

### Verification

1. Vault opens correctly in Obsidian — graph view shows structure, templates have valid YAML
2. `python _scripts/yt_transcript.py <URL>` and `web_clip.py <URL>` produce markdown in `raw/`
3. QMD MCP tools appear in Copilot's tool list; `qmd query "test"` returns results
4. `make lint` produces a health report; `make decay` modifies confidence scores correctly
5. After ingesting 3+ sources, Obsidian graph shows entity relationships; Dataview queries work
6. Session start/end prompts successfully load and save context

### Decisions

- **Interactive agent:** VS Code + GitHub Copilot (near-parity for wiki ops; proactive features need direct API)
- **Search:** QMD over custom FastEmbed — turnkey hybrid search with MCP, avoids building your own
- **Browsing:** Obsidian (graph view, Dataview, Marp). Same vault, two interfaces
- **Naming:** Lowercase kebab-case, wikilinks, no spaces
- **Scope in:** Vault structure, schema, ingest (YT/web/GitHub), search, memory lifecycle, knowledge graph, automation
- **Scope out:** Multi-user, cloud deploy, mobile, Gmail/Slack/Calendar integration (deferred to Phase 7)

### Further Considerations

1. **Obsidian plugins to install?** Recommend: Dataview, Templater, Graph Analysis, Marp Slides, Web Clipper. Want me to specify exact plugin config?
2. **Frontmatter complexity** — The full schema has ~20 fields. We could start with a minimal set (title, type, confidence, created, updated, tags, sources) and add relationship fields in Phase 6. Simpler start vs. consistency from day one?
3. **Git repo initialization** — Should the vault be a fresh git repo, or live inside an existing project? Standalone repo is cleaner for the knowledge base use case.