# KIRO Research Findings - Complete

## What is KIRO?
**Kiro** is an **agentic IDE** (3,432 GitHub stars) that works alongside developers from prototype to production. Made by AWS/Amazon, it emphasizes **spec-driven development** with agents, hooks, and steering for AI-powered development workflows.

**Key Technologies:**
- Built in TypeScript
- Available as Desktop IDE + CLI
- Supports macOS, Windows, Linux
- Integrates with GitHub, AWS services

---

## KIRO's Core Customization System

### 1. Directory Structure: `.kiro/`
```
.kiro/
├── steering/           # Always-active development guidelines
│   ├── [name].md      # Markdown files with project guidance
├── hooks/              # Automated workflow enforcement
│   ├── [name].kiro    # Hook configuration files
├── specs/              # Feature development specifications
│   ├── [feature-name]/
│   │   ├── requirements.md
│   │   ├── design.md
│   │   ├── tasks.md
│   │   └── pattern-rules/
└── settings/           # Kiro configuration
    └── mcp.json       # Model Context Protocol servers
```

### 2. STEERING FILES (`.kiro/steering/*.md`)
**Purpose:** Always-active project guidelines and development standards

**Frontmatter Format:**
```yaml
---
inclusion: always  # Options: always, fileMatch, manual
fileMatchPattern: "**/*.ts"  # Optional: glob pattern for selective application
---
```

**Common Steering Files:**
1. **tdd-methodology.md** - TDD cycle enforcement, test coverage standards
2. **development-workflow.md** - Mandatory development sequence, context preservation
3. **code-quality-gates.md** - Pre-commit requirements, automated checks
4. **architecture-decisions.md** - ADR templates, SOLID principles, DDD guidelines
5. **testing-standards.md** - Test file organization, assertion guidelines, mocking strategies
6. **Custom steering files** - Add any project-specific guidance (security, API style guides, etc.)

**Key Steering Metadata:**
- `inclusion: always` - Applied to all files always
- `inclusion: fileMatch` - Applied only to files matching `fileMatchPattern`
- `inclusion: manual` - Only applied when explicitly referenced

### 3. SPEC FILES (`.kiro/specs/[feature-name]/`)
**Four-Phase Spec-Driven Development Workflow:**

#### Phase 1: Requirements (`requirements.md`)
```markdown
# Feature Name

## Introduction
Overview of problem and objectives

## Requirements

**Requirement 1**
**User Story:** As a [role], I want [feature], so that [benefit]

**Acceptance Criteria**
- [ ] IF [condition], THEN [expected behavior]
- [ ] WHEN [action], THEN [outcome]
- [ ] WHERE [constraint], [behavior]
```

**Standards:** Uses EARS syntax (Easy Approach to Requirements Syntax)

#### Phase 2: Technical Design (`design.md`)
```markdown
# Technical Design: [Feature]

## 1. Architectural Overview
High-level solution and system integration

## 2. Data Flow Diagrams
Mermaid.js visualizations

## 3. Component & Interface Definitions
TypeScript types and specifications

## 4. API Endpoint Definitions
HTTP method, path, request/response schemas

## 5. Database Schema Changes
SQL DDL or ORM model definitions

## 6. Security Considerations
Risk assessment and mitigation

## 7. Test Strategy
Unit, integration, E2E test approach
```

#### Phase 3: Implementation Plan (`tasks.md`)
```markdown
# Implementation Plan

- [ ] 1. High-level goal description
  - Sub-task 1
  - Sub-task 2
  - Sub-task 3
  - _Requirements: [requirement numbers like 1.1, 1.2]_

- [ ] 2. Next high-level goal
  - Sub-task details
  - _Requirements: 2.1_
```

**Format Rules:**
- High-level tasks: `- [ ] 1. Description` (numbered, checkboxes)
- Sub-tasks: Indented bullet points
- Traceability: Requirement references like `1.1` (req 1, acceptance criterion 1)

#### Phase 4: Implementation Log
```markdown
## Implementation Log

### TDD Cycle Progress
#### [Date] - Cycle 1 Complete
- Tests Written: [list]
- Code Changes: [description]
- Refactoring: [improvements]
- Coverage: [percentage]

### Architecture Decisions Made
#### [Date] - Decision: [Title]
- Context: [why needed]
- Decision: [chosen solution]
- Trade-offs: [benefits vs costs]
- ADR Reference: [link]
```

### 4. HOOKS (`.kiro/hooks/*.kiro`)
**Purpose:** Automated workflow enforcement and quality gates

**Common Hook Types:**
1. **test-first-enforcement.kiro** - Prevents saving production code without tests
2. **pattern-recognition.kiro** - Detects recurring issues, generates proposed rules
3. **architecture-tracker.kiro** - Monitors architectural decisions consistency
4. **documentation-generator.kiro** - Auto-updates docs on code changes
5. **regression-detection.kiro** - Identifies potential regressions

**Hook File Pattern**: `.kiro` extension (custom KIRO format)

### 5. MCP INTEGRATION (`.kiro/settings/mcp.json`)
**Purpose:** Connect external tools via Model Context Protocol

```json
{
  "servers": [
    {
      "name": "tool-name",
      "command": "node",
      "args": ["path/to/tool"]
    }
  ]
}
```

### 6. ADR TEMPLATE (In Steering or Specs)
```markdown
# ADR-XXXX: [Title]

Date: YYYY-MM-DD
Status: [Proposed | Accepted | Deprecated | Superseded]
Deciders: [List]

## Context
[Problem and constraints]

## Decision
[Chosen solution]

## Consequences
### Positive: [Benefits]
### Negative: [Drawbacks]
### Mitigation: [Risk mitigation]

## Implementation Notes
[Technical guidance]
```

---

## KIRO Workflow/Commands

### Spec-Driven Development Prompts
KIRO uses structured prompts (like those in [wirelessr/kiro-workflow-prompts](https://github.com/wirelessr/kiro-workflow-prompts)):

1. **`/createSpec`** - Transform feature ideas → `requirements.md`
2. **`/design`** - Convert requirements → `design.md`
3. **`/createTask`** - Decompose design → `tasks.md`
4. **`/executeTask`** - Execute tasks from `tasks.md`
5. **`/commit`** - Professional Git commits
6. **`/prReview`** - Code review assistant

### Approval Gates
- Explicit approval required after each phase
- Mandatory context gathering before coding
- Design document as supreme authority

---

## KIRO File Format Standards

### Markdown Files
- **Requirements**: User Stories + EARS syntax for acceptance criteria
- **Design**: Sections with architectural details, code samples (TypeScript, SQL)
- **Tasks**: Hierarchical checklist format with traceability
- **Steering**: Plain markdown with YAML frontmatter

### Naming Conventions
- Test files: `ComponentName.test.ts`, `Feature.integration.test.ts`, `Flow.e2e.test.ts`
- Feature directories: `feature-name/` (kebab-case, lowercase)
- Steering files: descriptive names (e.g., `tdd-methodology.md`, `security-principles.md`)

### Code Coverage & Quality Standards
- **Minimum coverage**: 90% line coverage
- **Cyclomatic complexity**: < 10 per method
- **Max method lines**: 30 lines
- **Max class methods**: 20 methods
- **No duplicate code**: > 6 lines
- **Performance**: Unit tests < 100ms, integration tests < 1s, full suite < 30s

---

## Key KIRO Patterns

### Pattern Recognition
Hooks can detect recurring issues and auto-generate:
- `pattern-rules/detected-patterns.md` - Identified problems
- `pattern-rules/proposed-rules.md` - Suggested solutions

### Context Preservation
Each development session should log:
- TDD cycle tracking
- Architecture decisions
- Known patterns learned
- Environment-specific guidance (Windows/PowerShell focus in examples)

### Testing Architecture
- **Separated concerns**: Domain logic → Infrastructure
- **Dependency injection** for testability
- **Isolated unit test targets**
- **Layered testing**: Presentation → Application → Domain

---

## Reference Implementations

### Complete TDD Workflow
Repository: [5capeg0at/kiro-tdd-workflow-files](https://github.com/5capeg0at/kiro-tdd-workflow-files)
- Full steering files setup
- Hook implementations
- Spec templates
- Quality gate configurations

### Workflow Prompts
Repository: [wirelessr/kiro-workflow-prompts](https://github.com/wirelessr/kiro-workflow-prompts)
- `/createSpec`, `/design`, `/createTask`, `/executeTask` prompts
- Design principles (Linus Torvalds guidelines)
- Commit and PR review prompts

### Main KIRO IDE
Repository: [kirodotdev/Kiro](https://github.com/kirodotdev/Kiro) (3.4k stars)
- Official KIRO IDE source
- Example `.kiro/specs/github-issue-automation` directory
- TypeScript implementation reference

---

## npm Packages for KIRO Ecosystem

Key packages for spec-driven development:
- `kiro-spec-engine` - CLI tool for spec-driven work
- `kspec` - Spec-driven development workflow (2.1k downloads)
- `konductor` - Spec orchestrator for Kiro CLI (MCP server)
- `kiro-memory-skill` - Persistent memory system (3.2.0)
- `skillpull` - Sync AI agent skills to Claude, Codex, Kiro, Cursor

---

## Workspace Integration Status

**Current:** No KIRO files in `/Users/gagan/Code/Non-Production-Apps/add-ecosystem-global`

**To Create KIRO Configuration:**
1. Create `.kiro/` directory at project root
2. Add steering files: `steering/*.md` with project guidelines
3. Create spec templates: `specs/[feature-name]/{requirements,design,tasks}.md`
4. Configure hooks: `hooks/*.kiro` files
5. Add MCP config: `settings/mcp.json` if using external tools



What is KIRO?
KIRO is an AWS-built agentic IDE (3,432+ GitHub stars) that emphasizes spec-driven development. It's available as both a desktop application and CLI, supporting macOS, Windows, and Linux. Built in TypeScript, it integrates GitHub, AWS services, and Model Context Protocol (MCP) for AI-powered development.

KIRO's Directory Structure: .kiro/
.kiro/
├── steering/           # Always-active project guidelines (markdown)
│   ├── tdd-methodology.md
│   ├── development-workflow.md
│   ├── code-quality-gates.md
│   ├── architecture-decisions.md
│   ├── testing-standards.md
│   └── [custom-guide].md
├── hooks/              # Automated workflow enforcement (.kiro files)
│   ├── test-first-enforcement.kiro
│   ├── pattern-recognition.kiro
│   ├── architecture-tracker.kiro
│   ├── documentation-generator.kiro
│   └── regression-detection.kiro
├── specs/              # Feature specifications (organized by feature name)
│   ├── [feature-name]/
│   │   ├── requirements.md
│   │   ├── design.md
│   │   ├── tasks.md
│   │   └── pattern-rules/
│   └── ...
└── settings/           # Configuration
    └── mcp.json       # Model Context Protocol servers
1. STEERING FILES (.kiro/steering/)
Purpose: Project-wide development guidelines, always active

Format: Markdown with YAML frontmatter:

---
inclusion: always  # Options: always, fileMatch, manual
fileMatchPattern: "**/*.ts"  # Optional glob pattern
---
Standard Steering Files:

File	Purpose	Content
tdd-methodology.md	Enforce TDD practices	Red-Green-Refactor cycle, coverage (90%+), anti-patterns
development-workflow.md	Development sequence	Mandatory phases, context preservation, info handoff
code-quality-gates.md	Pre-commit requirements	Coverage gates, complexity metrics, refactoring rules
architecture-decisions.md	Architecture guidance	ADR templates, SOLID principles, DDD guidelines, layering
testing-standards.md	Testing conventions	Test file org, assertion styles, mocking strategy, performance
Custom Steering: Add any .md file with project-specific guidance (security, API style guides, etc.)

2. SPEC FILES (.kiro/specs/[feature-name]/)
Four-Phase Workflow – Each Creates One File:

Phase 1: requirements.md (Spec Creation)
# Feature Name

## Introduction
Problem overview and objectives

## Requirements

**Requirement 1**
**User Story:** As a [role], I want [feature], so that [benefit]

**Acceptance Criteria** (EARS syntax)
- [ ] IF [condition], THEN [behavior]
- [ ] WHEN [action], THEN [outcome]
- [ ] WHERE [constraint], [expected result]

**Acceptance Criteria**
- [ ] WHEN user submits review, THEN rating saved (1-5)
- [ ] IF comment > 500 chars, THEN rejection
Standards: User Stories + EARS (Easy Approach to Requirements Syntax)

Phase 2: design.md (Technical Design)
# Technical Design: [Feature]

## 1. Architectural Overview
High-level solution, system integration

## 2. Data Flow Diagrams
Mermaid.js visualizations of data movement

## 3. Component & Interface Definitions
TypeScript interfaces, types, method signatures

```typescript
interface Review {
  productId: string;
  rating: number;  // 1-5
  comment: string; // max 500 chars
}
4. API Endpoint Definitions
POST /api/v1/reviews
Request Body: { productId, rating, comment }
Response: { id, ...Review, createdAt }
5. Database Schema Changes
CREATE TABLE reviews (
  id UUID PRIMARY KEY,
  product_id VARCHAR(50),
  rating INT CHECK (rating BETWEEN 1 AND 5),
  comment VARCHAR(500),
  created_at TIMESTAMP
);
6. Security Considerations
Input validation, authentication checks, injection prevention

7. Test Strategy
Unit tests, integration tests, E2E test coverage


### **Phase 3: `tasks.md`** (Implementation Plan)
```markdown
# Implementation Plan

- [ ] 1. Set up database and data access layer
  - Create database migration for `reviews` table
  - Implement `ReviewRepository` with save method
  - Write unit tests for repository
  - _Requirements: 1.2_

- [ ] 2. Implement core business logic
  - Validate rating (1-5) and comment length (≤500)
  - Implement `ReviewService.createReview()`
  - Write service unit tests
  - _Requirements: 1.1, 1.4_

- [ ] 3. Expose via API
  - Create `POST /api/v1/reviews` endpoint
  - Connect controller to service
  - Write integration test
  - _Requirements: 1.3_
Format Rules:

High-level: - [ ] 1. Description (numbered, checkboxes)
Sub-tasks: Indented bullets
Traceability: _Requirements: 1.1, 2.2_ (requirement.criterion)
Phase 4: Implementation Log (In tasks.md or separate tracking)
## Implementation Log

### TDD Cycle Progress
#### [Date] - Cycle 1 Complete
- Tests Written: `calculateTotal(), validateRating()`
- Code Changes: Implemented Review model
- Refactoring: Extracted validation logic
- Coverage: 92%

### Architecture Decisions Made
#### [Date] - Decision: Repository Pattern
- Context: Multiple data sources possible
- Decision: Use Repository abstraction
- Trade-offs: Extra layer but flexible persistence
- ADR Reference: ADR-001
3. HOOKS (.kiro/hooks/*.kiro)
Purpose: Automated workflow enforcement and quality gates

Common Hook Types:

Hook	Purpose	Prevents
test-first-enforcement.kiro	Enforce TDD	Saving production code without tests
pattern-recognition.kiro	Learn from mistakes	Suggests new rules from recurring issues
architecture-tracker.kiro	Consistency	Architectural pattern violations
documentation-generator.kiro	Auto-update docs	Stale documentation
regression-detection.kiro	Test coverage	Uncovered critical paths
Format: .kiro extension (KIRO-specific format, typically contains rules/logic for automation)

4. STEERING FRONTMATTER CONTROLS
---
inclusion: always        # Applied to ALL files
---
---
inclusion: fileMatch
fileMatchPattern: "**/*.{test,spec}.{js,ts,tsx,jsx}"
---
# Testing Standards (only applied to test files)
---
inclusion: manual
---
# Custom rule (must be explicitly referenced to apply)
5. ADR TEMPLATE (In Steering or Specs)
# ADR-XXXX: [Title]

Date: YYYY-MM-DD
Status: [Proposed | Accepted | Deprecated | Superseded]
Deciders: [Names]

## Context
The problem or situation that necessitates the decision

## Decision
The chosen solution

## Consequences

### Positive
- Benefit 1
- Benefit 2

### Negative
- Risk 1
- Risk 2

### Mitigation
- How to address risks

## Implementation Notes
Technical details for future developers
6. MCP INTEGRATION (.kiro/settings/mcp.json)
{
  "servers": [
    {
      "name": "github-mcp",
      "command": "node",
      "args": ["./mcp-github-server.js"]
    },
    {
      "name": "jira-mcp",
      "command": "python",
      "args": ["./mcp-jira-server.py"]
    }
  ]
}
Connects external tools via Model Context Protocol for enhanced agent capabilities.

7. KIRO WORKFLOW COMMANDS
KIRO uses structured prompts (from wirelessr/kiro-workflow-prompts):

/createSpec → Generates requirements.md (approval-gated)
/design → Generates design.md (approval-gated)
/createTask → Generates tasks.md (ready-signal)
/executeTask → Executes tasks from tasks.md (interactive)
/commit → Professional Git commits
/prReview → Code review analysis
Key Pattern: Mandatory context gathering, comprehension verification, explicit approval gates between phases.

8. NAMING & CONVENTIONS
Item	Format	Example
Feature directory	kebab-case	.kiro/specs/product-review-system/
Unit tests	Name.test.ts	Review.test.ts
Integration tests	Name.integration.test.ts	ReviewAPI.integration.test.ts
E2E tests	Flow.e2e.test.ts	UserReview.e2e.test.ts
Steering files	Descriptive kebab-case	tdd-methodology.md, api-style-guide.md
9. QUALITY STANDARDS IN STEERING
# From code-quality-gates.md
Test Coverage:
  - Minimum: 90% line coverage
  - Critical: 100% coverage for business logic
  - Branch: > 85%

Complexity Metrics:
  - Cyclomatic: < 10 per method
  - Method length: < 30 lines
  - Class methods: < 20
  - Duplicate code: No blocks > 6 lines

Performance:
  - Unit tests: < 100ms each
  - Integration tests: < 1 second each
  - Full suite: < 30 seconds
10. REFERENCE IMPLEMENTATIONS
Repository	Purpose	Stars	Key Content
kirodotdev/Kiro	Official KIRO IDE	3.4k	Example .kiro/specs/github-issue-automation/
5capeg0at/kiro-tdd-workflow-files	Complete TDD setup	13	Full .kiro/ structure, all steering files, hooks templates
wirelessr/kiro-workflow-prompts	Workflow prompts	51	/createSpec, /design, /createTask, /executeTask prompt templates
gotalab/cc-sdd	Spec-driven workflows	-	Claude Code, Copilot, Cursor, Codex integration
11. KEY npm PACKAGES
kiro-spec-engine – CLI for spec-driven development (with steering rules engine)
kspec – Spec-driven workflow tool (2.1k downloads)
konductor – Spec orchestrator for Kiro CLI (MCP server)
kiro-memory-skill – Persistent memory system (3.2.0)
skillpull – Sync AI agent skills to KIRO, Claude, Codex, Cursor
WORKSPACE STATUS
✅ Research Complete – No KIRO files currently in workspace

To Create KIRO Configuration for Your Project:

Create .kiro/ at project root
Add steering/ files with project guidelines
Create specs/[feature-name]/ with requirements.md, design.md, tasks.md
Add hooks/ for automation rules (.kiro files)
Create settings/mcp.json for external tools
All findings saved to session memory for reference.