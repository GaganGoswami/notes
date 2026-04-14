1. AGENTS.md File Format & Specifications
Primary File: AGENTS.md (default project documentation)

Key Files:

AGENTS.md - Main project-level instructions
AGENTS.override.md - Local override (preferred over AGENTS.md when both exist)
Fallback files (configurable, e.g., EXAMPLE.md)
File Scope & Hierarchy:

Scope covers the entire directory tree rooted at the folder containing AGENTS.md
Files can appear anywhere in repository (not limited to root)
When multiple AGENTS.md files exist at different levels, the deepest nested file takes precedence
Direct system/developer/user instructions override any AGENTS.md content
Contents are hierarchically concatenated from project root to current working directory (CWD)
2. AGENTS.md Specification Content
From the docs/agents_md.md and protocol specifications:

Typical AGENTS.md Sections:

Repository Guidelines (title)
Project Structure & Module Organization - directory layout, source code locations
Build, Test, and Development Commands - key commands (npm test, make build)
Coding Style & Naming Conventions - indentation, language preferences, naming patterns
Instructions format for agent behavior
Example content structure from the initiative prompt:

Generate a file named AGENTS.md that serves as a contributor guide for this repository.
Follow the outline, but adapt as needed — add sections if relevant, and omit those that do not apply.

Document Requirements:
- Title: "Repository Guidelines"
- Use Markdown headings (#, ##, etc.)
- Keep document concise: 200-400 words optimal
- Provide examples (commands, directory paths, naming patterns)
- Professional, instructional tone
3. Configuration Through TOML Files
config.toml structure:

[agents]
# Define agent roles with descriptions and config
role_name = {
    description = "Human-facing role documentation",
    config_file = "path/to/role-config.toml",  # Relative paths resolved from config.toml
    nickname_candidates = ["nickname1", "nickname2"]
}
Agent Role File Structure:

description - Required unless supplied from agent role file
config_file - Path to role-specific config layer
nickname_candidates - Candidate nicknames for spawned agents
developer_instructions - Role-specific developer instructions
4. Agent Roles & Spawning
Default Role Name: DEFAULT_ROLE_NAME (used when no type specified)

AgentRoleToml Structure:

description (optional in role file, required elsewhere)
config_file (path to role-specific config)
nickname_candidates (list of nickname options)
Spawn Agent Tool Schema:

{
  "message": "Initial plain-text task for the new agent",
  "agent_type": "Optional type name (built-in or user-defined)",
  "model": "Optional model override",
  "reasoning_effort": "Optional reasoning effort override",
  "fork_turns": "Optional number of turns to fork"
}
Available roles are built from both:

Built-in roles
User-defined roles (from config)
5. Skills System
SKILL.md files for detailed implementations:

Located at .codex/skills/ directory
Used to preserve context window in large systems
Each skill has:
SKILL.md - Skill documentation
agents/openai.yaml - Skill configuration
Skills Initialization (from init_skill.py):

SKILL.md created with standardized template
agents/openai.yaml for tool definitions
Resource directories created as needed
6. Instruction Hierarchy & Merging
Instruction Precedence (highest to lowest):

Direct system/developer/user prompt instructions
AGENTS.override.md (local override)
AGENTS.md (project-level)
Base instructions
Instruction Separation:

When both system instructions AND project doc present: concatenated with separator:
{INSTRUCTIONS}

--- project-doc ---

{PROJECT_DOC}
Message Format:

Instructions wrapped with markers:
Start: # AGENTS.md instructions for {directory}
End: </INSTRUCTIONS>
7. Hierarchical Agents Message
When child_agents_md feature flag enabled in [features] of config.toml:

Codex appends guidance about AGENTS.md scope and precedence
Guidance appended to user instructions message
Emitted even when no AGENTS.md present
Message appears in hierarchical_agents_message.md
Hierarchical Agents Guidance:

Each AGENTS.md governs its entire directory subtree
For every file touched, comply with every applicable AGENTS.md scope
Naming conventions and style rules restricted to code within scope (unless explicitly stated otherwise)
When two AGENTS.md files disagree: deeper in directory structure wins
8. Discovery & File Resolution
Project Doc Filename Constants:

pub const DEFAULT_PROJECT_DOC_FILENAME: &str = "AGENTS.md";
pub const LOCAL_PROJECT_DOC_FILENAME: &str = "AGENTS.override.md";
Discovery Process:

Search from project root to CWD
Prefer AGENTS.override.md over AGENTS.md
Support configured fallback files
AGENTS.md always preferred over fallbacks
Respects project root markers (e.g., .git, .gitignore)
9. Default Base Instructions Reference
From protocol/src/prompts/base_instructions/default.md:

Key Agent Behavior Sections:

Responsiveness - Preamble messages for context exploration
Planning - Usage guidelines for multi-step tasks
Task Execution - How to approach tasks
Validating Work - Quality checking procedures
Ambition vs. Precision - Scope balancing
Sharing Progress Updates - Status communication
Final Message Structure - Output formatting guidelines
File Reference Standards in AGENTS.md:

Wrap file paths in inline code (backticks)
Use format: path/to/file or path/to/file:line[:column]
Support absolute, relative, a/b diff prefixes, or bare filenames
Line numbers: 1-based (optional: :line[:column] or #Lline[Ccolumn])
10. Environment-Level Instructions
Environment Integration:

Config::instructions - Base system instructions
Environment filesystem access for doc loading
codex_exec_server::Environment - Runtime environment
Respects environment variables for customization
Project Root Markers:

.git directory
.gitignore file
Custom markers from config.toml
11. Documentation & Remote References
Official Documentation: developers.openai.com/codex/guides/agents-md

Repository Structure:

docs/agents_md.md - Main documentation
.codex/skills/ - Project skills directory
Test suites: core/tests/suite/agents_md.rs
Summary: Full Customization Pattern
project-root/
├── AGENTS.md                          # Primary custom instructions
├── AGENTS.override.md                 # Local overrides (optional)
├── config.toml                        # Config with [agents] roles section
├── .codex/
│   └── skills/                        # Skill implementations
│       └── skill-name/
│           ├── SKILL.md               # Skill documentation
│           └── agents/openai.yaml     # Skill configuration
└── nested/project/
    └── AGENTS.md                      # (Overrides parent AGENTS.md scope)
Codex CLI Priority & Execution:

Reads project structure from CWD up to root markers
Loads all AGENTS.md files in hierarchy
Merges with base + system instructions
Provides to agent with proper markup (# AGENTS.md instructions for {dir})
Agents follow hierarchical rules: deeply-nested AGENTS.md override higher-level ones
Human instructions (via chat) > AGENTS.override.md > AGENTS.md > Base Instructions