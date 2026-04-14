# Plan: Agent-Driven SDLC Automation System

## TL;DR

Build a fully autonomous, cross-platform Agent-Driven-Development System that automates the complete SDLC pipeline (10 phases) using 17 specialized agents, 15+ reusable skills, hooks, prompts, and instructions. Compatible with GitHub Copilot Chat, Claude Code, OpenAI Codex, and KIRO. The system takes PRD/BRD/FRD as input and produces production-grade software with zero human intervention.

## Cross-Platform Compatibility Strategy

Each platform reads different file paths but supports overlapping concepts:

| Concept | GitHub Copilot | Claude Code | OpenAI Codex | KIRO |
|---------|---------------|-------------|--------------|------|
| Project Instructions | `copilot-instructions.md` or `AGENTS.md` | `CLAUDE.md` | `AGENTS.md` | `.kiro/steering/*.md` |
| Agents | `.github/agents/*.agent.md` | `.claude/agents/*.md` | Built-in + config.toml roles | N/A (steering-driven) |
| Skills | `.github/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` | `.codex/skills/<name>/SKILL.md` | N/A |
| Hooks | `.github/hooks/*.json` | `.claude/settings.json` | N/A | `.kiro/hooks/*.kiro` |
| Prompts | `.github/prompts/*.prompt.md` | `.claude/commands/*.md` | N/A | N/A |
| Instructions | `.github/instructions/*.instructions.md` | `.claude/rules/*.md` | N/A | `.kiro/steering/*.md` |

**Strategy**: Create files in ALL platform-specific locations. Use symlinks or duplicated content where formats are identical (skills share SKILL.md format across Copilot/Claude/Codex).

---

## Phase A: Foundation & Workspace Instructions

### Step 1: Create directory scaffold
*No dependencies*

Create the full directory tree:

```
.github/
├── agents/           # 17 Copilot agent files
├── skills/           # 15+ skill directories
├── hooks/            # 3 hook configs
├── prompts/          # 2 prompt files
└── instructions/     # 3 instruction files

.claude/
├── CLAUDE.md         # Claude Code project instructions
├── agents/           # 17 Claude agent files
├── skills/           # Symlinks or copies of .github/skills/
├── rules/            # Claude rules (maps to instructions)
├── commands/         # Claude commands (maps to prompts)
└── settings.json     # Claude hooks config

.kiro/
├── steering/         # KIRO project guidelines
├── specs/            # KIRO spec templates
└── hooks/            # KIRO hooks

.codex/
└── skills/           # Codex skill copies/symlinks

AGENTS.md             # Root-level (Copilot + Codex shared)
```

### Step 2: Create root project instructions
*Depends on Step 1*

- `AGENTS.md` (root) — Shared by Copilot + Codex. Contains: orchestrator protocol, pipeline rules, default tech stack, anti-fake-completion rules, auto-execution protocol
- `CLAUDE.md` (via `.claude/CLAUDE.md`) — Same core content adapted for Claude Code format with `@path` imports
- `.kiro/steering/sdlc-orchestration.md` — KIRO steering file with same pipeline rules

### Step 3: Create platform-specific instructions/rules
*Parallel with Step 2*

**GitHub Copilot** (`.github/instructions/`):
- `sdlc-pipeline.instructions.md` — Pipeline execution rules, phase ordering, conditional logic
- `anti-fake-completion.instructions.md` — Rules 15-19 from anti-fake-completion protocol
- `default-tech-stack.instructions.md` — Java 21 + Spring Boot 3.x + React 19/TS + H2 defaults

**Claude Code** (`.claude/rules/`):
- `sdlc-pipeline.md` — Same content, Claude rules format with optional `paths:` frontmatter
- `anti-fake-completion.md`
- `default-tech-stack.md`

**KIRO** (`.kiro/steering/`):
- `pipeline-execution.md` — Pipeline + anti-fake + tech stack combined (KIRO has fewer files)
- `quality-gates.md` — Validation enforcement

---

## Phase B: Agent Definitions (17 agents)

### Step 4: Create Orchestrator Agent
*Depends on Steps 1-3*

**@orchestrator** — The central pipeline controller. Delegates ALL work, never implements code directly.

- `.github/agents/orchestrator.agent.md` — Copilot format with frontmatter: `tools: [agent, read, search, execute, edit]`, `description: "Use when: running full SDLC pipeline, executing autonomous development, orchestrating multi-phase software generation"`. Body: phase transition logic, PIPELINE-EXECUTION.md management, auto-fix loop, git commit protocol, conditional phase detection.
- `.claude/agents/orchestrator.md` — Claude format with `tools: ["Agent", "Read", "Write", "Bash"]` frontmatter
- Create the `full-lifecycle` prompt that activates the orchestrator (Step 10)

### Step 5: Create Core SDLC Agents (12 agents)
*Parallel with Step 4 (but these are subagents, orchestrator references them)*

Each agent follows the lean pattern: YAML frontmatter + 30-80 lines of body. Deep logic is delegated to skills.

| Agent | File | Tools | Skills Referenced |
|-------|------|-------|-------------------|
| @requirement-analyst | `requirement-analyst.agent.md` | `read, search, edit, agent` | `requirement-analysis` |
| @product-decomposition | `product-decomposition.agent.md` | `read, search, edit, agent` | `product-decomposition` |
| @architect | `architect.agent.md` | `read, search, edit, agent` | `architecture-design`, `ddd-strategy`, `hexagonal-architecture`, `microservices-decision`, `event-driven-architecture` |
| @api-designer | `api-designer.agent.md` | `read, search, edit, agent` | `api-design-best-practices`, `openapi-generation` |
| @openapi | `openapi.agent.md` | `read, edit` | `openapi-generation` |
| @asyncapi | `asyncapi.agent.md` | `read, edit` | `asyncapi-generation` |
| @backend-engineer | `backend-engineer.agent.md` | `read, search, edit, execute, agent` | `java-spring-boot` |
| @frontend-engineer | `frontend-engineer.agent.md` | `read, search, edit, execute, agent` | `react-typescript` |
| @database-engineer | `database-engineer.agent.md` | `read, search, edit, execute` | `java-spring-boot` (JPA/schema subset) |
| @ai-ml-engineer | `ai-ml-engineer.agent.md` | `read, search, edit, execute` | N/A (conditional) |
| @testing-qa | `testing-qa.agent.md` | `read, search, edit, execute, agent` | `test-strategy-generator` |
| @validation-qa | `validation-qa.agent.md` | `read, search, execute` | `validation-and-qa` |

**Key pattern**: Each agent sets `user-invocable: false` (they are subagents invoked by @orchestrator). Body uses `#tool:<skill-name>` references so skills load on-demand.

Create corresponding `.claude/agents/<name>.md` files for each.

### Step 6: Create Post-Pipeline Agents (3 agents)
*Parallel with Step 5*

| Agent | File | Tools | Purpose |
|-------|------|-------|---------|
| @documentation-writer | `documentation-writer.agent.md` | `read, search, edit` | README, API docs, guides |
| @devops-engineer | `devops-engineer.agent.md` | `read, search, edit, execute` | Dockerfile, CI/CD, IaC |
| @self-learning | `self-learning.agent.md` | `read, search, edit` | Captures failures, generates improvement-rules.yaml |

### Step 7: Create System Agents (1 agent)
*Parallel with Steps 5-6*

| Agent | File | Tools | Purpose |
|-------|------|-------|---------|
| @long-term-memory | `long-term-memory.agent.md` | `read, edit` | Stores/retrieves design decisions, failure patterns, best practices |

**Total: 17 agents** × 2 platforms (Copilot + Claude) = **34 agent files**

---

## Phase C: Skills (15 skills)

### Step 8: Create Architecture Skills (4 skills)
*Depends on Step 1*

Each skill follows: `.github/skills/<name>/SKILL.md` + optional `references/` and `scripts/` subdirectories.

| Skill | Directory | Content Focus |
|-------|-----------|---------------|
| `ddd-strategy` | `.github/skills/ddd-strategy/` | Bounded contexts, aggregates, domain events, ubiquitous language extraction, context mapping |
| `hexagonal-architecture` | `.github/skills/hexagonal-architecture/` | Ports & adapters, dependency inversion, package structure templates |
| `microservices-decision` | `.github/skills/microservices-decision/` | Decision matrix (monolith vs microservices), decomposition strategies, communication patterns |
| `event-driven-architecture` | `.github/skills/event-driven-architecture/` | Event sourcing, CQRS, saga patterns, AsyncAPI integration |

Each SKILL.md: frontmatter (`name`, `description` with trigger phrases, `user-invocable: false`) + body (when to use, procedure, reference links). Under 500 lines each.

### Step 9: Create Coding Skills (3 skills)
*Parallel with Step 8*

| Skill | Directory | Content Focus |
|-------|-----------|---------------|
| `java-spring-boot` | `.github/skills/java-spring-boot/` | Spring Boot 3.x + Java 21 patterns: project structure, controller/service/repository, JPA entities, security config, application.yml templates, Maven pom.xml generation |
| `react-typescript` | `.github/skills/react-typescript/` | React 19 + TS patterns: Vite setup, component patterns, state management, routing, testing setup |
| `api-design-best-practices` | `.github/skills/api-design-best-practices/` | RESTful conventions, error handling, pagination, versioning, HATEOAS |

Each skill includes `references/` with code templates and `scripts/` with generation helpers.

### Step 10: Create SDLC Process Skills (6 skills)
*Parallel with Steps 8-9*

| Skill | Directory | Content Focus |
|-------|-----------|---------------|
| `requirement-analysis` | `.github/skills/requirement-analysis/` | PRD/BRD/FRD parsing, requirement-manifest.yaml generation, EARS syntax, user story extraction |
| `product-decomposition` | `.github/skills/product-decomposition/` | Feature breakdown, component identification, dependency mapping, decomposition-blueprint.yaml generation |
| `architecture-design` | `.github/skills/architecture-design/` | Architecture document generation, C4 model, technology selection, non-functional requirements |
| `openapi-generation` | `.github/skills/openapi-generation/` | OpenAPI 3.x spec generation from requirements, endpoint design, schema modeling |
| `asyncapi-generation` | `.github/skills/asyncapi-generation/` | AsyncAPI spec generation, event channel design, message schema modeling |
| `validation-and-qa` | `.github/skills/validation-and-qa/` | Build validation, test execution, coverage checks, lint checks, VALIDATION_REPORT.md generation |

### Step 11: Create Quality & DevOps Skills (4 skills)
*Parallel with Steps 8-10*

| Skill | Directory | Content Focus |
|-------|-----------|---------------|
| `test-strategy-generator` | `.github/skills/test-strategy-generator/` | Unit/integration/contract/e2e test generation, coverage strategy, test file organization |
| `security-hardening` | `.github/skills/security-hardening/` | OWASP Top 10, input validation, auth patterns, dependency scanning |
| `performance-optimization` | `.github/skills/performance-optimization/` | Profiling, caching strategy, query optimization, load testing |
| `cicd-pipelines` | `.github/skills/cicd-pipelines/` | GitHub Actions workflows, Dockerfile generation, docker-compose, IaC templates |

### Step 12: Copy/Symlink Skills to Other Platforms
*Depends on Steps 8-11*

- Copy each `.github/skills/<name>/` to `.claude/skills/<name>/` and `.codex/skills/<name>/`
- SKILL.md format is identical across all three platforms
- For `.agents/skills/` (shared), create symlinks pointing to `.github/skills/`

---

## Phase D: Prompts, Hooks, and Entry Points

### Step 13: Create Prompts (2 prompt files)
*Depends on Steps 4-7 (agents must exist)*

**GitHub Copilot** (`.github/prompts/`):
- `full-lifecycle.prompt.md` — Entry point. Frontmatter: `agent: "orchestrator"`, `tools: [agent, read, search, edit, execute]`. Body: instructs orchestrator to execute full 10-phase pipeline from PRD/BRD/FRD input.
- `phase-resume.prompt.md` — Resume prompt. Reads PIPELINE-EXECUTION.md and continues from last incomplete phase.

**Claude Code** (`.claude/commands/`):
- `full-lifecycle.md` — Equivalent slash command for Claude
- `phase-resume.md` — Resume command

### Step 14: Create Hooks (3 hook configurations)
*Depends on Steps 1-3*

**GitHub Copilot** (`.github/hooks/`):
- `pipeline-tracker.json` — `PostToolUse` hook: runs a script to check if PIPELINE-EXECUTION.md needs updating after file writes
- `artifact-validator.json` — `PostToolUse` hook: validates that claimed artifacts actually exist on disk (anti-fake-completion)
- `git-auto-commit.json` — `PostToolUse` hook: auto-commits after major phase completions

Hook scripts go in `.github/hooks/scripts/`:
- `check-artifacts.sh` — Verifies file existence after phase completion
- `update-pipeline.sh` — Updates PIPELINE-EXECUTION.md status

**Claude Code** (`.claude/settings.json`):
- Same hooks defined in Claude's `hooks` config format under `PostToolUse` events

**KIRO** (`.kiro/hooks/`):
- `artifact-validation.kiro` — KIRO hook for artifact verification
- `pipeline-tracking.kiro` — KIRO hook for pipeline status updates

### Step 15: Create KIRO Spec Templates
*Parallel with Steps 13-14*

`.kiro/specs/sdlc-pipeline/`:
- `requirements.md` — Template for capturing SDLC pipeline requirements
- `design.md` — Technical design of the agent orchestration system
- `tasks.md` — Implementation task checklist for the pipeline

---

## Phase E: Templates and Supporting Files

### Step 16: Create PIPELINE-EXECUTION.md Template
*Depends on Phase A*

Create `templates/PIPELINE-EXECUTION.template.md` — The template that gets copied to PROJECT_ROOT at Phase 0. Contains the dashboard structure, phase table, auto-fix log, decisions section, and artifact inventory placeholder.

### Step 17: Create requirement-manifest.yaml Schema
*Parallel with Step 16*

Create `templates/requirement-manifest.schema.yaml` — YAML schema defining the structure for requirement manifests that flow through the pipeline. Includes: project metadata, functional/non-functional requirements, tech stack overrides, conditional flags (frontendNeeded, aiMlScope, eventStreaming, devopsRequired).

### Step 18: Create improvement-rules.yaml Template
*Parallel with Step 16*

Create `templates/improvement-rules.template.yaml` — Template for self-learning agent to record failure patterns, root causes, and fixes.

---

## Relevant Files (all to be CREATED — workspace is currently empty)

**Root level:**
- `AGENTS.md` — Shared Copilot + Codex project instructions (orchestrator protocol, pipeline rules)

**GitHub Copilot (.github/):**
- `.github/agents/orchestrator.agent.md` — Central pipeline controller
- `.github/agents/requirement-analyst.agent.md` — Phase 1
- `.github/agents/product-decomposition.agent.md` — Phase 2
- `.github/agents/architect.agent.md` — Phase 3
- `.github/agents/api-designer.agent.md` — Phase 4
- `.github/agents/openapi.agent.md` — Phase 4 sub
- `.github/agents/asyncapi.agent.md` — Phase 4 sub (conditional)
- `.github/agents/backend-engineer.agent.md` — Phase 5a
- `.github/agents/frontend-engineer.agent.md` — Phase 5b
- `.github/agents/database-engineer.agent.md` — Phase 5c
- `.github/agents/ai-ml-engineer.agent.md` — Phase 5d
- `.github/agents/testing-qa.agent.md` — Phase 6
- `.github/agents/validation-qa.agent.md` — Phase 7
- `.github/agents/documentation-writer.agent.md` — Phase 8
- `.github/agents/devops-engineer.agent.md` — Phase 9
- `.github/agents/self-learning.agent.md` — System agent
- `.github/agents/long-term-memory.agent.md` — System agent
- `.github/skills/*/SKILL.md` — 15 skill directories (see Steps 8-11)
- `.github/hooks/*.json` — 3 hook configs + `scripts/` directory
- `.github/prompts/full-lifecycle.prompt.md` — Entry point prompt
- `.github/prompts/phase-resume.prompt.md` — Resume prompt
- `.github/instructions/sdlc-pipeline.instructions.md`
- `.github/instructions/anti-fake-completion.instructions.md`
- `.github/instructions/default-tech-stack.instructions.md`

**Claude Code (.claude/):**
- `.claude/CLAUDE.md` — Project instructions
- `.claude/agents/*.md` — 17 agent files (same content, Claude format)
- `.claude/skills/*/SKILL.md` — 15 skill copies
- `.claude/rules/*.md` — 3 rule files
- `.claude/commands/*.md` — 2 command files
- `.claude/settings.json` — Hooks configuration

**KIRO (.kiro/):**
- `.kiro/steering/sdlc-orchestration.md`
- `.kiro/steering/pipeline-execution.md`
- `.kiro/steering/quality-gates.md`
- `.kiro/specs/sdlc-pipeline/requirements.md`
- `.kiro/specs/sdlc-pipeline/design.md`
- `.kiro/specs/sdlc-pipeline/tasks.md`
- `.kiro/hooks/artifact-validation.kiro`
- `.kiro/hooks/pipeline-tracking.kiro`

**Codex (.codex/):**
- `.codex/skills/*/SKILL.md` — 15 skill copies

**Templates:**
- `templates/PIPELINE-EXECUTION.template.md`
- `templates/requirement-manifest.schema.yaml`
- `templates/improvement-rules.template.yaml`

---

## Verification

1. **Structure validation**: Run `find . -name "*.agent.md" | wc -l` — expect 17 in `.github/agents/` and 17 in `.claude/agents/`
2. **Skill validation**: Run `find . -name "SKILL.md" | wc -l` — expect 15 per platform directory (45+ total)
3. **Hook validation**: Verify `.github/hooks/*.json` files are valid JSON with `jq . .github/hooks/*.json`
4. **Frontmatter validation**: Verify all agent files have valid YAML frontmatter with `description` field
5. **Cross-platform consistency**: Verify that each agent exists in both `.github/agents/` and `.claude/agents/`
6. **Prompt invocation test**: Open GitHub Copilot Chat, type `/`, verify `full-lifecycle` and `phase-resume` appear
7. **Agent discovery test**: In Copilot Chat, reference `@orchestrator` and verify it loads
8. **Skill discovery test**: Verify skills are discoverable via agent description matching
9. **End-to-end smoke test**: Create a minimal PRD, invoke `/full-lifecycle`, verify Phase 0-1 executes and creates PIPELINE-EXECUTION.md

---

## Decisions

- **AGENTS.md over copilot-instructions.md**: Using `AGENTS.md` at root because it's the shared standard between Copilot and Codex (cannot use both)
- **Duplicate vs Symlink for skills**: SKILL.md format is identical across Copilot/Claude/Codex — use file duplication (not symlinks) for portability and git compatibility
- **17 agents (not 14)**: Added @self-learning, @long-term-memory, and @orchestrator beyond the 14 core SDLC agents
- **Skills over embedded logic**: All deep domain knowledge (DDD, Spring Boot patterns, React patterns, etc.) lives in skills, not agent bodies — preserves context window
- **Subagent pattern**: All SDLC agents set `user-invocable: false` — only @orchestrator and prompts are user-facing
- **KIRO limited mapping**: KIRO doesn't have agents — its steering files + specs provide equivalent guidance
- **Hook scripts in shell**: Hooks use bash scripts for portability across macOS/Linux

## Further Considerations

1. **MCP Server Integration**: Consider adding MCP server configs for external tool integration (e.g., database access, CI/CD API calls). Recommend deferring to a follow-up iteration.
2. **Model Selection**: Agent files can specify `model:` in frontmatter. Recommend defaulting to Claude Sonnet 4 for most agents, with Claude Opus 4.6 for @architect and @orchestrator. This is Copilot-specific; Claude Code uses its own model selection.
3. **Skill Reference Depth**: Skills reference `./references/*.md` for detailed templates. Keeping SKILL.md under 500 lines with progressive loading is critical for context efficiency.

## Estimated File Count

- Agent files: ~34 (17 × 2 platforms)
- Skill files: ~60 (15 skills × ~4 files each across platforms)
- Hook files: ~8
- Prompt/Command files: ~4
- Instruction/Rule files: ~6
- Steering/Spec files: ~8
- Templates: ~3
- Root files: ~1

**Total: ~125 files**
