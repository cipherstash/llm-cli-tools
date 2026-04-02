# llm-cli-discourse

A CLI tool for interacting with the Discourse API - designed to be used by AI agents.

Returns JSON by default. Use `--human` for human-readable output. Retrieves API credentials from 1Password at call time. Supports multiple instances via `--instance`.

## Usage

```sh
# List latest posts with pagination
llm-cli-discourse posts latest
llm-cli-discourse posts latest --page 2

# Get a topic
llm-cli-discourse posts get --id 42

# Create topics
llm-cli-discourse posts create --title "Topic" --category general --raw "Body"
llm-cli-discourse posts create --input topic.json
echo '{"title": "Topic", "category": "general", "raw": "Body"}' | llm-cli-discourse posts create --input -

# Delete a topic
llm-cli-discourse posts delete --id 42

# Comments
llm-cli-discourse comments create --topic-id 42 --raw "Reply"
llm-cli-discourse comments delete --id 55

# Discover available commands and arguments
llm-cli-discourse schema
```

## Output fields

Topics include: `id`, `title`, `slug`, `category_id`, `posts_count`, `views`, `like_count`, `reply_count`, `last_posted_at`, `tags`.

Posts include: `id`, `topic_id`, `topic_title`, `username`, `raw`, `cooked`, `post_number`, `created_at`, `like_count`, `reply_count`, `score`.

## Pagination

The `posts latest` command supports `--page N` (starts at 0). The JSON response includes a `pagination` object when more results may be available:

```json
{
  "success": true,
  "data": { "posts": [...] },
  "pagination": { "has_more": true, "next_cursor": "1" }
}
```

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

Retries once with 1s backoff on transient HTTP errors (429, 5xx). Delete operations are not retried.

## Configuration

Add to `~/.config/llm-cli/config.toml`:

```toml
[discourse.my-forum]
base_url = "https://forum.example.com"
op_item_id = "your-1password-item-id"
api_username = "your-username"
```

## Install

```sh
cargo install --path .
```

## License

MIT
