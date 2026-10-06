# 1Password Plugin for Claude Code

A [1Password](https://1password.com) plugin for [Claude Code](https://code.claude.com), built and maintained by 1Password. It ships three pieces that work together: a **PreToolUse hook** that validates locally mounted `.env` files before Bash commands run, an **agent skill** with the complete Developer Environment workflow, and **MCP configuration** for the 1Password desktop app server. Secret values stay in 1Password — the agent sees variable names and mount paths, not secret contents.

Install the **plugin** rather than hand-configuring an MCP entry on its own. The bundled `1password-environments` skill is the authoritative agent workflow; the MCP server's built-in documentation resources cover tool basics only and omit the import-and-mount steps.

For more on 1Password's developer tools, see the [1Password Developer Documentation](https://developer.1password.com).

## Requirements

- [1Password](https://1password.com) subscription
- [1Password desktop app](https://1password.com/downloads) on **macOS or Linux**
- [Claude Code](https://code.claude.com)

Additional requirements by feature:

- **Hook** — [sqlite3](https://www.sqlite.org/) installed and available in your `PATH` (pre-installed on macOS; install via your package manager on Linux)
- **MCP** — the 1Password Labs **MCP Server** experiment enabled in the desktop app (`onepassword://settings/labs`). If the setting is missing, your account may not have the `ai-local-mcp-server` feature flag. The plugin's `.mcp.json` launches the `1password-mcp` command from your `PATH`, as described in the [1Password MCP server documentation](https://www.1password.dev/environments/mcp-server).

> **Platform support:** MCP, local `.env` mounts, and mount validation are supported on **macOS and Linux**. On **Windows**, the hook exits immediately with no decision so Bash is not blocked; 1Password Environments has no local `.env` mounts on Windows.

## Installation and Setup

### Step 1: Set up your Environments

Before using this plugin, configure your secrets in 1Password:

1. [Create one or more Environments](https://developer.1password.com/docs/environments) in 1Password to store your project secrets.
2. [Configure locally mounted `.env` files](https://developer.1password.com/docs/environments/local-env-file) for them.

### Step 2: Install the plugin

Installing the plugin registers the validation hook, the `1password-environments` agent skill, and the MCP server configuration together.

**From this repository.** This repo is itself a plugin marketplace, so add it directly by its GitHub `owner/repo` name — no separate catalog needed:

```
claude /plugin marketplace add 1Password/1password-claude-plugin
claude /plugin install 1password@1password
```

Run `/plugin` afterwards to confirm the plugin is installed and the MCP server is connected.

**For local development**, point Claude Code at a checkout instead:

```bash
claude --plugin-dir /path/to/1password-claude-plugin
```

### Step 3: Enable MCP in 1Password (required for Environment management)

Enable the **MCP Server** experiment in the 1Password desktop app: open **Settings → Labs** (or use `onepassword://settings/labs`) and turn on **MCP Server**. The plugin's `.mcp.json` connects Claude Code to that server after this step.

The plugin registers the MCP command documented by 1Password:

```json
{
  "mcpServers": {
    "1password": {
      "command": "1password-mcp"
    }
  }
}
```

Install the 1Password desktop app on macOS or Linux so `1password-mcp` is available on your `PATH`. For platform-specific install paths and troubleshooting, see the [1Password MCP server documentation](https://www.1password.dev/environments/mcp-server).

## Features

### Hooks

See the [1Password Agent Hooks documentation](https://www.1password.dev/agent-hooks) for background on how 1Password's agent hooks work.

#### Local `.env` File Validation (`PreToolUse`)

Validates locally mounted `.env` files from [1Password Environments](https://developer.1password.com/docs/environments) before any Bash command runs. When required environment files are missing, disabled, or misconfigured, the hook blocks the command and surfaces actionable error messages so Claude can guide you to a fix. Reading a `.env` with the Read tool is unaffected — only Bash is gated.

This hook was originally developed in the [1Password Agent Hooks](https://github.com/1Password/agent-hooks) repository.

**How it works:**

Every time Claude Code attempts to run a Bash command, the hook:

1. **Discovers** your configured [local `.env` files](https://developer.1password.com/docs/environments/local-env-file) by querying the 1Password database.
2. **Validates** that each file exists as a valid FIFO (named pipe) and is enabled in 1Password.
3. **Passes** with no decision if all environment files are properly configured — your normal Bash permission settings apply unchanged.
4. **Blocks** the command and provides clear error messages when files are missing or disabled.

The hook uses a **"fail open"** approach: if 1Password is not installed, the database is unavailable, or `sqlite3` is missing, the hook reports no permission decision and execution proceeds. It never auto-approves a command. Returning `permissionDecision: "allow"` would skip the permission prompt, so a clean passing check emits nothing (exit 0, empty stdout) and your normal permission settings apply.

When validation is skipped rather than passed, the hook still reports no decision, but attaches `additionalContext` explaining why. Claude can then tell you the check did not run, instead of the command appearing to pass validation silently.

##### Validation Modes

The hook supports two validation modes depending on whether a TOML configuration file is present.

**Default Mode**

When no `.1password/environments.toml` file exists in your project (or when the file exists but doesn't contain a `mount_paths` field), the hook automatically:

1. Detects your operating system (macOS or Linux).
2. Queries the 1Password database for all configured mount entries.
3. Filters to only the local `.env` files relevant to the current workspace.
4. Validates that each discovered file is enabled and exists as a valid FIFO.

**Configured Mode**

When a `.1password/environments.toml` file exists at your project root **and** contains a `mount_paths` field, only the specified files are validated:

```toml
# Validate only these specific files
mount_paths = [".env", "billing.env", "database.env"]
```

This gives you precise control over which files the hook checks. Configuration examples:

| Configuration                           | Behavior                                                                |
| --------------------------------------- | ----------------------------------------------------------------------- |
| `mount_paths = [".env"]`                | Only `.env` is validated                                                |
| `mount_paths = [".env", "billing.env"]` | Both files are validated                                                |
| `mount_paths = []`                      | Validation is disabled — all commands allowed                           |
| *(no TOML file)*                        | Default mode — all 1Password-mounted files in the project are validated |

Mount paths can be relative to the project root or absolute. Multi-line arrays are supported:

```toml
mount_paths = [
    ".env",
    "billing.env",
    "database.env",
]
```

For each file, the hook checks:

- **Exists** — the file is present on disk.
- **Is FIFO** — the file is a named pipe (how 1Password mounts secrets).
- **Is enabled** — the mount is turned on in the 1Password app.

##### Debugging

**Claude Code debug output**

Run `claude --debug` to see hook matches and exit codes in the session output.

**Manual testing with debug mode**

Run the hook directly with `DEBUG=1` to see detailed output on stderr:

```bash
echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","cwd":"/path/to/your/project","tool_input":{"command":"echo test"}}' \
  | DEBUG=1 ./scripts/validate-mounted-env-files.sh
```

**Log file**

When not running in debug mode, the hook writes logs to `/tmp/1password-claude-code-hooks.log`. Log entries include timestamps and details about 1Password queries, validation results, and permission decisions.

### MCP and agent skill

The plugin connects Claude Code to the local 1Password MCP server and bundles the **`1password-environments`** skill (`skills/1password-environments/SKILL.md`). Claude reads that skill before calling MCP tools — it defines the complete workflow for importing a plain `.env` file (by default through the 1Password desktop app, so Claude never reads its secret values), adding variables, and mounting at the source path. The MCP server's built-in docs cover tool basics but omit those import-and-mount steps.

See `skills/1password-environments/reference.md` for setup, mount conflicts, and validation details.

#### Example prompts

- "List my 1Password Environments"
- "Mount my staging Environment as `.env` in this repo"
- "What variables are in my production Environment?"
- "Create a new Environment called `my-app-dev`"
- "Create an Environment from my project `.env` file"
- "Import `.env` into 1Password and mount it here"
- "Add a placeholder for my OpenAI API key"

#### MCP tools

Tools are namespaced `mcp__plugin_1password_1password__<tool>`.

| Tool | Description |
|------|-------------|
| `authenticate` | Authenticate with the 1Password desktop app; returns `accountId` |
| `list_environments` | List Developer Environments for an account |
| `create_environment` | Create a new Developer Environment |
| `rename_environment` | Rename an existing Developer Environment |
| `list_variables` | List variable names in an Environment (no values) |
| `append_variables` | Add or update Environment variables |
| `create_local_env_file` | Mount an Environment as a local `.env` file |
| `list_local_env_files` | List existing local `.env` mounts for an Environment |

Confirm the MCP server is connected with `/plugin` after installing the plugin and enabling the Labs experiment in 1Password.

## Plugin Structure

```
1password-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json                    # Plugin manifest
│   └── marketplace.json               # Marketplace catalog (for distribution)
├── hooks/
│   └── hooks.json                     # PreToolUse mount validation
├── skills/
│   └── 1password-environments/
│       ├── SKILL.md                   # Agent skill for MCP workflows
│       └── reference.md               # Setup, mount conflicts, troubleshooting
├── scripts/
│   ├── lib/
│   │   └── telemetry.sh               # Opt-in telemetry helpers for the validation hook
│   └── validate-mounted-env-files.sh  # Bash hook (macOS / Linux)
├── .mcp.json                          # MCP server configuration
├── LICENSE
└── README.md
```

## Telemetry

The validation hook emits **opt-in** telemetry so 1Password can understand plugin adoption and the prevalence of common failure modes (missing files, disabled mounts). Two event types are emitted:

- `agent_hook_execution` — fired once per hook invocation; carries the hook name, plugin version, client (`claude-code`), bucketed duration, decision (`allow`/`deny`), reason for deny, validation mode (`default`/`configured`), and a count of mounts checked.
- `agent_hook_install` — fired once per `(hook_name, plugin_version)` on the first hook run after installation or upgrade; `install_method` is `plugin_marketplace`.

**Opt-in only.** Events are written only when the file `~/.config/1Password/telemetry-enabled` exists. The 1Password desktop app creates and removes this file based on your in-app telemetry preference (Settings → Manage Account → Data Usage). If the app has never run, or all accounts have opted out, no events are written.

**No PII.** Events contain hook name and version, client, decision, bucketed duration, mode, mount count, and a deny reason. No paths, file contents, environment names, or workspace paths are recorded.

**Fail-open.** Telemetry runs in a detached background subshell after the hook has returned its decision to Claude Code. Any failure (missing helpers, disk full, permission denied) is silently swallowed — telemetry can never affect a hook decision.

**Where events are written.** Events are appended as JSON lines to `~/.config/1Password/data/hook-events/events.jsonl`. The 1Password desktop app periodically ingests this file and forwards events to 1Password's telemetry pipeline. Telemetry only fires on macOS and Linux; the Windows early exit does not emit events.

**To disable.** Open the 1Password desktop app → Settings → Manage Account → Data Usage and turn off product telemetry.

## Resources

- [1Password Agent Hooks documentation](https://www.1password.dev/agent-hooks) — how 1Password's agent hooks work
- [1Password Agent Hooks](https://github.com/1Password/agent-hooks) — the original hooks repository this plugin is based on
- [1Password Environments](https://developer.1password.com/docs/environments) — documentation for 1Password's environment and secrets management
- [1Password Local `.env` Files](https://developer.1password.com/docs/environments/local-env-file) — how local `.env` file mounting works
- [1Password MCP server documentation](https://www.1password.dev/environments/mcp-server) — MCP setup and troubleshooting
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks) — how Claude Code hooks work
- [Claude Code Plugins](https://code.claude.com/docs/en/plugins) — how to create and distribute Claude Code plugins

## License

[MIT](./LICENSE) — Copyright (c) 2026 1Password
