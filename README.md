# open-java-format-action

GitHub Action that checks Java code formatting using the [open-java-format](https://github.com/openjavaformat/open-java-format) native binary. The same check also comes as a [git pre-commit hook](#git-pre-commit-hook).

No Java, Maven, or Gradle required — downloads a native binary from Maven Central.

## Usage

The version of open-java-format to check with goes in `.open-java-format-version` at the repository root, one of the versions on [Maven Central](https://repo1.maven.org/maven2/dev/openjavaformat/open-java-format-native/):

```
21.0.1
```

The action and the [pre-commit hook](#git-pre-commit-hook) both read that file, so the two cannot drift apart; without it, both stop and ask for it. The action itself needs no inputs:

```yaml
- uses: actions/checkout@v7

- uses: openjavaformat/open-java-format-action@v4
```

v4 is for open-java-format 2.98.0.2 and later, which has a single style. v3 and v2 are for the same versions but take the version as a `version` input instead of the file. A workflow that pins 2.98.0.1 stays on `@v1`, which passes that version the `--ojf` flag it needs.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | no | `changed` | `changed` — only files from PR or push; `all` — every `.java` file in repo |
| `code-scanning` | no | `false` | Also report the version check to [code scanning](#code-scanning); needs the `security-events: write` permission |

## How `mode: changed` works

- **Pull request** — gets the list of changed files from the PR via `gh pr diff`
- **Push** — gets files changed between `before` and `after` commits in the push; needs that history, so check out with `fetch-depth: 0`
- **Other events**, or a push whose `before` commit is not in the checkout — falls back to checking all `.java` files

## Behaviour

- If `.open-java-format-version` is missing → stops and asks for the file ❌
- If all files are formatted correctly → passes ✅
- If any file has formatting issues → lists unformatted files and fails ❌
- If a newer open-java-format release is out → warns in the job log, and raises a [code scanning alert](#code-scanning) when that is turned on

Files are checked in the open-java-format style, the formatter's only style. When the check fails, fix locally with a native binary or the runnable jar from the [open-java-format releases](https://github.com/openjavaformat/open-java-format/releases):

```bash
open-java-format --replace <files>
# or, with Java 21 or later
java -jar open-java-format-21.0.1-all.jar --replace <files>
```

## Excluding files

Both the action and the [pre-commit hook](#git-pre-commit-hook) read an optional
`.open-java-format-exclude` file from the repository root. Every non-empty, non-comment
line is a [git pathspec](https://git-scm.com/docs/gitglossary#Documentation/gitglossary.txt-aiddefpathspecapathspec)
(relative to the repo root); any Java file matching a pattern is skipped. If the file is
absent, nothing is excluded.

```gitignore
# .open-java-format-exclude

# Code copied from another project — kept as it came
third_party/**

# Generated sources
**/build/generated/**
```

Notes:

- Patterns use git pathspec syntax, so both `third_party/**` and `third_party` match everything under `third_party/`.
- Lines starting with `#` are comments; blank lines are ignored.
- This file is the **single source of exclusions** — it is shared by the action and the hook, so there is no separate action input to configure.

## Recommended setup

Both setups check the files of a pull request and every file on a push to `main`, with the version from `.open-java-format-version`. They differ in where a newer open-java-format release is announced.

### With the version check in the job log

No permissions beyond the defaults. When a newer release is out, the job log gets a warning.

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

      - uses: openjavaformat/open-java-format-action@v4
        with:
          mode: ${{ github.event_name == 'push' && 'all' || 'changed' }}
```

### With the version check in code scanning too

The same, plus a [code scanning alert](#code-scanning) that stays open until the version is updated. It takes `code-scanning: true` and the `security-events: write` permission; the `permissions` block replaces the job's default permissions, so it names everything else the job needs as well.

```yaml
on:
  pull_request:
  push:
    branches: [main]

jobs:
  format:
    runs-on: ubuntu-latest
    permissions:
      contents: read # the checkout
      pull-requests: read # mode: changed reads the files of the pull request
      security-events: write # the code scanning upload
    steps:
      - uses: actions/checkout@v7

      - uses: openjavaformat/open-java-format-action@v4
        with:
          mode: ${{ github.event_name == 'push' && 'all' || 'changed' }}
          code-scanning: true
```

## Code scanning

The action compares the version in `.open-java-format-version` with the latest [open-java-format release](https://github.com/openjavaformat/open-java-format/releases) and warns in the job log when a newer one is out. A log warning is easy to miss, so the check can also be reported to code scanning: the action then uploads a SARIF file with one alert while the version is behind, and with none once it is current, which closes the alert. The alert shows up under **Security → Code scanning** and in the **Code scanning results** check on pull requests, and points at `.open-java-format-version`.

This is off by default, because the upload needs the `security-events: write` permission, which the default token does not have. To turn it on, set `code-scanning: true` and give the job the permission, as in the [second recommended setup](#with-the-version-check-in-code-scanning-too). If the permission is missing anyway, the upload is skipped with a line in the log, the check passes as before, and the log warning stays. Code scanning is free on public repositories; a private repository needs [GitHub Code Security](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-security-and-analysis-settings-for-your-repository) enabled. Pull requests from forks run with a read-only token and get the log warning only.

One report per job is enough. When the job runs on a matrix, turn it on for one leg only:

```yaml
        with:
          code-scanning: ${{ matrix.os == 'ubuntu-latest' }}
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

The hook reads the version from the same [`.open-java-format-version`](#usage) file as the action and honours the same [`.open-java-format-exclude`](#excluding-files) file. It downloads the native binary on first run and caches it in `~/.cache/open-java-format`, then checks only staged `.java` files on each commit.

## Supported platforms

- Linux x86_64 (glibc)
- Linux aarch64 (glibc)
- macOS aarch64 (Apple Silicon)
- macOS x86_64 (Intel)
- Windows x86_64

There is no native binary for Windows on ARM or for musl-based Linux such as Alpine.
