# ai-check

`ai-check` is a one-command linter for AI coding agents. Run it on any file an
agent just wrote or modified: it picks the right linter for that file type,
prints what is wrong, and returns a non-zero exit status when a check fails.
Mistakes get fixed immediately instead of surfacing later in CI or review.

## What it checks

| Extension(s) | Primary tool | Fallback |
| ----------- | ------------------------------------ | ---------------------- |
| `md` | `markdownlint` | — |
| `md` with Mermaid blocks | `npx -y @mermaid-js/mermaid-cli` | — |
| `js`, `jsx`, `ts`, `tsx` | `npx -y oxlint@latest` | — |
| `json` | `jq .` | — |
| `py` | `uvx ruff check` | `python3 -m py_compile` |
| `sh`, `bash`, `zsh` | `shellcheck` | `bash -n` |
| `yaml`, `yml` | `yamllint` | — |
| `go` | `go vet` | `go fmt -n` |
| `rs` | `rustc --parse-only` | — |
| `c`, `cpp`, `h`, `hpp` | `gcc -fsyntax-only` | `clang -fsyntax-only` |
| `html`, `xml` | `xmllint --noout` | — |

Any other extension is skipped with a notice and exits `0`.

## Installation

### Requirements

The script itself needs only `bash`. Each tool is used only for its own file
type, so install the ones for the languages you work with. Rows with a
fallback degrade to the lighter check when the primary tool is unavailable.

```sh
# macOS (Homebrew)
brew install markdownlint-cli shellcheck yamllint jq node go gcc libxml2

# uv provides `uvx`, which runs ruff without a global install
curl -LsSf https://astral.sh/uv/install.sh | sh
```

`npx` downloads oxlint and mermaid-cli on first use, and `uvx` downloads ruff
on first use. First runs are therefore slower and need network access.

### Install the script

Make it executable and put it on your `PATH`:

```sh
chmod +x ai-check
sudo ln -sf "$(pwd)/ai-check" /usr/local/bin/ai-check
```

Or copy it into a directory that is already on your `PATH`, such as
`~/.local/bin`. Verify the installation:

```console
$ ai-check
Usage: ai-check <filepath>
```

## Usage

Pass exactly one filepath per invocation:

```sh
ai-check src/app.py
ai-check docs/architecture.md
```

Diagnostics from the underlying linter are printed directly. Fix what it
reports and run it again until the file passes.

### Exit status

- `1` when no filepath is given
- otherwise, the exit status of the last check that ran

The fallback runs whenever the primary tool exits non-zero, including when it
reports findings. For Python, shell, and Go this means a file can print
findings yet still exit `0` because the fallback (a syntax-only check)
succeeded. Always read the output, not just the exit code.

## Using it with AI agents

The point of `ai-check` is to be wired into an agent's instructions so the
agent validates every file it writes. Add something like this to the
instructions file your agent reads (`AGENTS.md`, `CLAUDE.md`,
`.cursorrules`, `INSTRUCTIONS.md`):

```md
# IMPORTANT

Use ai-check every time a file is saved to check that its content is valid:

    ai-check <filepath>
```

## Notes and limitations

- One file per invocation; directories are not supported.
- stderr of the primary tool is discarded in the Markdown, Python, shell,
  Go, and C/C++ branches. markdownlint-cli2 prints findings to stderr, so
  prefer `markdownlint-cli` if you want Markdown diagnostics visible.
- `rustc --parse-only` has been removed from recent Rust toolchains; on
  those toolchains every `.rs` check fails regardless of file content.
- HTML is validated as XML, so valid HTML5 (for example unclosed void
  tags) can be flagged.
- `go fmt -n` only lists files whose formatting differs; it never modifies
  them.
