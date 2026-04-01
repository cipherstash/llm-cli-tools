# llm-cli-tools

A suite of CLI tools designed for LLM agents to interact with SaaS APIs. JSON output by default, `--human` flag for readable output.

## Tools

| Binary | Service | API |
|--------|---------|-----|
| `llm-cli` | Dispatcher | Execs `llm-cli-<subcommand>` from `$PATH` |
| `llm-cli-linear` | [Linear](https://linear.app) | GraphQL |
| `llm-cli-discourse` | [Discourse](https://www.discourse.org) | REST |
| `llm-cli-slack` | [Slack](https://slack.com) | REST |

## Install

```sh
./install.sh
```

This discovers all binary crates in the workspace and runs `cargo install --path` for each.

## Configuration

### Quick setup

Run the interactive setup wizard to generate your config file:

```sh
llm-cli init
```

This detects which `llm-cli-*` tools are installed, provides instructions for creating API keys, and prompts for the required configuration fields.

### Manual setup

All tools read from `~/.config/llm-cli/config.toml` (or `$XDG_CONFIG_HOME/llm-cli/config.toml`).

```toml
[linear]
op_item_id = "your-1password-item-id"

[discourse.my-forum]
base_url = "https://forum.example.com"
op_item_id = "your-1password-item-id"
api_username = "your-username"

[slack]
op_item_id = "your-1password-item-id"
```

API keys are retrieved from 1Password at call time via the `op` CLI. Each config section requires an `op_item_id` pointing to a 1Password item. The key is read from the `credential` field by default; set `op_field` to use a different field.

## Usage

```sh
# Dispatcher
llm-cli linear issues list
llm-cli discourse posts latest

# Direct invocation
llm-cli-linear issues list --limit 10
llm-cli-linear issues get --id PROJ-123
llm-cli-linear issues create --title "Bug" --team ENG
llm-cli-linear issues close --id PROJ-123

llm-cli-discourse posts latest
llm-cli-discourse posts get --id 42
llm-cli-discourse posts create --title "Topic" --category general --raw "Body"
llm-cli-discourse comments create --post-id 42 --raw "Reply"

llm-cli-slack messages send --channel general --text "hello"
llm-cli-slack messages read --channel general
llm-cli-slack messages dm --user U12345 --text "hey"
llm-cli-slack messages mentions
llm-cli-slack summary --channel general
```

## Shell completions

`llm-cli completions` generates completions for the dispatcher **and** all installed `llm-cli-*` subcommands in a single script. One file gives you tab-completion for everything.

### Bash

```sh
llm-cli completions --shell bash > ~/.local/share/bash-completion/completions/llm-cli
```

### Zsh

```sh
# Ensure completions directory exists and is in fpath.
# Add to ~/.zshrc if not already present:
#   fpath=(~/.zfunc $fpath)
#   autoload -Uz compinit && compinit
mkdir -p ~/.zfunc
llm-cli completions --shell zsh > ~/.zfunc/_llm-cli
```

### Fish

```sh
llm-cli completions --shell fish > ~/.config/fish/completions/llm-cli.fish
```

Re-run after installing new subcommands to pick up their completions.

## Common flags

- `--human` — human-readable output instead of JSON
- `--debug` — log HTTP requests/responses to stderr
- `--debug=pretty` — pretty-print JSON bodies and GraphQL queries
- `--debug=curl_cmd` — print reproducible curl commands (warns about unredacted secrets)
- `--debug=pretty,curl_cmd` — both

## Design principles

See [PRINCIPLES.md](PRINCIPLES.md) for the CLI design philosophy. These tools are agent-first: JSON output, structured errors with suggestions, named flags, no interactive prompts.

## Project structure

```
packages/
  llm-cli/           # Dispatcher (std only, no deps)
  llm-cli-linear/    # Linear GraphQL client
  llm-cli-discourse/ # Discourse REST client
  llm-cli-slack/     # Slack REST client
docs/
  plans/             # Design documents
```

## Rough edges that impact usablity (via an agent)

### Critical

1. Interactive prompt blocks agents — --debug=curl_cmd triggers a Y/N confirmation via stdin in all three API crates. An agent using this flag will hang indefinitely. Fix: skip the prompt when stdin is not a TTY.

2. Errors go to stderr only — In JSON mode, errors are eprintln!'d as JSON to stderr. An agent capturing stdout gets nothing on failure. Per your own PRINCIPLES.md, errors should be structured JSON to stdout (with non-zero exit code).

### High

3. No pagination cursors — Linear detects hasNextPage but emits a human-readable truncation message instead of a cursor. Slack returns has_more internally but doesn't expose it. Discourse has no pagination at all. Agents can't page through results.

4. Truncation messages aren't machine-parseable — Linear appends a string like "(showing 25 of 100+)" rather than a structured field. Agents need "has_more": true, "cursor": "..." in the JSON envelope.

### Medium

5. Missing output fields — Linear drops assignee, created_at, updated_at, labels. Slack drops reply_count, reactions. Discourse drops tags, like_count. These are high-signal for agent decision-making.

6. Limited filtering — Linear has --mine, --team, --state but no date range, priority, or label filters. Discourse and Slack list commands have almost no filtering. Agents can't narrow queries to avoid blowing context windows.

7. No retry/rate-limit handling — All three crates fail immediately on transient HTTP errors. Slack rate-limits aggressively. A single retry with backoff would prevent many agent workflow failures.

8. Single exit code — All errors return exit code 1. Differentiating config (2), auth (3), and API (4) errors would let agents choose recovery strategies without parsing the error body.

### Low

9. No --schema flag — PRINCIPLES.md suggests a flag to output JSON Schema of input/output for automated discovery. Not implemented yet.

10. No stdin/--input for complex input — PRINCIPLES.md suggests accepting JSON input via stdin for structured data. Currently all input is via flags.

#### Implementation notes

**Pagination** — Add a `--cursor` flag to list commands. When more results exist, include a structured `pagination` object in the JSON envelope alongside `data`:

```json
{
  "success": true,
  "data": { "issues": [...] },
  "pagination": {
    "has_more": true,
    "next_cursor": "WyIyMDI2LTA0LTAxIl0"
  }
}
```

The agent passes `--cursor <value>` on the next call. Per-crate mapping:
- **Linear**: Request `endCursor` in the GraphQL `pageInfo`, pass as `after:` variable
- **Slack**: Forward `response_metadata.next_cursor`, pass as `cursor` query param
- **Discourse**: Map to `?page=N` (cursor is the page number)

The `pagination` object lives alongside `data` in the `format_success` envelope, not inside the data. This keeps the data shape stable.

**Truncation messages** — Keep the human-readable `message` field (useful for `--human` mode, harmless in JSON) but add the structured `pagination` object as the machine-parseable replacement. Both can coexist.

**Missing output fields** — Add fields to existing API structs. Since the JSON schema is append-only (per PRINCIPLES.md), this is non-breaking. Specifically:
- **Linear Issue**: `assignee { name, email }`, `createdAt`, `updatedAt`, `labels { nodes { name } }`, `team { key, name }`
- **Slack Message**: `reply_count`, `reactions [{ name, count }]`, `edited.ts`
- **Discourse Post/Topic**: `like_count`, `reply_count`, `tags`, `last_posted_at`, `category_name`

Linear requires adding fields to the GraphQL selection set. Slack and Discourse just need struct fields added with `#[serde(default)]` since the upstream REST APIs already return them.

**Implementation order**: Missing output fields first (smallest diff, biggest signal gain), then pagination (Linear first — GraphQL cursor pagination is cleanest), then deprecate the human truncation messages.
