# open-java-format-action

GitHub Action that checks Java code formatting using the [open-java-format](https://github.com/openjavaformat/open-java-format) native binary. The same check also comes as a [git pre-commit hook](#git-pre-commit-hook).

No Java, Maven, or Gradle required — downloads a native binary from Maven Central.

## Usage

```yaml
- uses: actions/checkout@v7

- uses: openjavaformat/open-java-format-action@v1
  with:
    version: '2.98.0.1'
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `version` | yes | `2.98.0.1` | Version of the open-java-format native binary, one of those on [Maven Central](https://repo1.maven.org/maven2/dev/openjavaformat/open-java-format-native/) |
| `mode` | no | `changed` | `changed` — only files from PR or push; `all` — every `.java` file in repo |

## How `mode: changed` works

- **Pull request** — gets the list of changed files from the PR via `gh pr diff`
- **Push** — gets files changed between `before` and `after` commits in the push; needs that history, so check out with `fetch-depth: 0`
- **Other events**, or a push whose `before` commit is not in the checkout — falls back to checking all `.java` files

## Behaviour

- If all files are formatted correctly → passes ✅
- If any file has formatting issues → lists unformatted files and fails ❌

Files are checked in the open-java-format style, `--ojf` (formerly `--palantir`). When the check fails, fix locally with a native binary or the runnable jar from the [open-java-format releases](https://github.com/openjavaformat/open-java-format/releases):

```bash
open-java-format --ojf --replace <files>
# or, with Java 21 or later
java -jar open-java-format-2.98.0.1-all.jar --ojf --replace <files>
```

## Excluding files

Both the action and the [pre-commit hook](#git-pre-commit-hook) read an optional
`.open-java-format-exclude` file from the repository root. Every non-empty, non-comment
line is a [git pathspec](https://git-scm.com/docs/gitglossary#Documentation/gitglossary.txt-aiddefpathspecapathspec)
(relative to the repo root); any Java file matching a pattern is skipped. If the file is
absent, nothing is excluded.

```gitignore
# .open-java-format-exclude

# Standalone jbang scripts — the formatter would rewrite their //DEPS directives into comments
samples/**

# Generated sources
**/build/generated/**
```

Notes:

- Patterns use git pathspec syntax, so both `samples/**` and `samples` match everything under `samples/`.
- Lines starting with `#` are comments; blank lines are ignored.
- This file is the **single source of exclusions** — it is shared by the action and the hook, so there is no separate action input to configure.

## Recommended setup

```yaml
on:
  pull_request:
  push:
    branches: [main]

jobs:
  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: openjavaformat/open-java-format-action@v1
        with:
          version: '2.98.0.1'
          mode: ${{ github.event_name == 'push' && 'all' || 'changed' }}
```

## Git pre-commit hook

You can also use the same formatter as a local git hook to catch formatting issues before they reach CI.

Copy [`pre-commit`](pre-commit) to your project:

```bash
cp pre-commit .git/hooks/pre-commit
# or with a custom hooks directory
cp pre-commit .githooks/pre-commit
git config core.hooksPath .githooks
```

The hook automatically downloads the native binary on first run and caches it in `~/.cache/open-java-format`, then checks only staged `.java` files on each commit. It also honours the same [`.open-java-format-exclude`](#excluding-files) file as the action.

The hook pins its own formatter version in `FORMATTER_VERSION` at the top of the script — keep it in step with the `version` your workflow passes to the action.

## Supported platforms

- Linux x86_64 (glibc)
- Linux aarch64 (glibc)
- macOS aarch64 (Apple Silicon)
- macOS x86_64 (Intel)

There is no native binary for Windows or for musl-based Linux such as Alpine.
