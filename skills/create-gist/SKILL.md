---
name: create-gist
description: Creates a GitHub Gist from content in the current conversation or a file. Secret by default; pass --public to make it public.
disable-model-invocation: false
model: sonnet
---

# Create Gist

## 0. Preflight check

Verify `gh` is available and authenticated:

```sh
gh auth status
```

Handle errors distinctly:
- If the command is **not found** (`command not found` or similar): stop and tell the human "Install the GitHub CLI first: https://cli.github.com"
- If `gh` is installed but **not authenticated**: stop and tell the human "Run `gh auth login` first, then try again."

Do not proceed until this check passes.

## 1. Gather content and options

Collect everything needed to build the gist — in order of priority:

- **File path passed as an argument** (e.g. `/create-gist fix.py`) — use the file directly; skip temp file creation
- **Other arguments or flags** (e.g. `/create-gist the snippet above --public`)
- **Content referenced in the conversation** — code blocks, error output, diffs

Parse the following options from the arguments or conversation:

| Option | Default | Meaning |
|--------|---------|---------|
| `--public` | off | Make the gist public instead of secret |
| `--desc <text>` | inferred | Gist description |
| `--filename <name>` | inferred | Override the filename shown in the gist |

If no content is identifiable, ask the human what to put in the gist before proceeding.

**Multi-file gists are out of scope.** If multiple files are referenced, ask the human to pick one.

## 2. Infer filename and description

**Filename**: If a real file was referenced, use its basename. If `--filename` was passed, use that value. Otherwise infer from context (e.g. `fix.py`, `output.log`, `snippet.md`). Default to `gist.txt` only as a last resort. The extension matters — `gh` uses it for syntax highlighting.

**Description**: Summarize the purpose in one short line. Pull from conversation context, the skill args, or the content itself. Keep it under 72 characters.

## 3. Determine visibility

- If `--public` was passed → public gist
- Otherwise → **secret gist** (default)

## 4. Draft and review

Present a summary to the human before creating anything:

```
File:        <filename>
Description: <description>
Visibility:  <secret or public>
Content:
---
<first ~20 lines or full content if short>
---
```

If the content contains secrets (API keys, tokens, passwords, private URLs) — flag them explicitly. Warn the human that **even secret gists are accessible to anyone with the URL**. Let them decide whether to proceed.

Ask for confirmation. Do not create the gist until the human says yes. If running non-interactively or if the human passed `--yes`, skip the confirmation prompt.

## 5. Create the gist

**If the source is a file on disk**, pass it directly:

```sh
# Secret (default)
gh gist create --desc "<description>" --filename "<filename>" <filepath>

# Public
gh gist create --public --desc "<description>" --filename "<filename>" <filepath>
```

**If the source is content from the conversation**, write it to a temp file first using `mktemp` with the correct extension (e.g. `mktemp /tmp/gist-XXXXXX.py`), then run:

```sh
# Secret (default)
gh gist create --desc "<description>" --filename "<filename>" <tmpfile>

# Public
gh gist create --public --desc "<description>" --filename "<filename>" <tmpfile>
```

After the gist is created (or if creation fails), **delete the temp file immediately**:

```sh
rm -f <tmpfile>
```

## 6. Report back

Return:
- The gist URL
- The visibility (secret or public)
- The filename and description
- A one-liner on what was created
