# CLI Design Principles for LLM Agent Tools

These CLIs are primarily designed to be invoked by LLM agents (e.g., Claude) as tools, not by humans directly.

## Output

- **JSON to stdout, diagnostics to stderr.** This separation is critical for clean parsing.
- **Return only high-signal data.** LLM context windows are finite — implement pagination, filtering, and truncation with sensible defaults. When truncating, include a message that steers the agent toward a more targeted query.
- **Keep schemas stable and append-only.** Adding fields is fine; removing/renaming is breaking.
- Disable colors and terminal formatting in machine-readable output.

## Error Handling

- **Errors are instructions, not diagnostics.** Instead of `Error: 404`, return structured errors with a code, a human-readable message, and a suggestion for what to try next.
- **Include examples of correct usage in error messages** when the agent provides malformed input.
- Use structured error objects:
  ```json
  {
    "success": false,
    "error": {
      "code": "INVALID_DATE_FORMAT",
      "message": "Date must be ISO 8601 format (YYYY-MM-DD). Received: '03/15/2025'",
      "suggestion": "Use '2025-03-15' instead"
    }
  }
  ```
- **Exit 0 on success, non-zero on failure.** But carry real error semantics in the JSON body, not the exit code.

## Argument Design

- **Named flags over positional args.** `--user-id 123` is unambiguous; a bare `123` is not.
- **Enums for constrained choices.** Prevents hallucinated values.
- **Minimize required params.** Every required param is a chance for the agent to get it wrong. Provide sensible defaults.
- **Never prompt for interactive input.** Detect non-TTY stdin automatically.
- Accept JSON input via stdin or a `--input` flag for complex structured input.

## Discoverability

- **Detailed descriptions are the single most important factor.** 3-4+ sentences covering what the tool does, when to use it, when *not* to use it, and caveats.
- Consider a `--schema` flag that outputs a JSON Schema of the tool's input/output for automated discovery.

## Safety & Idempotency

- **Separate read and write operations** (CQRS). Lets agents safely explore before acting.
- **Make operations idempotent where possible.** Same args twice = same result, no side effects.
- **Annotate risk** — following MCP conventions: `readOnlyHint`, `destructiveHint`, `idempotentHint`.

## Granularity

- **Single responsibility per tool.** Avoid "god tools" with an `action` param that multiplexes operations.
- Closely related operations *can* be consolidated if it reduces selection ambiguity — test both approaches.
- Use meaningful namespace prefixes when you have multiple tools.

## Standards

- **MCP (Model Context Protocol)** is the emerging standard. Its stdio transport is a natural fit for CLIs — a binary can serve double duty as both a direct CLI tool and an MCP server.

## Key Sources

- [Anthropic: Implement Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use)
- [Anthropic: Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [MCP Specification: Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [Command Line Interface Guidelines](https://clig.dev/)
- [Rust CLI Recommendations](https://rust-cli-recommendations.sunshowers.io/)
