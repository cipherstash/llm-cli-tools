# llm-cli

A CLI tool for dispatching to llm-cli-* subcommands - designed to be used by AI agents.

Git-style dispatcher that scans `$PATH` for `llm-cli-*` binaries and execs the matching subcommand.

## Usage

```sh
llm-cli linear issues list
llm-cli discourse posts latest
llm-cli slack messages read --channel general
```

## Built-in commands

- `llm-cli init` — interactive setup wizard to generate `~/.config/llm-cli/config.toml`
- `llm-cli completions --shell <bash|zsh|fish>` — generate shell completions for the dispatcher and all installed subcommands

## Install

```sh
cargo install --path .
```

## License

MIT
