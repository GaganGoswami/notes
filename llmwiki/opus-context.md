Now I have full context from all three files. Here's the research report:

---

# LLM Wiki Second Brain: Architecture Research Report

## 1. Optimal Directory Structure (500+ pages)

Based on the three-layer architecture from v1 (raw sources → wiki → schema) combined with v2's memory tiers, consolidation pipeline, and the secondbrain.md's agent memory layer:

```
vault/
├── .obsidian/                    # Obsidian config (git-tracked selectively)
│   └── plugins/
├── .vscode/                      # VS Code workspace settings
│   └── settings.json
├── .github/
│   └── copilot-instructions.md   # Schema layer — the "AGENTS.md" equivalent
│
├── _soul/                        # Agent identity & system files
│   ├── SOUL.md                   # Agent personality, values, tone, boundaries
│   ├── USER.md                   # Owner profile, accounts, preferences, timezone
│   └── SCHEMA.md                 # Entity types, relationship types, frontmatter spec
│
├── _index/                       # Navigation & meta
│   ├── index.md                  # Master page catalog (category → page → one-liner)
│   ├── log.md                    # Append-only chronological ops log
│   ├── graph-schema.md           # Typed relationship registry
│   └── inbox.md                  # Triage queue for unprocessed observations
│
├── _templates/                   # Obsidian Templater / manual templates
│   ├── entity.md
│   ├── concept.md
│   ├── source-summary.md
│   ├── daily-log.md
│   ├── comparison.md
│   └── crystallization.md
│
├── _scripts/                     # Tooling (search, lint, ingest helpers)
│   ├── search.py
│   ├── lint.py
│   ├── ingest.py
│   └── consolidate.py
│
├── raw/                          # Layer 1: Immutable raw sources
│   ├── articles/                 # Web clipper output (Obsidian Web Clipper)
│   ├── papers/                   # PDFs + companion .md extracts
│   ├── transcripts/              # YouTube, podcast, meeting transcripts
│   ├── github/                   # README snapshots, issue dumps, code snippets
│   ├── books/                    # Chapter-by-chapter clippings
│   ├── data/                     # CSV, JSON, structured data files
│   └── assets/                   # Images, diagrams, screenshots
│
├── memory/                       # Layer 2a: Memory tiers (v2 consolidation pipeline)
│   ├── working/                  # Tier 0 — raw observations, not yet processed
│   │   └── YYYY-MM-DD.md         # Daily working memory / session capture
│   ├── episodic/                 # Tier 1 — session summaries, compressed
│   │   └── YYYY-MM-DD.md         # Consolidated daily digests
│   ├── semantic/                 # Tier 2 — cross-session established facts
│   │   └── MEMORY.md             # Key decisions, lessons, active projects
│   └── procedural/              # Tier 3 — workflows, patterns, SOPs
│       └── *.md                  # Extracted repeating patterns
│
├── wiki/                         # Layer 2b: The compiled wiki
│   ├── entities/                 # Entity pages (people, orgs, projects, tools)
│   │   ├── people/
│   │   ├── organizations/
│   │   ├── projects/
│   │   ├── tools/                # Libraries, frameworks, services
│   │   └── places/
│   ├── concepts/                 # Concept pages (ideas, theories, patterns)
│   ├── sources/                  # Source summary pages (1:1 with raw/ items)
│   ├── syntheses/                # Cross-cutting analyses, comparisons, theses
│   ├── decisions/                # Decision records (ADR-style)
│   └── maps/                     # MOCs (Maps of Content) — thematic indexes
│
├── daily/                        # Daily logs / journal
│   └── YYYY/
│       └── MM/
│           └── YYYY-MM-DD.md
│
└── output/                       # Generated artifacts (slide decks, exports)
    ├── decks/                    # Marp presentations
    ├── reports/                  # Long-form exports
    └── data/                     # JSON/CSV structured exports
```

### Key design decisions

- **Underscore-prefixed dirs** (`_soul/`, `_index/`, `_templates/`, `_scripts/`) sort to the top in both Obsidian and VS Code file explorers and signal "infrastructure, not content."
- **`raw/` is immutable.** The LLM reads from it but never writes to it. Source of truth.
- **`wiki/` is LLM-owned.** The LLM creates, updates, and maintains all pages. Humans read and guide.
- **`memory/` is the v2 consolidation pipeline.** Working → Episodic → Semantic → Procedural. Separate from the wiki because memory is session-oriented while the wiki is topic-oriented.
- **`daily/` is nested by year/month** to prevent a single folder from exceeding OS/editor limits at 500+ pages scale.
- **`output/` is ephemeral.** Generated artifacts that can be recreated from the wiki.

---

## 2. Memory Tiers — Directory Mapping

| Tier | Dir | Lifespan | Content | Promotion trigger |
|------|-----|----------|---------|-------------------|
| **Working** | `memory/working/` | Hours | Raw observations from current session. Unprocessed notes, questions, hunches. | Session end → compress into episodic |
| **Episodic** | `memory/episodic/` | Days–Weeks | Session summaries. "On 2026-04-14 I researched X and found Y." Timestamped narratives. | Pattern seen ≥3 times → extract to semantic |
| **Semantic** | `memory/semantic/` (+ `wiki/`) | Months–Permanent | Established facts. "Project X uses Redis." Cross-session, decontextualized. | Repeated application → extract to procedural |
| **Procedural** | `memory/procedural/` | Permanent | Workflows, SOPs, patterns. "When ingesting a YouTube transcript, always do X then Y." | Evolves through use, rarely demoted |

The `wiki/` directory itself is primarily **semantic memory** — it represents compiled, cross-referenced, established knowledge. The `memory/` hierarchy tracks the *pipeline* that feeds into it.

**MEMORY.md** (at `memory/semantic/MEMORY.md`) serves as the high-priority semantic index — the "load this first" file containing active projects, recent key decisions, and lessons learned. Analogous to the secondbrain.md pattern.

---

## 3. Special Files — Purposes & Locations

| File | Location | Purpose |
|------|----------|---------|
| **SOUL.md** | `_soul/SOUL.md` | Agent identity. Personality, tone, values, boundaries, what it should/shouldn't do. Loaded at every session start. |
| **USER.md** | `_soul/USER.md` | Owner profile. Name, role, timezone, accounts, preferences, communication style. Enables personalization. |
| **SCHEMA.md** | `_soul/SCHEMA.md` | Domain ontology. Lists all entity types, relationship types, frontmatter fields, naming conventions. The "data dictionary." |
| **MEMORY.md** | `memory/semantic/MEMORY.md` | Hot semantic memory. Key active facts, recent decisions, current projects. First file loaded for context injection. |
| **index.md** | `_index/index.md` | Master catalog. Every wiki page listed with link + one-liner + category. LLM reads this first to find pages. |
| **log.md** | `_index/log.md` | Chronological ops log. Append-only. Entries like `## [2026-04-14] ingest \| Article Title`. Parseable with grep. |
| **inbox.md** | `_index/inbox.md` | Triage queue. Unprocessed observations/sources waiting for integration. Cleared during ingest. |
| **graph-schema.md** | `_index/graph-schema.md` | Registry of valid relationship types and their semantics (`uses`, `depends-on`, `contradicts`, `supersedes`, etc.). |
| **copilot-instructions.md** | `.github/copilot-instructions.md` | Schema layer for GitHub Copilot. Rules, conventions, workflows. Makes Copilot a disciplined wiki maintainer. |

---

## 4. Frontmatter Schema

Every wiki page should carry YAML frontmatter enabling confidence scoring, temporal tracking, source provenance, and Dataview queries:

```yaml
---
# Identity
title: "Redis Caching Architecture"
type: concept                       # entity | concept | source | synthesis | decision | map | daily
aliases: ["Redis cache", "caching layer"]

# Taxonomy
domain: infrastructure              # top-level domain
tags: [redis, caching, performance, backend]

# Confidence & lifecycle (v2)
confidence: 0.85                    # 0.0–1.0, decays with time, strengthens with reinforcement
confidence_basis: 3                 # number of supporting sources
status: active                      # active | stale | superseded | draft
superseded_by:                      # wikilink to newer page if superseded
retention_class: slow-decay         # slow-decay | medium-decay | fast-decay

# Timestamps
created: 2026-03-01
updated: 2026-04-14
last_accessed: 2026-04-14
last_reinforced: 2026-04-10         # last time a new source confirmed this

# Source tracking
sources:                            # list of raw source refs
  - "[[raw/articles/redis-best-practices]]"
  - "[[raw/transcripts/systemdesign-ep42]]"
source_count: 2

# Relationships (for graph queries)
related_entities:
  - entity: "[[wiki/entities/projects/auth-migration]]"
    relation: uses
    confidence: 0.9
  - entity: "[[wiki/entities/tools/memcached]]"
    relation: contradicts
    confidence: 0.7

# Authorship
author: llm                         # llm | human | hybrid
reviewed: false                     # human review flag
quality_score: 0.8                  # LLM self-assessment 0.0–1.0
---
```

### Dataview query examples this enables

```dataview
TABLE confidence, source_count, updated
FROM "wiki/concepts"
WHERE confidence < 0.5
SORT confidence ASC
```

```dataview
TABLE title, status, superseded_by
FROM "wiki"
WHERE status = "stale"
SORT updated ASC
```

```dataview
TABLE title, confidence, last_reinforced
FROM "wiki"
WHERE date(now) - last_reinforced > dur(60 days)
SORT last_reinforced ASC
```

---

## 5. Knowledge Graph in Markdown

The goal: represent a typed knowledge graph that's compatible with Obsidian's graph view (which uses `[[wikilinks]]`) AND programmatically queryable.

### Strategy: dual representation

**1. Inline wikilinks for Obsidian graph view.** Every relationship appears as a `[[wikilink]]` in the page body so Obsidian draws the edge:

```markdown
## Relationships

- **uses**: [[Redis]], [[PostgreSQL]]
- **owned-by**: [[Sarah Chen]]
- **depends-on**: [[Auth Service]], [[User API]]
- **supersedes**: [[Legacy Cache Layer]]
```

Obsidian's graph view will render all of these as edges. The `**type**:` prefix is human-readable but invisible to the graph — all edges look the same in Obsidian's default graph view.

**2. Frontmatter `related_entities` for programmatic queries.** The structured YAML (shown above) carries the typed relationship, confidence, and directionality. Scripts and Dataview can query this:

```dataview
TABLE
  flatten(related_entities) as rel
FROM "wiki/entities"
WHERE any(related_entities, (r) => r.relation = "depends-on")
```

**3. Naming convention for entities.** Use the canonical name as the filename: `redis.md`, `sarah-chen.md`, `auth-migration.md`. Place in typed subdirectories (`entities/tools/`, `entities/people/`, `entities/projects/`). This lets you infer the entity type from the path without opening the file.

**4. Relationship registry** (`_index/graph-schema.md`):

```markdown
# Relationship Types

| Relation      | Inverse          | Weight | Example                    |
|---------------|------------------|--------|----------------------------|
| uses          | used-by          | 0.7    | Project → Library          |
| depends-on    | dependency-of    | 0.9    | Service → Service          |
| owned-by      | owns             | 0.5    | Project → Person           |
| contradicts   | contradicted-by  | 1.0    | Claim → Claim              |
| supersedes    | superseded-by    | 1.0    | NewPage → OldPage          |
| caused        | caused-by        | 0.8    | Event → Event              |
| instance-of   | has-instance     | 0.3    | Tool → Category            |
```

---

## 6. Source Type Organization

Within `raw/`, sources are organized by **format type** (not topic) because format determines ingest workflow:

| Directory | Source type | Ingest notes |
|-----------|------------|--------------|
| `raw/articles/` | Web articles (Obsidian Web Clipper) | Markdown ready. May need image download (`Ctrl+Shift+D`). |
| `raw/papers/` | Academic PDFs | Store PDF + companion `.md` extracted by LLM. Reference both. |
| `raw/transcripts/` | YouTube, podcasts, meetings | Plain text. Often noisy — LLM should extract signal. |
| `raw/github/` | READMEs, issues, PRs, code snippets | Snapshot with URL + date. Code blocks preserved. |
| `raw/books/` | Book chapters, highlights | One file per chapter or per book. Sequential processing. |
| `raw/data/` | CSV, JSON, structured datasets | LLM generates summary + key findings page. Raw preserved. |
| `raw/assets/` | Images, diagrams, screenshots | Referenced by other raw files and wiki pages. Not ingested standalone. |

Each raw source file should have minimal frontmatter:

```yaml
---
source_type: article
source_url: "https://example.com/article"
source_date: 2026-04-10
ingested: false            # flipped to true after processing
ingested_date:
clipped_date: 2026-04-14
---
```

---

## 3. Naming Convention & Linking Strategy

### File naming

| Rule | Convention | Example |
|------|-----------|---------|
| **Lowercase kebab-case** | All filenames | `redis-caching.md`, `sarah-chen.md` |
| **No spaces** | Use hyphens | `auth-migration.md` not `Auth Migration.md` |
| **Canonical name = filename** | The most common short name | `react.md` not `react-javascript-library.md` |
| **Date-prefixed for temporal** | `YYYY-MM-DD` prefix for daily/episodic | `2026-04-14.md` |
| **Source summaries mirror raw** | Same name, different dir | `raw/articles/redis-best-practices.md` → `wiki/sources/redis-best-practices.md` |

### Linking strategy

- **Use wikilinks** (`[[page-name]]`) as the primary link format. Works in Obsidian graph view AND is greppable.
- **Use aliases** in frontmatter for discoverability: `aliases: ["Redis", "redis cache"]` so `[[Redis]]` resolves to `redis.md`.
- **Use display text** for relationship context: `[[redis|Redis caching layer]]`.
- **Heading links** for precision: `[[redis-caching#Performance tuning]]`.
- **Every page must have ≥1 inbound link.** Orphan pages are lint violations.
- **Every entity relationship mentioned in prose must also appear in frontmatter `related_entities`.** This is the "dual representation" rule.
- **Programmatic search** works because: (a) filenames are kebab-case and greppable, (b) frontmatter is structured YAML parseable by any language, (c) wikilinks use a consistent `[[pattern]]` regex-matchable as `\[\[([^\]]+)\]\]`.

---

## 4. `.github/copilot-instructions.md` — Schema Layer

This file turns GitHub Copilot into a disciplined wiki maintainer. It should contain:

```markdown
# Wiki Maintainer Instructions

You are the maintainer of a personal knowledge wiki. You never write casually.
Every edit must follow the conventions below.

## Identity

- Role: Knowledge wiki maintainer
- Tone: Precise, concise, factual
- Never fabricate information. If unsure, say so and set confidence < 0.5.

## Directory Conventions

- `raw/` — Immutable source documents. NEVER modify.
- `wiki/` — LLM-maintained compiled knowledge. You own this.
- `memory/` — Tiered memory pipeline. You own this.
- `_index/` — Navigation files. Update on every ingest.
- `_soul/` — System identity files. Read but rarely modify.
- `daily/` — Daily logs. Append-only.

## Page Types & Templates

- **entity**: Person, org, project, tool, place → `wiki/entities/{subtype}/`
- **concept**: Idea, theory, pattern, technique → `wiki/concepts/`
- **source**: Summary of a raw source → `wiki/sources/`
- **synthesis**: Cross-cutting analysis → `wiki/syntheses/`
- **decision**: Decision record → `wiki/decisions/`
- **map**: Map of Content (thematic index) → `wiki/maps/`

Every page MUST have frontmatter matching the schema in `_soul/SCHEMA.md`.

## Ingest Workflow

When told to ingest a source:
1. Read the raw source completely.
2. Create/update a source summary page in `wiki/sources/`.
3. Extract entities. For each, create or update its entity page.
4. Extract concepts. For each, create or update its concept page.
5. For every claim, check for contradictions with existing pages.
   - If contradiction found: apply supersession rules, update both pages.
6. Add wikilinks bidirectionally between all related pages.
7. Update every touched page's frontmatter (confidence, source_count, updated, related_entities).
8. Update `_index/index.md` with any new pages.
9. Append an entry to `_index/log.md`: `## [YYYY-MM-DD] ingest | {Source Title}`
10. Mark raw source frontmatter `ingested: true`.

## Confidence Rules

- New claim from 1 source: confidence 0.5–0.7
- Confirmed by 2+ sources: confidence 0.7–0.9
- Confirmed by 3+ sources with no contradictions: confidence 0.9+
- Contradicted by any source: max confidence 0.7, note contradiction
- Not accessed/reinforced in 90+ days: decay by 0.1
- Status `stale` if confidence < 0.3

## Linking Rules

- Use `[[wikilinks]]` for all internal references.
- Every entity/concept mentioned in prose MUST be linked.
- Every page must have ≥1 inbound link (no orphans).
- Add all relationships to frontmatter `related_entities` with type and confidence.
- Prefer canonical page names over aliases in links.

## Lint Checklist

When told to lint:
- [ ] Orphan pages (no inbound links)
- [ ] Missing entity pages (mentioned but no page exists)
- [ ] Stale pages (not updated in 90+ days, confidence decaying)
- [ ] Contradictions unresolved
- [ ] Broken wikilinks
- [ ] Pages missing required frontmatter fields
- [ ] Source summaries without corresponding raw source
- [ ] Superseded pages not marked `status: superseded`

## Consolidation (Memory Tier Promotion)

- End of session: compress `memory/working/` → `memory/episodic/`
- Weekly: scan episodic for patterns → promote to `memory/semantic/` or `wiki/`
- Monthly: scan semantic for repeated procedures → promote to `memory/procedural/`

## Forbidden Actions

- Never modify files in `raw/` (except setting `ingested: true`).
- Never delete pages without explicit user instruction.
- Never set confidence > 0.95 (epistemic humility).
- Never fabricate sources or citations.
- Never skip frontmatter on any wiki page.
- Never create a page without at least one wikilink to an existing page.

## Query Behavior

When answering questions:
1. Read `_index/index.md` to find relevant pages.
2. Read the relevant pages.
3. Synthesize an answer with `[[wikilinks]]` as citations.
4. If the answer is substantive, offer to file it as a `wiki/syntheses/` page.
5. If the question reveals a knowledge gap, note it in `_index/inbox.md`.
```

---

## Summary of Key Architectural Principles

1. **Three layers preserved from v1**: raw (immutable) → wiki (LLM-maintained) → schema (co-evolved rules). The schema is the most important file.

2. **Four memory tiers from v2** map to `memory/working|episodic|semantic|procedural/`, with the wiki itself serving as the primary semantic store.

3. **Dual representation** for the knowledge graph: wikilinks in prose for Obsidian graph view + structured YAML in frontmatter for programmatic queries.

4. **Confidence is a first-class field.** Every claim carries a score that decays over time and strengthens with reinforcement. This prevents the wiki from becoming a "junk drawer" of equally-weighted stale facts.

5. **Kebab-case + wikilinks + typed subdirectories** gives you a naming system that works in Obsidian's graph, in `grep`, in Dataview, and in custom scripts simultaneously.

6. **The `copilot-instructions.md` is the schema layer** — it encodes ingest workflows, confidence rules, linking rules, lint checklists, and forbidden actions. It's what turns a generic LLM into a disciplined librarian.