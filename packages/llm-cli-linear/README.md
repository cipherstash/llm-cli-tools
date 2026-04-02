# llm-cli-linear

A CLI tool for interacting with the Linear API - designed to be used by AI agents.

Returns JSON by default. Use `--human` for human-readable output. Retrieves API credentials from 1Password at call time.

## Usage

```sh
# List issues with filters
llm-cli-linear issues list --limit 10
llm-cli-linear issues list --mine --team ENG --state "In Progress"
llm-cli-linear issues list --priority 1 --label bug
llm-cli-linear issues list --cursor <next_cursor>

# Get a single issue
llm-cli-linear issues get --id PROJ-123

# Create issues
llm-cli-linear issues create --title "Bug" --team ENG --priority 2
llm-cli-linear issues create --input issue.json
echo '{"title": "Bug", "team": "ENG"}' | llm-cli-linear issues create --input -

# Close an issue
llm-cli-linear issues close --id PROJ-123

# Discover available commands and arguments
llm-cli-linear schema
```

## Output fields

Issues include: `id`, `identifier`, `title`, `state`, `priority`, `description`, `url`, `assignee` (name, email), `team` (key, name), `labels`, `created_at`, `updated_at`.

## Pagination

List commands return a `pagination` object when more results are available:

```json
{
  "success": true,
  "data": { "issues": [...] },
  "pagination": { "has_more": true, "next_cursor": "..." }
}
```

Pass the cursor to fetch the next page: `--cursor <next_cursor>`.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 2 | Configuration error |
| 3 | Authentication error (1Password) |
| 4 | API error |
| 5 | Invalid CLI input |

Errors are JSON to stdout (not stderr) in default mode.

## Resilience

Retries once with 1s backoff on transient HTTP errors (429, 5xx).

## Configuration

Add to `~/.config/llm-cli/config.toml`:

```toml
[linear]
op_item_id = "your-1password-item-id"
```

## Install

```sh
cargo install --path .
```

## License

MIT
