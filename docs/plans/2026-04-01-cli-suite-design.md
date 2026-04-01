# CLI Suite Design

## Architecture

Three crates in a Cargo workspace, plus a thin dispatcher:

- **`llm-cli`** — git-style dispatcher that execs `llm-cli-<subcommand>`
- **`llm-cli-linear`** — Linear API client
- **`llm-cli-discourse`** — Discourse API client

All crates live under `packages/`.

## Output

JSON to stdout by default. `--human` flag on each tool for readable output. Diagnostics to stderr.

Success responses:

```json
{
  "success": true,
  "data": { ... }
}
```

Error responses:

```json
{
  "success": false,
  "error": {
    "code": "OP_NOT_FOUND",
    "message": "Could not retrieve API key from 1Password. Is the 1Password app running?",
    "suggestion": "Ensure the 1Password desktop app is running and unlocked"
  }
}
```

With `--human`, errors print as plain text to stderr, data as formatted text to stdout.

## Configuration

Path: `$XDG_CONFIG_HOME/llm-cli/config.toml` (defaults to `~/.config/llm-cli/config.toml`).

```toml
[linear]
api_url = "https://api.linear.app"
op_item_id = "abc123-some-uuid"

[discourse.my-forum]
base_url = "https://forum.example.com"
op_item_id = "def456-some-uuid"
api_username = "james"
```

## Authentication

Both tools retrieve API keys from 1Password at call time via `op item get <op_item_id> --field credential`. The item ID comes from config. No caching, no wrapper binary — each invocation calls `op` directly.

## `llm-cli` Dispatcher

Minimal binary with no dependencies beyond std.

- `llm-cli linear <args...>` — finds `llm-cli-linear` on `$PATH`, execs it with `<args...>`
- `llm-cli --help` or no args — lists available subcommands by scanning `$PATH` for `llm-cli-*` binaries
- Unknown subcommand — stderr error with available subcommands, exit 1

Uses `std::os::unix::process::CommandExt::exec` to replace the process (no child process management).

No flags of its own beyond `--help`. No config. No auth.

## `llm-cli-linear`

### Subcommands

- `issues list` — issues assigned to the authenticated user. Default 25 results, `--limit` to adjust. Truncated results include a message steering toward narrower filters.
- `issues get --id <issue-id>` — fetch a single issue
- `issues create --title <title> --team <team> [--description <desc>] [--priority <1-4>]` — create an issue
- `issues close --id <issue-id>` — close an issue (sets state to "Done")

### Common flags

- `--human` — human-readable output

### API

GraphQL. Each subcommand constructs the appropriate query/mutation with only the fields needed.

### Config

```toml
[linear]
api_url = "https://api.linear.app"  # optional, has default
op_item_id = "abc123-some-uuid"
```

## `llm-cli-discourse`

### Subcommands

- `posts get --id <post-id>` — fetch a single post
- `posts create --title <title> --category <category> [--raw <body>]` — create a new topic/post
- `posts delete --id <post-id>` — delete a post
- `comments create --post-id <post-id> --raw <body>` — reply to a post
- `comments delete --id <comment-id>` — delete a comment/reply

### Common flags

- `--human` — human-readable output
- `--instance <name>` — which Discourse instance to use (maps to `[discourse.<name>]` in config). Required when multiple instances configured; if only one exists, used automatically.

### API

REST/JSON. API key and username sent via `Api-Key` and `Api-Username` headers.

### Config

```toml
[discourse.my-forum]
base_url = "https://forum.example.com"
op_item_id = "def456-some-uuid"
api_username = "james"
```

## Error Handling

Common error scenarios across both tools:

- `op` not found or not running — clear error with setup instructions
- API key retrieval failed — include the item ID in the error message
- Config file missing or malformed — error with expected config path and example
- API request failed — pass through HTTP status and API error message

## Dependencies

**`llm-cli`:** no dependencies (std only)

**`llm-cli-linear` and `llm-cli-discourse`:**

- `clap` — argument parsing (derive)
- `serde` / `serde_json` — JSON serialization
- `toml` — config parsing
- `ureq` — HTTP client (blocking, no async runtime)
- `dirs` — XDG directory resolution

No async runtime. Short-lived CLI invocations making one or two HTTP calls.
