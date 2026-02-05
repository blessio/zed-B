# Tool Permissions

Configure which agent actions run automatically and which require your approval.

## Quick Start

You can use Zed's Settings UI to configure tool permissions, or add rules directly to your `settings.json`:

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "default": "allow",
      "tools": {
        "terminal": {
          "default": "confirm",
          "always_allow": [
            { "pattern": "^cargo\\s+(build|test|check)" },
            { "pattern": "^npm\\s+(install|test|run)" }
          ],
          "always_confirm": [
            { "pattern": "sudo\\s+/" }
          ]
        }
      }
    }
  }
}
```

This example auto-approves `cargo` and `npm` commands in the terminal tool, while requiring manual confirmation on a case-by-case basis for `sudo` commands. Non-terminal commands are allowed by default, because of `"default": "allow"` under `"tool_permissions"`.

## How It Works

The `tool_permissions` setting lets you customize tool permissions by specifying regex patterns that:

- **Auto-approve** actions you trust
- **Auto-deny** dangerous actions (blocked even when `always_allow_tool_actions` is enabled)
- **Always confirm** sensitive actions regardless of other settings

## Supported Tools

| Tool | Input Matched Against |
| --- | --- |
| `terminal` | The shell command string |
| `edit_file` | The file path |
| `delete_path` | The path being deleted |
| `move_path` | Source and destination paths |
| `copy_path` | Source and destination paths |
| `create_directory` | The directory path |
| `save_file` | The file paths |
| `fetch` | The URL |
| `web_search` | The search query |

For MCP tools, use the format `mcp:<server>:<tool_name>`. For example, a tool called `create_issue` on a server called `github` would be `mcp:github:create_issue`.

## Configuration

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "default": "confirm",
      "tools": {
        "<tool_name>": {
          "default": "confirm",
          "always_allow": [{ "pattern": "...", "case_sensitive": false }],
          "always_deny": [{ "pattern": "...", "case_sensitive": false }],
          "always_confirm": [{ "pattern": "...", "case_sensitive": false }]
        }
      }
    }
  }
}
```

### Options

| Option | Description |
| --- | --- |
| `default` | Fallback when no patterns match: `"confirm"` (default), `"allow"`, or `"deny"` |
| `always_allow` | Patterns that auto-approve (unless deny or confirm also matches) |
| `always_deny` | Patterns that block immediately—highest priority, cannot be overridden |
| `always_confirm` | Patterns that always prompt, even with `always_allow_tool_actions` enabled |

### Pattern Syntax

```json [settings]
{
  "pattern": "your-regex-here",
  "case_sensitive": false
}
```

Patterns use Rust regex syntax. Matching is case-insensitive by default.

## Rule Precedence

From highest to lowest priority:

1. **Built-in security rules** — Hardcoded protections (e.g., `rm -rf /`). Cannot be overridden.
2. **`always_deny`** — Blocks matching actions
3. **`always_confirm`** — Requires confirmation for matching actions
4. **`always_allow`** — Auto-approves matching actions
5. **`default`** — Fallback when no regex patterns match

## Examples

### Terminal: Auto-Approve Build Commands

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "tools": {
        "terminal": {
          "default": "confirm",
          "always_allow": [
            { "pattern": "^cargo\\s+(build|test|check|clippy|fmt)" },
            { "pattern": "^npm\\s+(install|test|run|build)" },
            { "pattern": "^git\\s+(status|log|diff|branch)" },
            { "pattern": "^ls\\b" },
            { "pattern": "^cat\\s" }
          ],
          "always_deny": [
            { "pattern": "rm\\s+-rf\\s+(/|~)" },
            { "pattern": "sudo\\s+rm" }
          ],
          "always_confirm": [
            { "pattern": "sudo\\s" },
            { "pattern": "git\\s+push" }
          ]
        }
      }
    }
  }
}
```

### File Editing: Protect Sensitive Files

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "tools": {
        "edit_file": {
          "default": "confirm",
          "always_allow": [
            { "pattern": "\\.(md|txt|json)$" },
            { "pattern": "^src/" }
          ],
          "always_deny": [
            { "pattern": "\\.env" },
            { "pattern": "secrets?/" },
            { "pattern": "\\.(pem|key)$" }
          ]
        }
      }
    }
  }
}
```

### Path Deletion: Block Critical Directories

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "tools": {
        "delete_path": {
          "default": "confirm",
          "always_deny": [
            { "pattern": "^/etc" },
            { "pattern": "^/usr" },
            { "pattern": "\\.git/?$" },
            { "pattern": "node_modules/?$" }
          ]
        }
      }
    }
  }
}
```

### URL Fetching: Control External Access

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "tools": {
        "fetch": {
          "default": "confirm",
          "always_allow": [
            { "pattern": "docs\\.rs" },
            { "pattern": "github\\.com" }
          ],
          "always_deny": [
            { "pattern": "internal\\.company\\.com" }
          ]
        }
      }
    }
  }
}
```

### MCP Tools

```json [settings]
{
  "agent": {
    "tool_permissions": {
      "tools": {
        "mcp:github:create_issue": {
          "default": "confirm"
        },
        "mcp:github:create_pull_request": {
          "default": "confirm"
        }
      }
    }
  }
}
```

## Global Auto-Approve

To auto-approve all tool actions:

```json [settings]
{
  "agent": {
    "always_allow_tool_actions": true
  }
}
```

This bypasses confirmation prompts but `always_deny` patterns and built-in security rules still apply.

## Shell Compatibility

For the `terminal` tool, Zed parses chained commands (e.g., `echo hello && rm file`) to check each sub-command against your patterns.

All supported shells work with tool permission patterns, including sh, bash, zsh, dash, fish, PowerShell 7+, pwsh, cmd, xonsh, csh, tcsh, Nushell, Elvish, and rc (Plan 9).

## Writing Patterns

- Use `\b` for word boundaries: `\brm\b` matches "rm" but not "storm"
- Use `^` and `$` to anchor patterns to start/end of input
- Escape special characters: `\.` for literal dot, `\\` for backslash
- Test carefully—a typo in a deny pattern blocks legitimate actions

## Default Rules

Zed includes default `always_deny` rules for dangerous operations. View them with {#action zed::OpenDefaultSettings}:

- **Terminal:** `rm -rf /`, disk destruction commands, fork bombs, system file access
- **Edit file:** `.env` files, secrets directories, private keys
- **Delete path:** System directories, home directory roots, `.git` directories

These apply unless you override them. We recommend keeping these protections and adding your own rules on top.

## UI Options

When the agent requests permission, the dialog includes:

- **Allow once** / **Deny once** — One-time decision
- **Always allow** / **Always deny** — Adds a pattern to your settings

Selecting "Always allow" or "Always deny" updates your `settings.json` with a pattern for that input.
