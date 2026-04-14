Act as seasoned prompt Engineer and AI Architect , Github copilot chat, and Claude, Codex expert then generate prompt for   "I am building an "Agent-Driven-Development System" that automates the full SDLC lifecycle using Agents and Agent Orchestration. The system takes inputs like PRD, BRD, Features, and Requirements, and executes end-to-end software development. This will be Fully Autonomous Software Development Life Cycle . Must be compatible Github copilot, Claude Code , Open AI Codex and KIRO. Generate Corresponding agents , Skills, prompts and hooks to have this automation. For this implementation no human intervention so it must execute complete pipeline and for each phase it must Plan -> Design -> Implement -> Validate and retry in case failed. It must also have PIPELINE-EXECUTION.md file generated and updated after each step. It must be production grade. Keep agents file as sort and implement Skills for detailed use cases and implementations to preserve context window. Must use subagents as subagents have their own  context window. " <SDLC-PIPELINE>```
PRD/BRD/FRD
  │
  ▼
┌─────────────────────────────────────┐
│ Phase 1: REQUIREMENT INTELLIGENCE   │  ← agent(@requirement-analyst)
│  Output: Requirement Manifest       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Phase 2: PRODUCT DECOMPOSITION      │  ← agent(@product-decomposition)
│  Output: Decomposition Blueprint    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Phase 3: ARCHITECTURE               │  ← agent(@architect)
│  Output: Architecture Document      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Phase 4: API CONTRACT DESIGN        │  ← agent(@api-designer) + agent(@openapi)
│  Output: OpenAPI + AsyncAPI specs   │     + agent(@asyncapi) if events detected
└──────────────┬──────────────────────┘
               │
     ┌─────────┼─────────┬─────────┐
     ▼         ▼         ▼         ▼ (conditional)
┌─────────┐┌─────────┐┌─────────┐┌─────────┐
│Phase 5a ││Phase 5b ││Phase 5c ││Phase 5d │  ← PARALLEL
│BACKEND  ││FRONTEND ││DATABASE ││AI/ML    │
│@backend ││@frontend││@db-engr ││@ai-ml   │
│@spring  ││@react   ││         ││(cond.)  │
│         ││(cond.)  ││         ││         │
└────┬────┘└────┬────┘└────┬────┘└────┬────┘
     │          │          │          │
     └──────────┴──────────┴──────────┘
                │
                ▼
┌─────────────────────────────────────┐
│ Phase 6: TEST GENERATION            │  ← agent(@testing-qa)
│  Output: Test Suites (unit/integ/   │
│          contract/e2e)              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Phase 7: VALIDATION & QA            │  ← agent(@validation-qa)
│  Output: Validation Report          │
│  PASS → Phase 8                     │
│  FAIL → Auto-Fix Loop (max 3x)     │
└──────────────┬──────────────────────┘
               │
          ┌────┴────┐
          ▼         ▼
      [PASS]    [FAIL]──→ Auto-Fix Loop ──→ Re-validate
          │                                     │
          │         ┌───────────────────────────┘
          ▼         ▼
┌─────────────────────────────────────┐
│ Phase 8: DOCUMENTATION              │  ← agent(@documentation-writer)
│  Output: README, API docs, guides   │
└──────────────┬──────────────────────┘
               │
               ▼ (conditional)
┌─────────────────────────────────────┐
│ Phase 9: DEVOPS (CONDITIONAL)       │  ← agent(@devops-engineer)
│  Output: Dockerfile, CI/CD, IaC    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Phase 10: FINAL ASSEMBLY            │  ← self (@orchestrator)
│  Output: VALIDATION_REPORT.md,      │
│  traceability matrix, completeness  │
│  score. Mark PIPELINE-EXECUTION.md  │
│  as ✅ COMPLETED.                   │
└─────────────────────────────────────┘
```</SDLC-PIPELINE>  <OTHER-DETAILS> ## AUTO-EXECUTION PROTOCOL — Zero Human Intervention

When activated with a PRD/BRD/FRD document or Requirement Manifest, execute the full pipeline without stopping.

```
PIPELINE AUTO-EXECUTION RULES:

1. NEVER ask the user for confirmation between phases
2. NEVER stop to ask clarifying questions — make default decisions and document them
3. Apply default tech stack (Java 21 + Spring Boot 3.x + React 19/TS + H2 In memory DB) unless manifest overrides
4. Detect and activate conditional phases automatically:
   - Frontend (Phase 5b): SKIP if manifest says "API-only" or frontendNeeded=false
   - AI/ML (Phase 5d): ACTIVATE as parallel sub-phase if complexity.aiMlScope=EXTENSIVE or CRITICAL
   - DevOps (Phase 9): SKIP unless PRD mentions deployment/infra/CI-CD/Docker/cloud
   - AsyncAPI in Phase 4: INCLUDE if eventStreaming.detected=true
5. Write ALL artifacts to disk using editFiles
6. Run build/test validation using runCommand
7. On validation failure: enter auto-fix loop (max 3 cycles) before producing final report
8. Update PIPELINE-EXECUTION.md after EVERY phase transition
9. Git commit after each phase: `git add -A && git commit -m "Phase N: {name} complete"`
10. On resume: check for existing PIPELINE-EXECUTION.md and skip completed phases
11. NEVER skip Phase 1 even if plan/design/spec files are found in workspace — pass them as
    SUPPLEMENTARY CONTEXT to @requirement-analyst, not as a substitute for Phase 1
12. NEVER implement code directly as @orchestrator — every phase delegates to specialist agent
13. ALL generated files MUST be written to PROJECT_ROOT — never inside .github/ or the agents-lab structure
14. If workspace is AGENTS_META (agents-lab repo): PROJECT_ROOT = {workspace}/{project-slug}/
    (a new subfolder created INSIDE the workspace, not at workspace parent or a sibling directory)

⛔ CRITICAL ANTI-FAKE-COMPLETION RULES (added after observed pipeline failure):

15. CONTEXT FILES ARE READ-ONLY.
    Any file found via workspace scan (Step 1c of @full-lifecycle or step (d) of Phase 0)
    MUST NEVER be written to, edited, or overwritten — not even partially.
    These files are immutable inputs. Violating this rule is the #1 observed failure mode.

16. A PHASE IS ONLY ✅ DONE WHEN FILES EXIST ON DISK.
    After every phase that should produce files, run:
      runCommand("ls -la {PROJECT_ROOT}/{expected_artifact_path}")
    If the command returns no files or errors → the phase is ❌ FAILED, not done.
    A text response from a sub-agent claiming "files were written" is NOT proof of completion.

17. NEVER FABRICATE COMPLETION STATISTICS.
    Numbers like "42 tests", "85% coverage", "26 components" are ONLY valid if produced by
    actual runCommand output (e.g., mvn test, npm test, coverage report).
    Inventing these statistics is a critical failure — it deceives the user.

18. CREATE PROJECT_ROOT BEFORE PHASE 1.
    Run: runCommand("mkdir -p {PROJECT_ROOT}") as the very first action.
    Verify it exists before proceeding.

19. REJECT SUB-AGENT FAKE COMPLETION.
    If a sub-agent returns a summary claiming phases are done but
    runCommand("ls {PROJECT_ROOT}/{expected_artifact}") shows no files:
    → Mark the phase ❌ FAILED in PIPELINE-EXECUTION.md
    → Do NOT accept the text claim as success
    → Attempt to re-invoke the sub-agent with explicit editFiles instruction
```</OTHER-DETAILS> <PIPELINE-EXECUTION-SAMPLE> Pipeline Execution Dashboard

**Project:** {PROJECT-NAME} 
**Started:** {DATE} 
**Status:** ✅ COMPLETED  
**Resume Point:** —

---

## Phase Status

| # | Phase | Agent | Status | Artifacts |
|---|---|---|---|---|
| 0 | Workspace Setup | @orchestrator | ✅ Done | PIPELINE-EXECUTION.md |
| 1 | Requirement Intelligence | @requirement-analyst | ✅ Done | requirements/requirement-manifest.yaml |
| 2 | Product Decomposition | @product-decomposition | ✅ Done | specs/decomposition-blueprint.yaml |
| 3 | Architecture | @architect | ✅ Done | specs/architecture/architecture.md |
| 4 | API Contract Design | @api-designer | ✅ Done | specs/apis/component-contracts.md |
| 5a | Backend Generation | @backend-java @spring-boot | ⏭️ Skipped | Frontend-only project |
| 5b | Frontend Generation | @frontend-react | ✅ Done | All src/** files, package.json, vite.config.ts |
| 5c | Database Generation | @database-engineer | ⏭️ Skipped | Frontend-only project |
| 5d | AI/ML Integration | @ai-ml-engineer | ⏭️ Skipped | No AI/ML scope detected |
| 6 | Test Generation | @testing-qa | ✅ Done | src/test/setup.ts, src/test/calculations.test.ts |
| 7 | Validation & QA | @validation-qa | ✅ Done | VALIDATION_REPORT.md |
| 8 | Documentation | @documentation-writer | ✅ Done | README.md |
| 9 | DevOps | @devops-engineer | ✅ Done | Dockerfile, nginx.conf, .github/workflows/ci.yml |
| 10 | Final Assembly | @orchestrator | ✅ Done | VALIDATION_REPORT.md, PIPELINE-EXECUTION.md |

## Auto-Fix Log

No auto-fix cycles required.

## Decisions & Assumptions

- **App Type:** React/TypeScript SPA — phases 5a, 5c, 5d all skipped (frontend-only)
- **Tech Stack:** React 18, TypeScript 5, Vite 5, Framer Motion 11, Lucide React, CSS Custom Properties
- **State:** React Context + useReducer with localStorage persistence
- **No Tailwind:** Pure CSS Custom Properties design token system as specified in PRD
- **DevOps:** Phase 9 activated — PRD mentions deployment consideration
- **Formulas:** All 9 PRD acceptance criteria formulas implemented exactly as specified
- **Dark Mode:** System-preference-aware + manual toggle
- **Font:** FKGroteskNeue loaded from perplexity CDN as specified in PRD design tokens

## Artifact Inventory

```
wbs-estimator/
├── PIPELINE-EXECUTION.md
├── VALIDATION_REPORT.md
├── README.md
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.node.json
├── index.html
├── Dockerfile
├── nginx.conf
├── .eslintrc.cjs
├── .gitignore
├── .github/workflows/ci.yml
├── requirements/requirement-manifest.yaml
├── specs/
│   ├── decomposition-blueprint.yaml
│   ├── architecture/architecture.md
│   └── apis/component-contracts.md
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── index.css
    ├── types/wbs.ts
    ├── utils/calculations.ts
    ├── utils/exportSummary.ts
    ├── context/wbsReducer.ts
    ├── context/WBSContext.tsx
    ├── context/ThemeContext.tsx
    ├── hooks/useLocalStorage.ts
    ├── hooks/useWBSCalculations.ts
    ├── test/setup.ts
    ├── test/calculations.test.ts
    ├── components/ui/
    │   ├── Card.tsx
    │   ├── AnimatedNumber.tsx
    │   ├── TabPanel.tsx
    │   ├── ComplexityItem.tsx
    │   ├── ComponentItem.tsx
    │   ├── InfoTooltip.tsx
    │   ├── CalculationResult.tsx
    │   └── Toast.tsx
    ├── components/layout/
    │   ├── Header.tsx
    │   ├── SummaryCards.tsx
    │   ├── InfoCards.tsx
    │   ├── TabNavigation.tsx
    │   └── ExportSection.tsx
    └── components/tabs/
        ├── ApiEstimator.tsx
        ├── BackendComponents.tsx
        ├── DatabaseDesign.tsx
        ├── BatchJobs.tsx
        ├── FrontendUI.tsx
        ├── Infrastructure.tsx
        ├── AWSMigration.tsx
        ├── PERTEstimation.tsx
        ├── RiskAssessment.tsx
        ├── TestingFactors.tsx
        ├── AIProductivity.tsx
        ├── ResourcePlanning.tsx
        └── Reference.tsx
```
</PIPELINE-EXECUTION-SAMPLE>