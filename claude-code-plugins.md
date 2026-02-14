# Claude Code Plugin Architecture Reference

Comprehensive reference for building Claude Code plugins, MCP servers, hooks, skills, subagents, and commands. Based on official documentation at [code.claude.com/docs/en/plugins-reference](https://code.claude.com/docs/en/plugins-reference), the [official plugin repository](https://github.com/anthropics/claude-code/blob/main/plugins/README.md), and analysis of the working `claude-mem` plugin.

---

## 1. Plugin Structure

### Directory Layout

A plugin is a directory containing one or more components. Claude Code discovers components by convention (default directories) and optionally via a manifest.

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json           # Manifest (optional, only name required)
├── commands/                  # Slash commands (legacy, use skills/ for new)
│   └── my-command.md
├── skills/                    # Skills with SKILL.md entrypoints
│   └── my-skill/
│       ├── SKILL.md           # Required entrypoint
│       ├── reference.md       # Optional supporting files
│       └── scripts/
├── agents/                    # Subagent definitions
│   └── my-agent.md
├── hooks/                     # Hook configurations
│   └── hooks.json             # Main hook config
├── modes/                     # Custom modes (JSON configs)
├── scripts/                   # Hook and utility scripts
├── .mcp.json                  # MCP server definitions
├── .lsp.json                  # LSP server configurations
├── CLAUDE.md                  # Plugin-level context (injected when enabled)
├── package.json               # Node.js dependencies (if any)
└── README.md
```

### Component Location Defaults

| Component    | Default Location              | Purpose                              |
|-------------|-------------------------------|--------------------------------------|
| Manifest    | `.claude-plugin/plugin.json`  | Plugin metadata and config           |
| Commands    | `commands/`                   | Slash command markdown files         |
| Skills      | `skills/`                     | Skill directories with `SKILL.md`    |
| Agents      | `agents/`                     | Subagent markdown files              |
| Hooks       | `hooks/hooks.json`            | Hook configuration                   |
| MCP servers | `.mcp.json`                   | MCP server definitions               |
| LSP servers | `.lsp.json`                   | Language server configurations       |

Components go at the **plugin root**, not inside `.claude-plugin/`. Only `plugin.json` belongs in `.claude-plugin/`.

### Plugin Manifest (plugin.json)

The manifest is optional. If omitted, Claude Code auto-discovers components and derives the name from the directory. When present, only `name` is required.

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "Brief plugin description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "commands": ["./custom/commands/special.md"],
  "agents": "./custom/agents/",
  "skills": "./custom/skills/",
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "outputStyles": "./styles/",
  "lspServers": "./.lsp.json"
}
```

Custom paths **supplement** default directories -- they do not replace them. All paths must be relative and start with `./`.

### Plugin Discovery and Loading

Claude Code loads plugins from these sources:

1. **Marketplace plugins** -- installed via `claude plugin install <name>@<marketplace>` or `/plugin` UI
2. **Local directory** -- loaded via `claude --plugin-dir <path>` for the duration of a session
3. **Settings-registered** -- listed in `enabledPlugins` in settings files

Plugins are copied to a **cache directory** for security. Files outside the plugin root are not accessible unless symlinked into the plugin directory before installation.

### Plugin Installation Scopes

| Scope     | Settings File                    | Use Case                               |
|-----------|----------------------------------|----------------------------------------|
| `user`    | `~/.claude/settings.json`       | Personal plugins, all projects         |
| `project` | `.claude/settings.json`         | Team plugins, committed to VCS         |
| `local`   | `.claude/settings.local.json`   | Project-specific, gitignored           |
| `managed` | `managed-settings.json`         | Admin-deployed, read-only              |

### Environment Variables

- `${CLAUDE_PLUGIN_ROOT}` -- absolute path to the plugin directory. Use in hooks, MCP server configs, and scripts for portable paths.
- `$CLAUDE_PROJECT_DIR` -- the project root directory.
- `$CLAUDE_CODE_REMOTE` -- set to `"true"` in remote web environments.
- `$CLAUDE_ENV_FILE` -- (SessionStart hooks only) path to write persistent env vars.

### Example: claude-mem Plugin Structure

The working `claude-mem` plugin at `~/.claude-mem/plugin/` demonstrates a real plugin:

```
~/.claude-mem/plugin/
├── .claude-plugin/plugin.json    # name: "claude-mem", version: "9.0.5"
├── commands/
│   ├── do.md                     # Orchestrator command
│   └── make-plan.md              # Planning command
├── skills/
│   └── mem-search/SKILL.md       # Memory search skill
├── hooks/
│   └── hooks.json                # SessionStart, PostToolUse, Stop, UserPromptSubmit
├── modes/
│   └── code.json (+ translations)
├── scripts/
│   ├── mcp-server.cjs            # MCP server (stdio, ~340KB bundled)
│   ├── worker-service.cjs        # Background worker (~1.7MB bundled)
│   ├── context-generator.cjs     # Context injection script
│   └── worker-wrapper.cjs        # Worker lifecycle wrapper
├── .mcp.json                     # Registers mcp-search server
├── CLAUDE.md                     # Plugin context (auto-generated)
└── package.json                  # Runtime deps declaration
```

---

## 2. Hooks

Hooks are user-defined shell commands, LLM prompts, or agent verifiers that execute automatically at specific lifecycle points. Unlike skills and MCP tools (which Claude decides to use), hooks are **system-facing** -- Claude has no say in whether they run.

### Hook Types

| Type      | Description                                    | Key Fields                |
|-----------|------------------------------------------------|---------------------------|
| `command` | Execute a shell command or script              | `command`, `async`        |
| `prompt`  | Single-turn LLM evaluation (yes/no decision)   | `prompt`, `model`         |
| `agent`   | Multi-turn subagent with tool access            | `prompt`, `model`         |

### Hook Events

| Event                | When It Fires                              | Can Block? | Matcher Field      |
|---------------------|--------------------------------------------|------------|---------------------|
| `SessionStart`      | Session begins or resumes                  | No         | startup/resume/clear/compact |
| `UserPromptSubmit`  | User submits prompt, before processing     | Yes        | (none)              |
| `PreToolUse`        | Before a tool call executes                | Yes        | tool name           |
| `PermissionRequest` | Permission dialog shown                    | Yes        | tool name           |
| `PostToolUse`       | After a tool call succeeds                 | No         | tool name           |
| `PostToolUseFailure`| After a tool call fails                    | No         | tool name           |
| `Notification`      | Claude Code sends notification             | No         | notification type   |
| `SubagentStart`     | Subagent spawned                           | No         | agent type          |
| `SubagentStop`      | Subagent finishes                          | Yes        | agent type          |
| `Stop`              | Claude finishes responding                 | Yes        | (none)              |
| `TeammateIdle`      | Agent team teammate about to go idle       | Yes        | (none)              |
| `TaskCompleted`     | Task marked as completed                   | Yes        | (none)              |
| `PreCompact`        | Before context compaction                  | No         | manual/auto         |
| `SessionEnd`        | Session terminates                         | No         | exit reason         |

### Hook Configuration Format

Hooks live in JSON settings files or in `hooks/hooks.json` inside a plugin. Three levels of nesting: event -> matcher group -> handler array.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Evaluate if all tasks complete: $ARGUMENTS",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

Plugin hook files can include a top-level `description` field:

```json
{
  "description": "Claude-mem memory system hooks",
  "hooks": { ... }
}
```

### Hook Input (stdin JSON)

All hooks receive common fields via stdin:

```json
{
  "session_id": "abc123",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

Event-specific fields vary. For example:
- `PreToolUse` / `PostToolUse`: `tool_name`, `tool_input`, `tool_use_id`
- `PostToolUse`: adds `tool_response`
- `Stop`: adds `stop_hook_active` (boolean, true if already continuing from a stop hook)
- `UserPromptSubmit`: adds `prompt`
- `SessionStart`: adds `source`, `model`
- `TaskCompleted`: adds `task_id`, `task_subject`, `task_description`

### Hook Output (exit codes + stdout JSON)

**Exit 0** -- success. Stdout parsed as JSON for structured control.

**Exit 2** -- blocking error. Stderr fed back to Claude. Effect depends on event:
- `PreToolUse`: blocks the tool call
- `UserPromptSubmit`: blocks and erases the prompt
- `Stop` / `SubagentStop`: prevents stopping, continues conversation
- `TaskCompleted` / `TeammateIdle`: prevents the action, stderr as feedback
- `PostToolUse` / `Notification` / `SessionStart`: stderr shown (non-blocking)

**Other exit codes** -- non-blocking error, stderr shown in verbose mode.

### JSON Output Fields

```json
{
  "continue": true,
  "stopReason": "message when continue=false",
  "suppressOutput": false,
  "systemMessage": "warning shown to user"
}
```

### PreToolUse Decision Control

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Destructive command blocked",
    "updatedInput": { "command": "safer-command" },
    "additionalContext": "Extra info for Claude"
  }
}
```

`permissionDecision`: `"allow"` (bypass permissions), `"deny"` (block), `"ask"` (prompt user).

### Stop / SubagentStop Decision Control

```json
{
  "decision": "block",
  "reason": "Tests haven't passed yet"
}
```

### SessionStart Context Injection

Stdout text (or `additionalContext` in JSON) is added as context Claude can see:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Injected context string"
  }
}
```

SessionStart hooks can also persist environment variables via `$CLAUDE_ENV_FILE`:

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

### Async Hooks

Set `"async": true` on command hooks to run in the background without blocking Claude. Only available for `type: "command"`. Async hooks cannot return decisions (the action has already proceeded). Output is delivered on the next conversation turn.

```json
{
  "type": "command",
  "command": "/path/to/run-tests.sh",
  "async": true,
  "timeout": 120
}
```

### Prompt-Based Hooks

Instead of a shell command, send the hook context to an LLM for evaluation:

```json
{
  "type": "prompt",
  "prompt": "Check if all tasks complete: $ARGUMENTS. Respond with JSON: {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
  "model": "haiku",
  "timeout": 30
}
```

The LLM must respond with `{"ok": true}` to allow or `{"ok": false, "reason": "..."}` to block.

### Agent-Based Hooks

Like prompt hooks but with multi-turn tool access (Read, Grep, Glob). Up to 50 turns:

```json
{
  "type": "agent",
  "prompt": "Verify all unit tests pass. $ARGUMENTS",
  "timeout": 120
}
```

### Matcher Patterns

Matchers are regex strings. Use `"*"`, `""`, or omit to match all. Examples:
- `"Bash"` -- exact tool name
- `"Write|Edit"` -- either tool
- `"mcp__memory__.*"` -- all tools from the memory MCP server
- `"mcp__.*__write.*"` -- any write tool from any MCP server

### Hook Handler Common Fields

| Field           | Required | Description                                               |
|-----------------|----------|-----------------------------------------------------------|
| `type`          | Yes      | `"command"`, `"prompt"`, or `"agent"`                     |
| `timeout`       | No       | Seconds. Defaults: 600 (command), 30 (prompt), 60 (agent) |
| `statusMessage` | No       | Custom spinner message while running                      |
| `once`          | No       | Run only once per session (skills only)                   |
| `command`       | Yes*     | Shell command (command hooks only)                        |
| `async`         | No       | Run in background (command hooks only)                    |
| `prompt`        | Yes*     | Prompt text with `$ARGUMENTS` placeholder (prompt/agent)  |
| `model`         | No       | Model for evaluation (prompt/agent only)                  |

### Hooks in Skills and Agents

Hooks can be defined in skill/agent YAML frontmatter, scoped to the component lifecycle:

```yaml
---
name: secure-operations
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

For subagents, `Stop` hooks in frontmatter are auto-converted to `SubagentStop`.

### Hook Locations (Scope)

| Location                          | Scope                    | Shareable? |
|-----------------------------------|--------------------------|------------|
| `~/.claude/settings.json`        | All your projects        | No         |
| `.claude/settings.json`          | Single project           | Yes (VCS)  |
| `.claude/settings.local.json`    | Single project           | No         |
| Managed policy                    | Organization-wide        | Yes        |
| Plugin `hooks/hooks.json`        | When plugin enabled      | Yes        |
| Skill/Agent frontmatter          | While component active   | Yes        |

Hooks are snapshot at startup. Mid-session modifications require review via `/hooks` menu.

### Real-World Example: claude-mem Hook Configuration

```json
{
  "description": "Claude-mem memory system hooks",
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          { "type": "command", "command": "node \"${CLAUDE_PLUGIN_ROOT}/scripts/smart-install.js\"", "timeout": 300 },
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" start", "timeout": 60 },
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" hook claude-code context", "timeout": 60 },
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" hook claude-code user-message", "timeout": 60 }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" start", "timeout": 60 },
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" hook claude-code session-init", "timeout": 60 }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" start", "timeout": 60 },
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" hook claude-code observation", "timeout": 120 }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" start", "timeout": 60 },
          { "type": "command", "command": "bun \"${CLAUDE_PLUGIN_ROOT}/scripts/worker-service.cjs\" hook claude-code summarize", "timeout": 120 }
        ]
      }
    ]
  }
}
```

Pattern: each hook first ensures the background worker is running (`start`), then delegates to the worker with a specific action.

---

## 3. MCP Servers

The [Model Context Protocol](https://modelcontextprotocol.io/introduction) is an open standard for AI-tool integrations. MCP servers expose tools, resources, and prompts to Claude Code through a standardized interface.

### Transport Types

| Transport          | Description                         | Config Key  |
|--------------------|-------------------------------------|-------------|
| `stdio`            | Local process, stdin/stdout         | `command`   |
| `http`             | Remote HTTP (streamable, preferred) | `url`       |
| `sse`              | Remote Server-Sent Events (legacy)  | `url`       |

### Plugin MCP Configuration (.mcp.json)

Place at the plugin root:

```json
{
  "mcpServers": {
    "mcp-search": {
      "type": "stdio",
      "command": "${CLAUDE_PLUGIN_ROOT}/scripts/mcp-server.cjs"
    }
  }
}
```

Or inline in `plugin.json`:

```json
{
  "name": "my-plugin",
  "mcpServers": {
    "plugin-api": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/api-server",
      "args": ["--port", "8080"],
      "env": {
        "DB_PATH": "${CLAUDE_PLUGIN_ROOT}/data"
      }
    }
  }
}
```

### Adding MCP Servers via CLI

```bash
# HTTP (recommended for remote)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# SSE (deprecated, use HTTP)
claude mcp add --transport sse asana https://mcp.asana.com/sse

# stdio (local process)
claude mcp add --transport stdio --env API_KEY=xxx myserver -- npx -y @company/mcp-server

# From JSON
claude mcp add-json weather '{"type":"http","url":"https://api.weather.com/mcp"}'

# Manage
claude mcp list
claude mcp get github
claude mcp remove github
```

### MCP Server Scopes

| Scope     | Storage                              | Use Case                          |
|-----------|--------------------------------------|-----------------------------------|
| `local`   | `~/.claude.json` (per-project)       | Personal, project-specific        |
| `project` | `.mcp.json` (project root, VCS)      | Team-shared                       |
| `user`    | `~/.claude.json` (global)            | Personal, cross-project           |

### MCP Tool Naming Convention

MCP tools appear as `mcp__<server>__<tool>`. For example:
- `mcp__memory__create_entities`
- `mcp__filesystem__read_file`

These names are used in hook matchers and permission rules.

### MCP Resources

MCP servers can expose resources referenced via `@` mentions:
```
> Analyze @github:issue://123
> Compare @postgres:schema://users with @docs:file://database/user-model
```

### Tool Search

When MCP tool definitions exceed 10% of the context window, Claude Code automatically switches to on-demand tool loading. Configurable via `ENABLE_TOOL_SEARCH`:
- `auto` (default) -- activates at 10% threshold
- `auto:5` -- custom 5% threshold
- `true` -- always enabled
- `false` -- disabled, all tools loaded upfront

### Environment Variable Expansion in .mcp.json

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

Syntax: `${VAR}` or `${VAR:-default}`.

---

## 4. Commands (Slash Commands)

Commands are markdown files that create `/slash-command` shortcuts. They are the legacy form of skills; new development should use skills instead.

### Location

- `~/.claude/commands/` -- personal, all projects
- `.claude/commands/` -- project-specific
- `<plugin>/commands/` -- plugin-provided

### File Format

A simple markdown file. The filename (without `.md`) becomes the command name.

```markdown
You are an ORCHESTRATOR.

Primary instruction: deploy subagents to execute *all* work for #$ARGUMENTS.

## Execution Protocol
1. Each phase uses fresh subagents
2. Assign one clear objective per subagent
3. Do not advance until verification passes
```

### Arguments and Substitution

- `$ARGUMENTS` -- all arguments passed after the command name
- `$ARGUMENTS[0]`, `$1` -- positional arguments
- `${CLAUDE_SESSION_ID}` -- current session ID

### YAML Frontmatter

Commands support the same frontmatter as skills:

```yaml
---
name: my-command
description: What this command does
disable-model-invocation: true
allowed-tools: Read, Grep, Glob
context: fork
agent: Explore
---
```

### Dynamic Context Injection

Use `!`command`` to run shell commands before content is sent to Claude:

```markdown
## Current state
- Branch: !`git branch --show-current`
- Status: !`git status --short`
```

---

## 5. Skills

Skills extend what Claude can do. They follow the [Agent Skills](https://agentskills.io) open standard. A skill is a directory with a `SKILL.md` entrypoint, optionally with supporting files.

### Skill Structure

```
my-skill/
├── SKILL.md           # Required entrypoint
├── reference.md       # Detailed reference (loaded on demand)
├── examples/
│   └── sample.md
└── scripts/
    └── helper.py
```

### SKILL.md Format

```yaml
---
name: explain-code
description: Explains code with visual diagrams and analogies
argument-hint: [filename]
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob
model: sonnet
context: fork
agent: Explore
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
---

When explaining code, always include:
1. **Start with an analogy**
2. **Draw a diagram** using ASCII art
3. **Walk through the code** step-by-step
4. **Highlight a gotcha**
```

### Frontmatter Fields

| Field                      | Required    | Description                                                |
|---------------------------|-------------|------------------------------------------------------------|
| `name`                    | No          | Display name (defaults to directory name)                  |
| `description`             | Recommended | What the skill does; Claude uses this for auto-invocation  |
| `argument-hint`           | No          | Hint for autocomplete (e.g., `[issue-number]`)             |
| `disable-model-invocation`| No          | `true` = manual-only invocation                            |
| `user-invocable`          | No          | `false` = hidden from `/` menu, Claude-only                |
| `allowed-tools`           | No          | Tools allowed without asking permission                    |
| `model`                   | No          | Model override when skill is active                        |
| `context`                 | No          | `fork` = run in isolated subagent                          |
| `agent`                   | No          | Subagent type for `context: fork` (Explore, Plan, etc.)    |
| `hooks`                   | No          | Lifecycle hooks scoped to this skill                       |
| `once`                    | No          | Hook runs only once per session (in hook config)           |

### Invocation Control

| Configuration                    | User Can Invoke | Claude Can Invoke |
|---------------------------------|-----------------|-------------------|
| (default)                        | Yes             | Yes               |
| `disable-model-invocation: true` | Yes             | No                |
| `user-invocable: false`          | No              | Yes               |

### Skill Scopes

| Location                                       | Scope                     |
|------------------------------------------------|---------------------------|
| `~/.claude/skills/<name>/SKILL.md`             | Personal, all projects    |
| `.claude/skills/<name>/SKILL.md`               | Project-specific          |
| `<plugin>/skills/<name>/SKILL.md`              | Plugin-provided           |
| Managed settings                                | Enterprise, all users     |

### Progressive Disclosure

Skill descriptions are loaded into context so Claude knows what is available. Full skill content only loads when invoked. This keeps context lightweight. The description budget is 2% of the context window (fallback: 16,000 chars). Configurable via `SLASH_COMMAND_TOOL_CHAR_BUDGET`.

---

## 6. Subagents

Subagents are specialized AI assistants with their own context windows, system prompts, tool access, and permissions. They run in isolation and return results to the parent conversation.

### Built-In Subagents

| Agent           | Model   | Tools          | Purpose                        |
|-----------------|---------|----------------|--------------------------------|
| Explore         | Haiku   | Read-only      | Codebase search and analysis   |
| Plan            | Inherit | Read-only      | Research for planning          |
| general-purpose | Inherit | All            | Complex multi-step tasks       |
| Bash            | Inherit | Bash           | Terminal commands               |

### Custom Subagent Definition

Markdown files with YAML frontmatter:

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after changes.
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
model: sonnet
permissionMode: default
maxTurns: 50
skills:
  - api-conventions
  - error-handling-patterns
memory: user
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
---

You are a senior code reviewer. When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately
```

### Subagent Frontmatter Fields

| Field             | Required | Description                                          |
|-------------------|----------|------------------------------------------------------|
| `name`            | Yes      | Unique identifier (lowercase, hyphens)               |
| `description`     | Yes      | When Claude should delegate to this agent            |
| `tools`           | No       | Allowed tools (inherits all if omitted)              |
| `disallowedTools` | No       | Tools to deny                                        |
| `model`           | No       | `sonnet`, `opus`, `haiku`, or `inherit`              |
| `permissionMode`  | No       | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan`, `delegate` |
| `maxTurns`        | No       | Max agentic turns before stopping                    |
| `skills`          | No       | Skills preloaded into context at startup             |
| `mcpServers`      | No       | MCP servers available to this agent                  |
| `hooks`           | No       | Lifecycle hooks scoped to this agent                 |
| `memory`          | No       | Persistent memory scope: `user`, `project`, `local`  |

### Subagent Scopes

| Location                   | Scope                | Priority |
|----------------------------|----------------------|----------|
| `--agents` CLI flag        | Current session      | Highest  |
| `.claude/agents/`          | Current project      | 2        |
| `~/.claude/agents/`        | All projects         | 3        |
| Plugin `agents/` directory | When plugin enabled  | Lowest   |

### CLI-Defined Subagents

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer",
    "prompt": "You are a senior code reviewer...",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

### Foreground vs Background Subagents

- **Foreground**: blocks main conversation, passes permission prompts through
- **Background**: runs concurrently, pre-approves permissions, auto-denies unapproved ones
- Press **Ctrl+B** to background a running task
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to disable

### Persistent Memory

When `memory` is set, the subagent gets a persistent directory:
- `user`: `~/.claude/agent-memory/<agent-name>/`
- `project`: `.claude/agent-memory/<agent-name>/`
- `local`: `.claude/agent-memory-local/<agent-name>/`

The subagent's system prompt includes `MEMORY.md` (first 200 lines) and instructions to maintain it.

---

## 7. Settings Integration

### Plugin Registration in settings.json

Plugins appear in `enabledPlugins`:

```json
{
  "enabledPlugins": {
    "context7@claude-plugins-official": true,
    "claude-mem@my-marketplace": true
  }
}
```

### Hook Registration in settings.json

Hooks can live directly in settings (user-level or project-level):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/.claude/hooks/validate.sh"
          }
        ]
      }
    ]
  }
}
```

### MCP Server Registration

MCP servers configured via CLI end up in `~/.claude.json` (local/user scope) or `.mcp.json` (project scope):

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    },
    "local-db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@company/db-server"],
      "env": { "DB_URL": "postgresql://..." }
    }
  }
}
```

### Permission Rules for Tools and Skills

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test:*)",
      "Skill(commit)",
      "mcp__plugin_context7_context7__resolve-library-id"
    ],
    "deny": [
      "Task(Explore)",
      "Skill(deploy *)"
    ]
  }
}
```

---

## 8. Building an MCP Server in Rust

The official Rust SDK for MCP is [rmcp](https://github.com/modelcontextprotocol/rust-sdk). An alternative is [rust-mcp-sdk](https://crates.io/crates/rust-mcp-sdk).

### Cargo.toml

```toml
[dependencies]
rmcp = { version = "0.15", features = ["server", "transport-io"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
schemars = "0.8"
```

### Feature Flags

| Feature                       | Description                        |
|-------------------------------|------------------------------------|
| `server`                      | Server functionality and tool system |
| `client`                      | Client features                    |
| `macros`                      | `#[tool]` macro (default)          |
| `transport-io`                | Async stdin/stdout support         |
| `transport-child-process`     | Child process transport            |
| `transport-async-rw`          | Low-level async I/O                |
| `transport-streamable-http-server` | HTTP server streaming         |
| `schemars`                    | JSON Schema generation             |
| `auth`                        | OAuth2 support                     |

### Complete Server Example

```rust
use rmcp::{
    ServerHandler, ServiceExt,
    handler::server::tool::ToolRouter,
    model::*,
    tool, tool_handler, tool_router,
    transport::stdio,
    ErrorData as McpError,
};
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Clone)]
pub struct MyServer {
    data: Arc<Mutex<Vec<String>>>,
    tool_router: ToolRouter<Self>,
}

#[tool_router]
impl MyServer {
    fn new() -> Self {
        Self {
            data: Arc::new(Mutex::new(Vec::new())),
            tool_router: Self::tool_router(),
        }
    }

    #[tool(description = "Search stored entries by keyword")]
    async fn search(
        &self,
        #[tool(param)]
        #[schemars(description = "Search query string")]
        query: String,
        #[tool(param)]
        #[schemars(description = "Maximum results to return")]
        limit: Option<usize>,
    ) -> Result<CallToolResult, McpError> {
        let data = self.data.lock().await;
        let limit = limit.unwrap_or(10);
        let results: Vec<_> = data.iter()
            .filter(|e| e.contains(&query))
            .take(limit)
            .collect();
        Ok(CallToolResult::success(vec![Content::text(
            serde_json::to_string(&results).unwrap(),
        )]))
    }

    #[tool(description = "Add a new entry")]
    async fn add_entry(
        &self,
        #[tool(param)]
        #[schemars(description = "The entry content to store")]
        content: String,
    ) -> Result<CallToolResult, McpError> {
        let mut data = self.data.lock().await;
        data.push(content.clone());
        Ok(CallToolResult::success(vec![Content::text(
            format!("Added entry: {}", content),
        )]))
    }
}

#[tool_handler]
impl ServerHandler for MyServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            instructions: Some("A data store with search and add tools".into()),
            capabilities: ServerCapabilities::builder()
                .enable_tools()
                .build(),
            ..Default::default()
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let service = MyServer::new()
        .serve(stdio())
        .await?;
    service.waiting().await?;
    Ok(())
}
```

### Structured Input with Aggregate Parameters

```rust
use rmcp::handler::server::wrapper::Parameters;
use schemars::JsonSchema;
use serde::{Serialize, Deserialize};

#[derive(Deserialize, JsonSchema)]
struct SearchParams {
    /// The search query
    query: String,
    /// Maximum number of results
    #[serde(default = "default_limit")]
    limit: usize,
    /// Filter by date range
    date_range: Option<String>,
}

fn default_limit() -> usize { 10 }

#[tool(name = "advanced_search", description = "Search with filters")]
async fn advanced_search(
    &self,
    #[tool(aggr)] params: Parameters<SearchParams>,
) -> Result<CallToolResult, McpError> {
    // params.0 is the deserialized SearchParams
    let query = &params.0.query;
    let limit = params.0.limit;
    // ...
}
```

### Integrating with Claude Code

Register the compiled binary as a stdio MCP server:

```bash
claude mcp add --transport stdio my-rust-server -- /path/to/my-server
```

Or in a plugin's `.mcp.json`:

```json
{
  "mcpServers": {
    "my-rust-server": {
      "type": "stdio",
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/my-server",
      "args": ["--db", "${CLAUDE_PLUGIN_ROOT}/data/store.db"]
    }
  }
}
```

---

## 9. Communication Patterns

### Hook-to-Worker Communication

The claude-mem plugin demonstrates a robust pattern: hooks communicate with a long-running background worker process via HTTP.

**Architecture:**

```
Claude Code Session
    |
    ├── SessionStart hook
    |   ├── Ensure worker running (start if not)
    |   ├── Request context generation
    |   └── Inject context into session
    |
    ├── PostToolUse hook (every tool call)
    |   ├── Ensure worker running
    |   └── Queue observation for async extraction
    |
    ├── UserPromptSubmit hook
    |   ├── Ensure worker running
    |   └── Initialize session tracking
    |
    └── Stop hook
        ├── Ensure worker running
        └── Generate session summary
```

**Worker lifecycle:**
1. Each hook first calls `worker-service.cjs start` to ensure the worker is running
2. The worker runs on a known port (e.g., `127.0.0.1:37777`)
3. A PID file (`worker.pid`) tracks the process
4. Hooks then call `worker-service.cjs hook claude-code <action>` to dispatch work

### HTTP-Based Communication

Hooks can POST to a local HTTP server:

```bash
#!/bin/bash
INPUT=$(cat)
curl -s -X POST http://127.0.0.1:37777/hook/observation \
  -H "Content-Type: application/json" \
  -d "$INPUT"
exit 0
```

### File-Based Communication

For simpler setups, hooks can write to shared files:

```bash
#!/bin/bash
INPUT=$(cat)
echo "$INPUT" >> /tmp/claude-hook-log.jsonl
exit 0
```

### Queue Patterns

The claude-mem plugin queues tool observations for async processing:
1. Hook receives tool call data via stdin
2. Hook posts to worker's queue endpoint
3. Worker processes queue asynchronously with an SDK agent (extraction, classification)
4. Results stored in SQLite + vector DB
5. Next session's SessionStart hook reads from DB to inject context

### Ensuring Worker Availability

Every hook starts with a worker health check. If the worker is not running, it starts it. This is idempotent and handles:
- First hook in a session (cold start)
- Worker crash recovery
- Multiple hooks racing to start (PID file guards)

---

## 10. Data Storage

### SQLite for Structured Data

The claude-mem plugin uses SQLite (`claude-mem.db`) for:
- Observations (bugfix, feature, refactor, discovery, decision, change)
- Session summaries (request, investigated, learned, completed, next_steps)
- Session metadata

SQLite is ideal for:
- Structured queries (filter by type, date, session)
- ACID transactions
- Single-file portability
- WAL mode for concurrent reads during writes

### Vector DB for Semantic Search

The claude-mem plugin uses Chroma (`vector-db/`) for semantic similarity search. This enables:
- Finding related observations across sessions by meaning
- Fuzzy matching when exact keywords are unknown
- Embedding-based retrieval

### File-Based Storage

Simpler plugins can use file-based storage:
- JSONL files for append-only logs
- Markdown files for human-readable state
- JSON files for configuration

### Storage Location Conventions

Store data alongside the plugin or in a well-known user directory:

```
~/.claude-mem/
├── claude-mem.db         # SQLite database
├── claude-mem.db-shm     # WAL shared memory
├── claude-mem.db-wal     # Write-ahead log
├── vector-db/            # Chroma vector database
├── settings.json         # Plugin configuration
├── worker.pid            # Worker process ID
└── logs/                 # Daily log files
```

---

## 11. Best Practices

### Hook Performance

- **Keep hooks fast.** SessionStart runs on every session; slow hooks degrade startup.
- **Offload to a background worker.** Do not run LLM inference or heavy computation synchronously in a hook. Post to a worker and return immediately.
- **Use `async: true`** for hooks that do not need to block (logging, metrics, background tests).
- **Set appropriate timeouts.** Default is 600s for commands. Set shorter timeouts (30-60s) for hooks that should not block long.

### Error Handling

- **Hooks should not crash Claude.** A failing hook should exit with a non-blocking code (not 2) and log the error. Only use exit 2 when you intend to block an action.
- **Validate stdin JSON.** Use `jq` or equivalent to safely parse. Handle missing fields gracefully.
- **Guard against empty input.** Some events may have optional fields.

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if [ -z "$COMMAND" ]; then
  exit 0  # Nothing to validate
fi
```

### Idempotency

- **Worker start must be idempotent.** Multiple hooks may try to start the worker simultaneously. Use PID files or port checks.
- **Observation extraction should handle duplicates.** The same tool call might be observed multiple times after compaction events.

### Session Isolation

- **SessionStart resets context.** Do not assume state persists across sessions.
- **Use `stop_hook_active` to prevent infinite loops.** If your Stop hook blocks stopping, check this flag to avoid running forever.

```bash
#!/bin/bash
INPUT=$(cat)
ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active')
if [ "$ACTIVE" = "true" ]; then
  exit 0  # Already continuing, do not block again
fi
```

### Plugin Portability

- **Use `${CLAUDE_PLUGIN_ROOT}`** for all paths in hook commands and MCP configs.
- **Never hardcode absolute paths** in hook configs that ship with a plugin.
- **All manifest paths must be relative** and start with `./`.
- **Declare Node/Bun engine requirements** in `package.json`.

### MCP Server Design

- **MCP servers expose their own tools** -- they do not inherit Claude's Read, Write, or Bash.
- **Keep tool count reasonable.** Too many tools consume context. Use Tool Search for large servers.
- **Use clear tool descriptions.** Claude relies on descriptions to select tools.
- **Return structured JSON** in tool results for Claude to parse.

### Security

- **Validate and sanitize all hook inputs.** Never trust stdin data blindly.
- **Quote shell variables.** Use `"$VAR"` not `$VAR`.
- **Block path traversal.** Check for `..` in file paths.
- **Avoid exposing secrets.** Do not log sensitive environment variables.

### Debugging

- Run `claude --debug` to see hook execution details (which hooks matched, exit codes, output).
- Toggle verbose mode with `Ctrl+O` to see hook progress in the transcript.
- Use `/hooks` to view, add, and delete hooks interactively.
- Use `/mcp` to check MCP server status and authenticate.
- Plugin errors appear in the `/plugin` Errors tab.

### Common Pitfalls

| Issue                         | Cause                               | Fix                                      |
|-------------------------------|--------------------------------------|------------------------------------------|
| Plugin not loading            | Invalid `plugin.json`                | Validate JSON, check `/plugin` errors    |
| Components missing            | Wrong directory structure            | Components at root, not in `.claude-plugin/` |
| Hooks not firing              | Script not executable                | `chmod +x script.sh`                     |
| MCP server fails              | Missing `${CLAUDE_PLUGIN_ROOT}`      | Use variable for all plugin paths        |
| Hook JSON parse failure       | Shell profile prints to stdout       | Ensure clean stdout, no banner text      |
| Infinite Stop loop            | Not checking `stop_hook_active`      | Check flag before blocking               |
| Stdin crash on Linux          | Bun stdin fstat EINVAL               | Handle stdin piping carefully            |
| Worker race conditions        | Multiple hooks starting worker       | PID file, port check, idempotent start   |

---

## Sources

- [Plugins reference (official)](https://code.claude.com/docs/en/plugins-reference)
- [Hooks reference (official)](https://code.claude.com/docs/en/hooks)
- [Skills documentation (official)](https://code.claude.com/docs/en/skills)
- [Subagents documentation (official)](https://code.claude.com/docs/en/sub-agents)
- [MCP documentation (official)](https://code.claude.com/docs/en/mcp)
- [Claude Code plugins repository](https://github.com/anthropics/claude-code/blob/main/plugins/README.md)
- [rmcp -- Official Rust MCP SDK](https://github.com/modelcontextprotocol/rust-sdk)
- [rmcp crate documentation](https://docs.rs/rmcp)
- [rust-mcp-sdk (alternative)](https://crates.io/crates/rust-mcp-sdk)
- [Model Context Protocol specification](https://modelcontextprotocol.io/introduction)
