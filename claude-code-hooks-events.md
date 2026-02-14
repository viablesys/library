# Claude Code Hook Event Schemas

Event-level reference for Claude Code hooks. What data each event provides via stdin JSON, what varies by tool type, and what can be extracted for structured observation capture without LLM. Supports nmem ADR-002 (Position A).

Companion to `claude-code-plugins.md` (hook architecture and configuration).

---

## 1. Event Overview

| Event                | Fires When                    | Extra Fields Beyond Common                        | nmem Use               |
|----------------------|-------------------------------|---------------------------------------------------|------------------------|
| `PostToolUse`        | After tool call succeeds      | `tool_name`, `tool_input`, `tool_response`, `tool_use_id` | Observation extraction |
| `PostToolUseFailure` | After tool call fails         | `tool_name`, `tool_input`, `tool_response`, `tool_use_id` | Error capture          |
| `PreToolUse`         | Before tool call executes     | `tool_name`, `tool_input`, `tool_use_id`          | Gating, validation     |
| `Stop`               | Agent finishes responding     | `stop_hook_active`                                | Session summary        |
| `SubagentStop`       | Subagent finishes             | `stop_hook_active`                                | Subagent summary       |
| `SessionStart`       | Session begins/resumes        | `source`, `model`                                 | Context injection      |
| `SessionEnd`         | Session terminates            | _(matcher: exit reason)_                          | Cleanup                |
| `UserPromptSubmit`   | User submits prompt           | `prompt`                                          | Intent capture         |
| `TaskCompleted`      | Task marked complete          | `task_id`, `task_subject`, `task_description`     | Task tracking          |
| `PreCompact`         | Before context compaction     | _(matcher: `manual`/`auto`)_                      | Pre-compaction snapshot|
| `Notification`       | Claude sends notification     | _(matcher: notification type)_                    | Alerting               |
| `PermissionRequest`  | Permission dialog shown       | `tool_name`, `tool_input`                         | Permission auditing    |
| `SubagentStart`      | Subagent spawned              | _(matcher: agent type)_                           | Subagent tracking      |
| `TeammateIdle`       | Team agent about to idle      | _(common fields only)_                            | Team coordination      |

---

## 2. Common Input Fields

Every hook receives these via stdin JSON:

```json
{
  "session_id": "d4f8a2b1-...",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PostToolUse"
}
```

| Field              | Type   | Notes                                                |
|--------------------|--------|------------------------------------------------------|
| `session_id`       | string | UUID, stable for session lifetime                    |
| `transcript_path`  | string | JSONL file, each line: `{type, message: {content}}`  |
| `cwd`              | string | Working directory of the session                     |
| `permission_mode`  | string | `default`, `plan`, `bypassPermissions`, etc.         |
| `hook_event_name`  | string | Which event fired                                    |

---

## 3. PostToolUse

The richest event. Adds `tool_name`, `tool_input`, `tool_response`, `tool_use_id` to common fields.

### tool_input by Tool Type

**Bash**
```json
{ "command": "cargo test --lib", "description": "Run library unit tests", "timeout": 120000 }
```
`command` (required), `description` (optional), `timeout` (optional ms).

**Read**
```json
{ "file_path": "/home/user/project/src/main.rs", "offset": 0, "limit": 200 }
```
`file_path` (required), `offset`/`limit` (optional).

**Write**
```json
{ "file_path": "/home/user/project/src/config.rs", "content": "use serde::Deserialize;\n..." }
```
`file_path` (required), `content` (required, full file contents).

**Edit**
```json
{ "file_path": "/home/user/project/src/lib.rs", "old_string": "fn old_name(", "new_string": "fn new_name(", "replace_all": false }
```
`file_path`, `old_string`, `new_string` (required), `replace_all` (optional).

**Grep**
```json
{ "pattern": "fn handle_request", "path": "/home/user/project/src", "glob": "*.rs", "output_mode": "files_with_matches" }
```
`pattern` (required), `path`/`glob`/`type`/`output_mode` (optional).

**Glob**
```json
{ "pattern": "**/*.toml", "path": "/home/user/project" }
```
`pattern` (required), `path` (optional).

**Task** (subagent spawn): `{ "prompt": "...", "subagent_type": "Explore" }`

**MCP tools** (`mcp__<server>__<tool>`): Fields vary per tool definition.

### tool_response

A **string** for all built-in tools (not a JSON object):

| Tool  | Response Content                                          |
|-------|----------------------------------------------------------|
| Bash  | stdout/stderr combined. May be truncated for large output |
| Read  | File content with `cat -n` line numbers. Lines >2000 chars truncated |
| Write | Confirmation message (e.g., `"Successfully wrote to ..."`) |
| Edit  | Confirmation message with replacement result              |
| Grep  | File paths or content lines depending on `output_mode`    |
| Glob  | Newline-separated matching file paths                     |

### PostToolUseFailure

Same structure. `tool_response` contains the error message.

---

## 4. Stop / SubagentStop

### Extra Fields

```json
{ "stop_hook_active": false }
```

`stop_hook_active` (boolean): `true` if already continuing from a prior Stop hook block. **Always check this to prevent infinite loops.**

The final response is NOT in the payload. Extract from transcript:

```bash
INPUT=$(cat)
TRANSCRIPT=$(echo "$INPUT" | jq -r '.transcript_path')
# Last assistant message from JSONL
tac "$TRANSCRIPT" | while IFS= read -r line; do
  TYPE=$(echo "$line" | jq -r '.type')
  if [ "$TYPE" = "assistant" ]; then
    echo "$line" | jq -r '.message.content | if type == "array" then
      [.[] | select(.type == "text") | .text] | join("\n") else . end'
    break
  fi
done
```

Decision output: `{ "decision": "block", "reason": "Tests not run" }`

---

## 5. SessionStart

### Extra Fields

```json
{ "source": "startup", "model": "claude-opus-4-6" }
```

| `source` Value | Meaning                         |
|----------------|---------------------------------|
| `startup`      | Fresh session                   |
| `resume`       | Resuming previous session       |
| `clear`        | Context reset                   |
| `compact`      | Context compaction occurred     |

`source` is what hook matchers match against (e.g., `"startup|clear|compact"`).

Context injection output:
```json
{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "## Recent\n- Fixed auth bug" } }
```

Plain text on stdout also works. `$CLAUDE_ENV_FILE` (SessionStart only) accepts persistent env vars.

---

## 6. UserPromptSubmit

### Extra Fields

```json
{ "prompt": "Fix the failing test in src/auth/mod.rs" }
```

`prompt` (string): The full user input. Sensitivity considerations:
- Prompts starting with `/` are slash commands -- strip prefix for storage
- May contain sensitive data -- consider `<private>` tag filtering
- Primary signal for session intent capture
- Exit 2 blocks and erases the prompt

---

## 7. Other Events

**PreToolUse**: Same tool fields as PostToolUse minus `tool_response`. Can return:
```json
{ "hookSpecificOutput": { "hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "Blocked" } }
```

**TaskCompleted**: `task_id`, `task_subject`, `task_description` fields.

**PreCompact**: Matcher values `manual`/`auto`. No context snapshot in payload.

**Notification**: Matcher filters by type. Minimal payload beyond common fields.

**SessionEnd**: Matcher is exit reason. No summary data, just a termination signal.

---

## 8. Extraction Patterns

What can be derived from each event without LLM processing.

### File Operations (PostToolUse)
```
tool_name in [Read, Write, Edit]:
  files_read     <- tool_input.file_path  (Read)
  files_modified <- tool_input.file_path  (Write, Edit)
  change_type    <- "create" (Write) | "modify" (Edit)
  old_string     <- tool_input.old_string  (Edit)
  new_string     <- tool_input.new_string  (Edit)
```

### Commands (PostToolUse, Bash)
```
  command     <- tool_input.command
  description <- tool_input.description
  is_test     <- command matches /test|pytest|cargo test|npm test/
  is_build    <- command matches /build|compile|cargo build/
  exit_status <- parse tool_response for exit code patterns
```

### Search (PostToolUse, Grep/Glob)
```
  search_pattern <- tool_input.pattern
  search_scope   <- tool_input.path
  result_count   <- count newlines in tool_response
```

### MCP Tools (PostToolUse)
```
  mcp_server <- split tool_name on "__", index 1
  mcp_tool   <- split tool_name on "__", index 2+
  mcp_input  <- tool_input (varies per tool)
```

### Intent (UserPromptSubmit)
```
  user_intent  <- prompt (first 200 chars for indexing)
  is_command   <- prompt starts with "/"
  command_name <- first token after "/"
```

### Session Boundaries
```
SessionStart:
  is_fresh      <- source == "startup"
  is_compaction <- source == "compact"

Stop:
  is_continuation <- stop_hook_active
  transcript      <- transcript_path (for post-hoc summary)
```

---

## 9. Limitations

### Content Gaps
- **Read responses truncated**: Large files cut off; lines >2000 chars truncated
- **Edit has no diff**: `tool_response` is a success message, not a patch. Only `old_string`/`new_string` from `tool_input`
- **Write content can be huge**: `tool_input.content` is the full file. Truncate or hash for storage
- **Bash output truncated**: Large stdout/stderr cut by Claude Code before reaching the hook

### Structural Gaps
- **No timing**: No tool call duration in payload
- **No token counts**: Per-tool-call token usage not exposed
- **No sequence number**: No incrementing tool call counter. Track it yourself
- **No parent context**: PostToolUse lacks the assistant message that triggered it. Use transcript to correlate
- **No prompt number**: Which user turn triggered this. Derive via UserPromptSubmit counting

### Workarounds

| Gap                  | Workaround                                             |
|----------------------|--------------------------------------------------------|
| No prompt number     | Count UserPromptSubmit events per session               |
| No tool sequence     | Maintain counter in hook handler                        |
| No assistant context | Read `transcript_path` for last assistant message       |
| Large Write content  | Hash or truncate `tool_input.content` before storage    |
| No diff for Edit     | Store `old_string`/`new_string` pairs directly          |

---

## Sources

- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Plugin dev -- hook development](https://github.com/anthropics/claude-code/blob/main/plugins/plugin-dev/skills/hook-development/SKILL.md)
- [Plugin dev -- advanced hooks](https://github.com/anthropics/claude-code/blob/main/plugins/plugin-dev/skills/hook-development/references/advanced.md)
- claude-mem plugin (`~/.claude-mem/plugin/`) -- real-world hook payload processing
