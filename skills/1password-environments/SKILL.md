---
name: 1password-environments
description: >-
  Manages secrets in 1Password Developer Environments. Use when the user asks to
  import or mount a .env file, list or create an Environment, view or add
  environment variables, or store an API key, token, or credential.
---

# 1Password Environments

Workflow for the 1Password MCP tools bundled with this plugin. Read this skill
before calling MCP tools — the MCP server's built-in docs cover tool basics but
omit import-and-mount steps. Conceptual background and edge cases:
[reference.md](reference.md).

## Prerequisites

- **macOS or Linux** with the 1Password desktop app installed (MCP, local `.env` mounts, and mount validation are not supported on Windows)
- **MCP Server** Labs experiment enabled in the desktop app (`onepassword://settings/labs`)
- Plugin installed via `/plugin install` (registers `1password-mcp` MCP config, this skill, and the mount validation hook together)

Setup details: [reference.md](reference.md)

## Windows

**Detect the platform before any workflow** (the session's environment info, `C:\` paths,
`OS=Windows_NT`, etc.). On Windows, local `.env` mounts and the desktop MCP server are
**not supported**. Note that WSL reports as Linux and *is* supported.

**Hard stop:** do **not** call `create_local_env_file`, attempt mounting, or run mount-only
flows. Import is complete after `create_environment` (or resolve) + importing the
values (see **Import from a `.env` file**) + `list_variables` to verify.

Recommend **1Password CLI environment injection** ([Load secrets into the environment](https://www.1password.dev/cli/secrets-environment-variables)).

Example to share:

```shell
op run --environment=<environmentId> -- <their-start-command>
```

Use the `environmentId` from the import response. Apps that read a `.env` file from disk need
this wrapper.

This section **overrides** mount steps below on Windows.

## Getting started (new user)

Use when the request is open-ended — "how do I set up my environment?", "where do
I start?" — rather than naming an operation.

Orient in a sentence or two: secrets live in 1Password, the `.env` path becomes a
live FIFO that 1Password feeds on demand, and their tooling reads `.env` as
before. Do not paste the prerequisites list into chat.

Check state before asking anything: **`authenticate`** (fastest real prerequisite
check — on failure, fix setup per [reference.md](reference.md) → **When things
fail**), **`list_environments`**, then look for a `.env` with Glob — never Read
or Bash (a `.env` holds secret values; see **Never read secret values**).

Then route:

| State | Go to |
|-------|-------|
| `.env` with real values | **Import from a `.env` file** — the common case |
| Only `.env.example` / `.env.template` | **Create new Environment** using the template's keys (templates hold no secrets, so Read is fine) → ask the user for values → `append_variables` |
| No `.env`, no Environments | **Create new Environment** → `append_variables` → mount at `{workspace_root}/.env` |
| Environment exists, no mount | **Mount existing Environment** |

Ask one question at a time, and only when the answer is not discoverable.

## Not done until

**Import / create from `.env`** (including "using values from the project `.env`"):

- [ ] Environment created or resolved
- [ ] User chose an import method, the variables were imported, and `list_variables` shows the names
- [ ] `create_local_env_file` at the **source** `.env` absolute path — **always**
- [ ] Mount verified with `list_local_env_files`

Mounting at the source `.env` path is mandatory, not optional follow-up. The
**only** exception is the user explicitly opting out ("without mounting", "do not
mount", "skip the mount"). Never ask "want me to mount?" — just mount.

Stopping after the import, before mounting, is **incomplete**.
`list_variables` is not mount verification. Do not report success until the mount
checklist is done.

**Mount only:** mount exists at the requested path (`list_local_env_files`).

On **Windows**, skip mount steps — see **Windows** above.

## Do not

- Attempt mounting or call `create_local_env_file` on **Windows**
- Skip `create_local_env_file` on import unless the user explicitly opted out
- Report success before the import checklist (including mount) is complete
- Offer mounting as optional follow-up — it is mandatory on import
- Call `create_environment` when `list_environments` already shows that name — ask the user first (see **Duplicate environment name**)
- Reveal secret values in chat
- Read, Grep, `cat`, or otherwise parse a `.env` file that holds real values, unless the user chose to let Claude import it — see **Never read secret values**
- Choose the import method for the user — always ask (step 4 of **Import from a `.env` file**)
- Read a mounted `.env` path — once mounted, the path is a live FIFO (named pipe); use `list_variables` instead
- Verify a mount with **any** shell check (`test -f`, `[ -f ]`, `find -type f`, `test -p`, etc.). Use `list_local_env_files` — it is the only check that reflects both file state and whether the mount is *enabled* in 1Password. A disabled mount leaves the FIFO on disk, so `test -p` succeeds while the hook keeps blocking every Bash command. Shell checks are also unusable in exactly the broken case: when a mount is missing or invalid the hook denies all Bash, so the check cannot run. A failed `-f` check does not mean the mount is missing

## MCP tools

Tools are namespaced `mcp__plugin_1password_1password__<tool>` — for example
`mcp__plugin_1password_1password__authenticate`. Read each tool's schema before
invoking it.

Request params use **camelCase** (`accountId`, `environmentId`); responses may use
snake_case (`account_id`, `environment_id`). Pass every required param.

Non-obvious schema details:

- `create_local_env_file`: `mountPath` must be the **absolute** path of the source `.env` file (macOS/Linux)
- Environment-level tools may prompt the user for per-environment approval on first use

If MCP calls fail with authentication errors, call `authenticate`, then retry with
the returned account ID (use as `accountId` in subsequent calls).

## Resolve environment

After `authenticate` → `list_environments`:

1. Match `environmentName` exactly (case-sensitive) to get `environmentId`
2. If no match, ask the user for the correct name — do not guess
3. If multiple accounts/environments confuse the match, list names only and ask

## Duplicate environment name

Before **`create_environment`**, call **`list_environments`** and check whether the
target `environmentName` already exists.

If it does, **stop and ask the user** how they want to proceed. Offer options such as:

- **Use the existing environment** — skip `create_environment`; use its `environmentId` for later steps (`append_variables`, mount, etc.)
- **Use a different name** — wait for a new name from the user, then `create_environment`
- **Cancel** — do not create or modify anything

Do not silently choose one of these paths. Do not call `create_environment` with a
name that already exists unless the user has explicitly chosen a different name.

## Never read secret values

Anything you read becomes part of the conversation: it is sent to the model and
saved in the session transcript. Keep secret values out of it:

- Do not read a `.env` file that holds real values — not with Read, Grep, or any shell command — unless the user chose **Let Claude import them** in step 4 of **Import from a `.env` file**. Templates (`.env.example`, `.env.template`, `.env.sample`) are fine to read.
- By default, values get into 1Password through the desktop app's **Import .env file**, which reads the file without passing values through you.
- Use `append_variables` only for values the user gives you directly, and only when they explicitly ask to add or update variables. If the user pasted a secret into chat, refer to it by variable name and never repeat it back.

## Import from a `.env` file

Default path: `{workspace_root}/.env` unless the user names another path.

1. **Confirm the file exists** with Glob. Do not read it.
2. **`authenticate`** → `accountId`
3. **`list_environments`** — if the target name already exists, follow **Duplicate environment name** and wait for the user's choice. Otherwise **`create_environment`** (new name) or resolve the existing environment per the user's choice.
4. **Ask how to import the values.** Ask once, and wait for the answer:
   - **Import in the 1Password app (recommended)** — secret values go straight from the file into 1Password and never pass through Claude.
   - **Let Claude import them** — faster, but Claude reads the file, so every value in it is sent to the model and saved in the session transcript.

   **If they choose the 1Password app:** tell them to go to **Developer** > **View Environments**, select *{environmentName}*, select **Import .env file**, and choose `{absolute path to the .env}`. Ask them to tell you when it's done, and wait.

   **If they choose Claude:** Read the `.env` to get its keys and values. Strip optional surrounding quotes from values. Call `append_variables` with all variables (see **Concealed variables**). Never repeat values in chat.
5. **`list_variables`** to confirm the import. Report the variable names only. If nothing arrived, return to step 4.
6. **Git-tracked `.env`** — **all platforms**, including Windows:
   - If the `.env` is **git-tracked**, stop and tell the user to delete it and commit that removal ([local `.env` file docs](https://www.1password.dev/environments/local-env-file.md)). On macOS/Linux, do not proceed to mount until this is done.
7. **Mount (macOS/Linux only)** — **always** (skip only if the user explicitly said not to mount; on **Windows**, skip entirely — see **Windows**):
   - `list_local_env_files` — skip `create_local_env_file` only if a mount already exists at the source path
   - `create_local_env_file` with `accountId`, `environmentId`, `environmentName`, `mountPath` (absolute path of the original `.env`)
   - `list_local_env_files` again to verify

If shell commands are blocked because 1Password expects a mount at the path, see
[reference.md](reference.md) (mount conflict).

## Other flows

**Create new Environment:** authenticate → `list_environments` → if the name exists, follow **Duplicate environment name** → `create_environment` only when the name is available or the user chose a different name.

**Mount existing Environment:** authenticate → resolve environment → steps 6–7 above (step 7 macOS/Linux only).

**Inspect names:** authenticate → resolve environment → `list_variables` → summarize names only.

**Rename:** authenticate → resolve environment → confirm name → `rename_environment`.

**Add/update variables:** only when the user explicitly asks → collect missing names and values from the user → authenticate → resolve environment → `list_variables` → `append_variables`. To keep a secret out of the chat entirely, the user can add it in the 1Password desktop app instead; offer that when they haven't pasted the value yet.

## Concealed variables

When calling `append_variables`, set `concealed` per variable:

- Set `concealed: true` for API keys, tokens, passwords, private keys, and connection strings with credentials.
- Set `concealed: false` for ports, public URLs, feature flags, and non-sensitive config unless the user says otherwise.
- When unsure, default to `concealed: true`.

## Safety

- Never reveal secret values in chat
- Never read a `.env` file that holds real values unless the user chose to let Claude import it (see **Never read secret values**)
- Ask before changing variables unless the request is explicit

## Plugin hook

The plugin runs `validate-mounted-env-files.sh` as a `PreToolUse` hook on the `Bash`
tool. It blocks Bash commands when 1Password expects a mount that is missing,
disabled, or not a FIFO (for example a plain `.env` still on disk at the mount
path). Only Bash is gated — Read, Edit, Grep, and the MCP tools are unaffected, so
the import flow never needs a shell.

Validation modes and recovery steps: [reference.md](reference.md)

## Troubleshooting

Mount conflict, validation modes, setup: [reference.md](reference.md)
