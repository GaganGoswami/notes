# Claude Code Customization System - Complete Research

## Key Findings

### 1. Directory Structure
- **`.claude/` directory** - Project-level configuration (shared with team)
- **`~/.claude/` directory** - User-level configuration (personal, all projects)
- **Managed policy locations:**
  - macOS: `/Library/Application Support/ClaudeCode/`
  - Linux/WSL: `/etc/claude-code/`
  - Windows: `C:\Program Files\ClaudeCode\`

### 2. CLAUDE.md File Format

**Locations (priority order):**
1. `./CLAUDE.md` or `./.claude/CLAUDE.md` (project-level, shared)
2. `~/.claude/CLAUDE.md` (user-level, personal)
3. `./CLAUDE.local.md` (project-local, not committed to git)
4. Managed policy: `/path/to/system/CLAUDE.md`

**Format:** Plain markdown, loaded at session start
**Load behavior:** Concatenated (not overriding) - all found files merge
**Scope hierarchy:** Local > Project > User > Organization
**Size consideration:** Ideal under 200 lines; first 200 lines/25KB of CLAUDE.md loaded per session

**Import syntax:** `@path/to/file` can import additional files

### 3. Settings Files (JSON Configuration)

**File locations:**
- `~/.claude/settings.json` - User settings (all projects)
- `.claude/settings.json` - Project settings (shared via git)
- `.claude/settings.local.json` - Local overrides (gitignored)
- `~/.claude.json` - Global preferences (themes, editor mode, MCP servers)
- `.mcp.json` - Project-scoped MCP servers

**Settings precedence (high to low):**
1. Managed settings (server, MDM, file-based)
2. Command line arguments
3. Local project settings (`.claude/settings.local.json`)
4. Project settings (`.claude/settings.json`)
5. User settings (`~/.claude/settings.json`)

### 4. .claude/ Directory Structure

```
.claude/
├── CLAUDE.md                      # Project instructions
├── settings.json                  # Shared project settings
├── settings.local.json            # Local personal settings (gitignored)
├── agents/                        # Custom subagents (.md files)
│   └── agent-name.md
├── rules/                         # Scoped instruction rules
│   ├── code-style.md
│   ├── testing.md
│   └── frontend/
├── commands/                      # Custom slash commands
│   └── command-name.md
├── CLAUDE.local.md                # Local personal instructions (gitignored)
└── hookify.*.local.md             # Plugin configuration files
```

### 5. Rules Organization (`.claude/rules/`)

**Features:**
- Modular instruction files (one topic per file)
- Path-specific rules using YAML frontmatter `paths:` field
- Recursively discovered (subdirectories supported)
- Loaded on demand (conditional to matching files)

**Path-specific rule example:**
```yaml
---
paths:
  - "src/api/**/*.ts"
  - "tests/**/*.test.ts"
---

# Rules for these files...
```

### 6. Plugins System

**Plugin manifest location:** `.claude-plugin/plugin.json`

**Plugin structure:**
```
plugin-name/
├── .claude-plugin/
│   └── plugin.json            # Required manifest
├── commands/                  # Slash commands
│   └── name.md
├── agents/                    # Subagents
│   └── agent-name.md
├── skills/                    # Skills (auto-discoverable)
│   └── skill-name/
│       └── SKILL.md
├── hooks/
│   └── hooks.json             # Event handlers
├── .mcp.json                  # MCP servers
└── scripts/                   # Helper scripts
```

**Plugin.json fields:**
- `name` (required)
- `version`, `description`, `author`
- Custom paths for components (supplement defaults)
- `commands`, `agents`, `skills`, `hooks`, `mcpServers`

**Auto-discovery:** All `.md` files in standard directories auto-load
**Namespace syntax:** `/command (plugin:name)` or `agent:subdir:name`

### 7. Agents/Subagents Format

**File location:** `.claude/agents/agent-name.md` or `plugin/agents/agent-name.md`

**Front matter structure:**
```markdown
---
name: agent-identifier
description: When to use this agent. Examples:

<example>
Context: [situation]
user: "[user request]"
assistant: "[How assistant responds]"
<commentary>
[Why this agent should trigger]
</commentary>
</example>

model: inherit
color: blue
tools: ["Read", "Write", "Grep"]  # Optional tool restrictions
---

You are [agent role]...

**Your Core Responsibilities:**
1. [Responsibility 1]
2. [Responsibility 2]

**Analysis Process:**
[Step-by-step workflow]

**Output Format:**
[What to return]
```

### 8. Skills Format

**File location:** `.claude/skills/skill-name/SKILL.md` or `plugin/skills/skill-name/SKILL.md`

**Front matter:**
```markdown
---
name: Skill Name
description: When to use this skill; trigger phrases
version: 1.0.0
---

Skill instructions and guidance...
```

**Auto-invocation:** Auto-activates based on trigger phrases
**Subdirectories:** Skills can include scripts/, references/, examples/

### 9. Hooks System

**Configuration formats:**
- `hooks/hooks.json` (in plugins)
- Inline in `plugin.json`

**Hook types:** PreToolUse, PostToolUse, Stop, SubagentStop, SessionStart, SessionEnd, UserPromptSubmit, PreCompact, Notification

**Hook configuration example:**
```json
{
  "PreToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "command",
      "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/validate.sh",
      "timeout": 30
    }]
  }]
}
```

### 10. MCP Configuration

**Locations:**
- User: `~/.claude.json` (MCP servers)
- Project: `.mcp.json` (project-specific MCP servers)

**Can be configured in:**
- `settings.json` (via `mcpServers` field)
- Separate `.mcp.json` file

### 11. Plugin Settings Files

**Pattern:** `.claude/plugin-name.local.md`

**Format:** YAML frontmatter + markdown body
```markdown
---
setting1: value1
setting2: "value with spaces"
---

Markdown body content...
```

**Parsing patterns (bash):**
```bash
# Extract frontmatter
FRONTMATTER=$(sed -n '/^---$/,/^---$/{ /^---$/d; p; }' "$FILE")

# Read field
VALUE=$(echo "$FRONTMATTER" | grep '^field:' | sed 's/field: *//')

# Extract body
BODY=$(awk '/^---$/{i++; next} i>=2' "$FILE")
```

### 12. Environment Variables

**Key variables:**
- `CLAUDE_PLUGIN_ROOT` - Plugin root directory path (for portable references)
- `CLAUDE_PROJECT_DIR` - Project directory
- Custom environment variables via `env` setting in `settings.json`

### 13. Auto Memory

**Default behavior:** Enabled, stores in `~/.claude/projects/<project>/memory/`

**Structure:**
```
memory/
├── MEMORY.md          # Index (first 200 lines/25KB loaded per session)
├── debugging.md       # Topic files (loaded on demand)
├── api-conventions.md
└── ...
```

**Configuration:** 
- Toggle via `/memory` command or `autoMemoryEnabled` setting
- Custom location via `autoMemoryDirectory` setting

### 14. Key Configuration Settings

**Common settings in `settings.json`:**
- `permissions` - Tool access rules (allow, deny, ask)
- `env` - Environment variables
- `model` - Default model override
- `agent` - Run main thread as specific subagent
- `hooks` - Custom lifecycle hooks
- `sandbox` - Sandboxing configuration
- `outputStyle` - Configure output style
- `autoMemoryEnabled` - Toggle auto memory
- `autoUpdatesChannel` - Release channel ("stable" or "latest")
- `enabledPlugins` - Plugin enable/disable status

### 15. Subagent Configuration

**Locations:**
- User: `~/.claude/agents/`
- Project: `.claude/agents/`

**Invocation patterns:**
- Manual: `/agent:agent-name`
- Automatic: Claude selects based on context
- System-wide: Via settings `agent` field

### 16. Scope System

**Applies to:**
| Item | User | Project | Local |
|------|------|---------|-------|
| Settings | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| Subagents | `~/.claude/agents/` | `.claude/agents/` | None |
| CLAUDE.md | `~/.claude/CLAUDE.md` | `CLAUDE.md` or `.claude/CLAUDE.md` | `CLAUDE.local.md` |
| MCP servers | `~/.claude.json` | `.mcp.json` | `~/.claude.json` (per-project) |

### 17. File Discovery Patterns

**CLAUDE.md loading:**
- Walks up directory tree from working directory
- Loads all found `CLAUDE.md` and `CLAUDE.local.md` files
- Subdirectory files load on-demand when opening those files
- Exclude patterns: `claudeMdExcludes` in settings

**Components auto-discovery:**
- Commands: All `.md` files in `commands/` directory
- Agents: All `.md` files in `agents/` directory
- Skills: All `SKILL.md` files in subdirectories of `skills/`
- Takes effect on plugin enable, next session reload

### 18. Import Syntax

**In CLAUDE.md files:**
```markdown
See @README for overview and @package.json for commands.
@docs/workflows.md for detailed workflows
@~/.claude/my-instructions.md for personal notes
```

**Relative paths:** Resolve relative to containing file
**Maximum depth:** 5 hops of imports

### 19. Path Prefixes in Settings

**Sandbox settings paths:**
- `/` - Absolute from filesystem root
- `~/` - Relative to home directory
- `./` or no prefix - Relative to project root (or `~/.claude` for user settings)
- `//path` - (legacy) Absolute path

---

## Implementation Best Practices

1. **CLAUDE.md:** Keep under 200 lines, use specific concrete instructions
2. **Rules:** Use `.claude/rules/` for modular, path-scoped instructions
3. **Settings:** Use JSON schema validation in editor
4. **Plugins:** Leverage `${CLAUDE_PLUGIN_ROOT}` for portable paths
5. **Scope:** Use more specific scopes (local > project > user)
6. **Size:** Monitor context consumption with `/status` command
7. **Symlinks:** Support symlinks in `.claude/rules/` for sharing rules
