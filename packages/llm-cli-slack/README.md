# llm-cli-slack

A CLI tool for interacting with the Slack API - designed to be used by AI agents.

Returns JSON by default. Use `--human` for human-readable output. Retrieves API credentials from 1Password at call time.

## Usage

```sh
# Send messages
llm-cli-slack messages send --channel general --text "hello"
llm-cli-slack messages send --channel general --text "reply" --thread-ts 1234567890.123456
llm-cli-slack messages send --input message.json
echo '{"channel": "general", "text": "hello"}' | llm-cli-slack messages send --input -

# Read messages with filtering and pagination
llm-cli-slack messages read --channel general
llm-cli-slack messages read --channel general --limit 50
llm-cli-slack messages read --channel general --oldest 1711900000 --latest 1711990000
llm-cli-slack messages read --channel general --cursor <next_cursor>

# Direct messages
llm-cli-slack messages dm --user U12345 --text "hey"

# Mentions
llm-cli-slack messages mentions --limit 10

# Channel summary
llm-cli-slack summary --channel general
llm-cli-slack summary --channel general --oldest 2026-03-30 --latest 2026-04-01

# Discover available commands and arguments
llm-cli-slack schema
```

## Output fields

Messages include: `ts`, `user`, `text`, `thread_ts`, `channel`, `reply_count`, `reactions` (name, count), `edited` (ts).

Search messages additionally include: `permalink`, `channel` (id, name).

## Pagination

The `messages read` command returns a `pagination` object when more messages are available:

```json
{
  "success": true,
  "data": { "messages": [...], "has_more": true },
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

Retries once with backoff on transient HTTP errors (429, 5xx). Respects the `Retry-After` header from Slack's rate limiter (clamped to 1-30 seconds).

## Configuration

Add to `~/.config/llm-cli/config.toml`:

```toml
[slack]
op_item_id = "your-1password-item-id"
```

## Install

```sh
cargo install --path .
```

## License

MIT
