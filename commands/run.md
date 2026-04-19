---
description: Compress a Markdown instruction file to validated YAML with round-trip review.
argument-hint: [<file> | --all <path>] [--dry-run] [--include-all]
---

Compress one or more Markdown instruction files through the Terse-MD pipeline (normalize → compress → validate → decompress → human review). Write `.approved.yaml` only on explicit approval.

## Step 1 — Parse arguments

The user invoked this command with: $ARGUMENTS

Parse flags and positional arguments in this order, **before touching any path on disk**:

1. Tokenize `$ARGUMENTS` on whitespace. Let the token list be `{TOKENS}`.
2. If `--dry-run` appears in `{TOKENS}`, set `{dry_run}` to `true` and remove that token from the list; otherwise `{dry_run}` is `false`.
3. If any token in `{TOKENS}` is exactly equal to the string `--include-all` (no prefix or suffix variation), set `{include_all}` to `true` and remove that token from the list; otherwise `{include_all}` is `false`. Substring matches like `--include-all-the-things` or `--include-all=true` do NOT count and are passed through to subsequent parsing steps (where they'll fail as bad paths).
4. If the first remaining token is `--all`, set `{mode}` to `all` and take the second remaining token as `{raw_path}`. If no second token is present, print `Usage: /terse-md:run [<file> | --all <path>] [--dry-run] [--include-all]` and STOP. Discard any further tokens.
5. Otherwise, if there is exactly one remaining token, set `{mode}` to `file` and take it as `{raw_path}`.
6. Otherwise (no remaining tokens), set `{mode}` to `all` and `{raw_path}` to `.`. In this case — and only this case — print a single line `Interpreting as: /terse-md:run --all .` before continuing, so the user knows what scope was inferred.

Only **after** parsing, resolve `{raw_path}` to an absolute path, passing it via env var to avoid shell metacharacter injection:

```bash
P="<raw_path>" realpath -- "$P"
```

Store the resolved path as `{TARGET_PATH}`.

If `{mode}` is `file`, confirm `{TARGET_PATH}` is a regular file and not a symlink:

```bash
P="<TARGET_PATH>" bash -c 'test -f "$P" && test ! -L "$P" && echo regular || echo bad'
```

If the result is `bad`, print `Error: <TARGET_PATH> is not a regular file (or is a symlink). Refusing to run. If you meant to scan a directory, use /terse-md:run --all <path>.` and STOP.

## Step 2 — Discover candidate files (only when `{mode}` is `all`)

Skip this step if `{mode}` is `file`; set the candidate list to `[{TARGET_PATH}]` and set `{SKIPPED_BY_DEFAULT}` to empty. (Explicit `file` mode bypasses the default-skip filter — if you name a file directly, Terse-MD trusts you. If `{include_all}` is also `true` in this case, it has no effect; print one line `Note: --include-all has no effect in file mode (the default-skip filter is already bypassed).` once after parsing, then continue.)

Use the Glob tool to find files matching each of these patterns under `{TARGET_PATH}`:
- `**/CLAUDE.md`
- `**/SKILL.md`
- `**/user_*.md`
- `**/feedback_*.md`
- `**/reference_*.md`
- `**/project_*.md`
- `**/MEMORY.md`

Combine all results and deduplicate by absolute path.

Filter out:
- Any path containing `/.git/`, `/node_modules/`, `/venv/`, `/.venv/`, `/__pycache__/`, or `/target/` as a path component.
- Any file whose name ends in `.approved.yaml`.
- Any symlink (`test -L`).
- Any candidate whose real path is outside the scan root. Run one Bash call per candidate with paths passed via env vars:

  ```bash
  C="<candidate-absolute-path>" R="<TARGET_PATH>" bash -c '
    REAL_C=$(realpath -- "$C") || exit 1
    R="${R%/}"                       # strip trailing slash; realpath normally removes it already
    if [ -z "$R" ]; then             # R was "/" (or empty) — accept everything absolute
      case "$REAL_C" in /*) exit 0 ;; *) exit 2 ;; esac
    else
      case "$REAL_C" in "$R"|"$R"/*) exit 0 ;; *) exit 2 ;; esac
    fi'
  ```

  Non-zero exit means the candidate's real path escapes `{TARGET_PATH}` (typically via a symlinked directory component). Drop the candidate and continue.

This plugin does not interpret `.gitignore` globs. If you want a path excluded, don't include it under `{TARGET_PATH}`.

### Step 2a — Default-skip filter (only when `{mode}` is `all`)

Unless `{include_all}` is `true`, partition the remaining files into the candidate list and `{SKIPPED_BY_DEFAULT}`. A file goes to `{SKIPPED_BY_DEFAULT}` if its basename matches either:

- Exactly `MEMORY.md` — reason: `index file; already one-liners`.
- Matches the shell glob `project_*.md` — reason: `narrative scratchpad; short half-life`.

Both checks are **case-sensitive on the basename**. `Project_foo.md`, `MEMORY.MD`, and `memory.md` are NOT default-skipped; only the exact-case forms shown above match.

All other files stay in the candidate list.

If `{include_all}` is `true`, the candidate list is everything that survived the discover + filter pass earlier in Step 2 and `{SKIPPED_BY_DEFAULT}` is empty.

If the candidate list is empty, print `No candidate files under {TARGET_PATH}.` (with the resolved path wrapped in backticks). If `{SKIPPED_BY_DEFAULT}` is non-empty, let `{SKIP_COUNT}` be the count of `{SKIPPED_BY_DEFAULT}` and append a second line: `{SKIP_COUNT} file(s) skipped by default — re-run with --include-all to include.` (substituting the literal count). Then STOP.

Sort the candidate list alphabetically by absolute path.

## Step 3 — Size and count caps

Count the candidate files. If the count exceeds **250**, print the count and a message saying `Refusing to process more than 250 files in a single run; narrow the path and retry.` and STOP.

For each candidate, measure bytes via `P="<path>" wc -c -- "$P"`. If any single file exceeds **1,048,576 bytes (1 MiB)**, skip that file and note it in the final summary.

## Step 4 — First-run message (once per invocation, before the first file)

If `{mode}` is `all` and `{SKIPPED_BY_DEFAULT}` is non-empty, print this block first (before the main message), then a blank line:

```
Skipped by default (re-run with --include-all to include):
  <path-relative-to-TARGET_PATH>    <reason>
  ...
```

Before printing any path, **sanitize the displayed string**: replace any byte less than `0x20` or equal to `0x7f` with the literal character `?`. This prevents a filename containing control characters (e.g. `\r`, `\n`, ANSI escape sequences) from rewriting earlier lines or forging fake entries in the rendered block. Sanitization is display-only; the actual path used internally is unchanged.

Indent each entry by 2 spaces. Left-align the sanitized paths in a single column; pad each with spaces so the reason column begins 2 spaces past the longest path in this block. Reasons: `MEMORY.md` → `index file; already one-liners`. `project_*.md` → `narrative scratchpad; short half-life`.

**Cap:** if `{SKIPPED_BY_DEFAULT}` contains more than **10** files, print only the first 10 (sorted alphabetically) and append a final summary line in the same indent: `(... and {EXTRA} more — {project_count} project_*.md, {memory_count} MEMORY.md)` where `{EXTRA}` = total minus 10, and the per-type counts are over ALL skipped files (not just the omitted tail). Keeps the prompt readable for auto-memory directories that may default-skip 100+ project files.

Then print the following block verbatim:

```
Terse-MD will:
  - Read each file and run 3 transform steps (normalize, compress, decompress)
  - Show you the compressed result alongside a prose version of it
  - Write a new `.approved.yaml` only if you approve it
  - Never modify, move, or delete your original files

What to know:
  - Wording will shift even when meaning is preserved. Read the review carefully.
  - If you approve a drifted version, Claude will follow the drifted version.
  - `.approved.yaml` is what Claude reads. Edit it directly for small changes;
    re-run from source for large ones. The review step does not re-run on
    direct edits — that is your call.
  - Each step runs a Sonnet subagent.
```

Then call `AskUserQuestion` with question "Ready?" and a single option "Continue". If the user cancels or does not choose Continue, STOP without writing anything.

## Step 5 — Create an isolated work directory

Create a single private work directory for this entire invocation:

```bash
mktemp -d /tmp/terse-md-run.XXXXXXXX
```

Capture the absolute path as `{WORK_DIR}`. Immediately assert the directory exists and is writable:

```bash
D="<WORK_DIR>" bash -c '[ -d "$D" ] && [ -w "$D" ]'
```

If the assertion fails, print `Error: failed to create a usable work directory under /tmp.` and STOP.

All temp files for every file processed in this invocation go inside `{WORK_DIR}`. Do not call `mktemp` again later.

## Step 6 — Per-file pipeline

For each file in the candidate list (index `i`, starting at 1), run steps 6a–6n in order.

### 6a — Read source, hash, generate nonce

Read the file content via the Read tool. Store it as `{SOURCE}`. Then run exactly one Bash call:

```bash
P="<absolute-path>" sha256sum -- "$P" | awk '{print $1}'
```

Capture the output as `{SOURCE_HASH}`.

Generate a fresh nonce for this file via one Bash call:

```bash
head -c 32 /dev/urandom | base64 | tr -dc 'a-zA-Z0-9' | head -c 16
```

Store as `{NONCE_NORM}`. Generate two more the same way and store as `{NONCE_COMP}` and `{NONCE_DEC}` — each subagent invocation uses its own unique nonce.

Per-file temp paths live under `{WORK_DIR}/file-{i}-*` (use the loop index `i`, not `$$`).

### 6b — Normalize

Read `${CLAUDE_PLUGIN_ROOT}/prompts/normalize.md`. Substitute `{NONCE_NORM}` in place of every occurrence of the literal string `{NONCE}` in that prompt.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted normalize prompt, followed by a blank line, then the literal line `---BEGIN SOURCE {NONCE_NORM}---`, then a newline, then `{SOURCE}` verbatim, then a newline, then the literal line `---END SOURCE {NONCE_NORM}---`.

Capture the subagent's output as `{NORMALIZED}`.

### 6c — Show normalize diff

1. Write `{SOURCE}` to `{WORK_DIR}/file-{i}-src.md` using the **Write tool**.
2. Write `{NORMALIZED}` to `{WORK_DIR}/file-{i}-norm.md` using the **Write tool**.
3. Run Bash:

```bash
diff -u -- "<WORK_DIR>/file-{i}-src.md" "<WORK_DIR>/file-{i}-norm.md" || true
```

Print the diff output under a heading `## Normalize diff`. If there is no diff output, print `(no changes in normalize step)`.

### 6d — Compress

Read `${CLAUDE_PLUGIN_ROOT}/prompts/compress.md` and `${CLAUDE_PLUGIN_ROOT}/schema.yaml`. Substitute the verbatim contents of `schema.yaml` in place of the literal string `{SCHEMA}`, and `{NONCE_COMP}` in place of every occurrence of `{NONCE}`, in the compress prompt.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted compress prompt, followed by a blank line, then the literal line `---BEGIN INPUT {NONCE_COMP}---`, then a newline, then `{NORMALIZED}` verbatim, then a newline, then the literal line `---END INPUT {NONCE_COMP}---`.

Capture the subagent's output as `{COMPRESSED_YAML}`.

### 6e — Validate

Write `{COMPRESSED_YAML}` to `{WORK_DIR}/file-{i}-out.yaml` using the **Write tool**.

Run Bash — pass paths as positional arguments, never interpolated into the Python source string:

```bash
python3 -c '
import sys, yaml, jsonschema
doc = yaml.safe_load(open(sys.argv[1]))
schema = yaml.safe_load(open(sys.argv[2]))
jsonschema.validate(doc, schema)
print("ok")
' "<WORK_DIR>/file-{i}-out.yaml" "${CLAUDE_PLUGIN_ROOT}/schema.yaml"
```

If `python3`, `yaml`, or `jsonschema` are unavailable, fall back to a structural closure check: read `${CLAUDE_PLUGIN_ROOT}/schema.yaml` and confirm that every top-level key in `{COMPRESSED_YAML}` is one of `meta`, `rules`, `transforms`, `triggers`, `thresholds`, and that every key inside those sections exists in the schema's `properties` definition for that section.

If validation fails, append the validation error message to the compress prompt as a new section `# Validation error — retry` and invoke the compress subagent one more time with the same `{NORMALIZED}` input and the **same** `{NONCE_COMP}`. Capture the new output as `{COMPRESSED_YAML}`. Re-validate using the same logic.

If validation still fails on the second attempt, print the error and `Aborting this file after two failed validation attempts.` In `{mode} = all` mode continue to the next file; otherwise STOP.

### 6f — Inject `meta.source_hash`

Re-parse and re-serialize the YAML via Bash to guarantee a deterministic, safe transformation — never edit YAML text by hand:

```bash
python3 -c '
import sys, yaml
with open(sys.argv[1]) as f:
    doc = yaml.safe_load(f) or {}
doc.setdefault("meta", {})
doc["meta"].setdefault("version", 1)
doc["meta"]["source_hash"] = sys.argv[2]
with open(sys.argv[1], "w") as f:
    yaml.safe_dump(doc, f, sort_keys=False, default_flow_style=False)
' "<WORK_DIR>/file-{i}-out.yaml" "<SOURCE_HASH>"
```

Re-validate by running the Step 6e validator again. If this re-validation fails, print the error and abort this file.

Read the re-written `{WORK_DIR}/file-{i}-out.yaml` back into a variable as `{FINAL_YAML}`.

### 6g — Decompress

Read `${CLAUDE_PLUGIN_ROOT}/prompts/decompress.md`. Substitute `{NONCE_DEC}` in place of every occurrence of the literal string `{NONCE}` in that prompt.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted decompress prompt, followed by a blank line, then the literal line `---BEGIN YAML {NONCE_DEC}---`, then a newline, then `{FINAL_YAML}` verbatim, then a newline, then the literal line `---END YAML {NONCE_DEC}---`.

Capture the subagent's output as `{RECONSTRUCTED}`.

### 6h — Review display

Emit three Markdown sections in order, separated by horizontal rules (`---`). Render vertically — do not try to make columns.

1. A level-2 heading `## Original (normalized)` followed by a blank line and the full `{NORMALIZED}` text.
2. A horizontal rule.
3. A level-2 heading `## Reconstructed from YAML` followed by a blank line and the full `{RECONSTRUCTED}` text.
4. A horizontal rule.
5. A level-2 heading `## Compressed YAML` followed by a blank line and the `{FINAL_YAML}` content inside a triple-backtick fence tagged `yaml`.

No commentary between sections.

### 6i — Per-file decision

Call `AskUserQuestion` with:
- question: "Does the reconstructed version preserve what you meant?"
- options: `["Approve", "Reject", "Skip"]`

### 6j — Approve: compute and contain the output path

Compute the output path. Source file basename loses its trailing `.md` and gains `.approved.yaml`; the output goes in the same directory as the source.

Scan-root containment is enforced upstream (Step 1 for `file` mode; Step 2's real-path filter for `all` mode). Step 6j re-checks only the file leaf: that the source is still a regular file and still not a symlink at write time. This closes any TOCTOU window between earlier checks and the write:

```bash
S="<absolute-source-path>"
[ -f "$S" ] && [ ! -L "$S" ] || exit 1
dirname -- "$S"
```

If the Bash call exits non-zero or stdout is not a valid absolute directory path, print `Error: source file disappeared or became a symlink — refusing to write.` and treat this as `Reject` (see 6l). Otherwise, `{REAL_DIR}` is the stdout value and output will be written under it.

If the output file already exists at `{REAL_DIR}/<stem>.approved.yaml`, call `AskUserQuestion` with:
- question: "`<output-path>` already exists. Overwrite?"
- options: `["Overwrite", "Skip this file"]`

If the user picks `Skip this file`, treat it as Skip (see 6m).

### 6k — Approve (not dry-run)

Write `{FINAL_YAML}` to `{REAL_DIR}/<stem>.approved.yaml` via the Write tool. Print:

```
Written: <absolute-output-path>
```

### 6l — Approve (dry-run)

Print:

```
[dry-run] Would write to <absolute-output-path>
```

Do not write any file.

### 6m — Reject

Print:

```
User rejected. Edit <source-path> and re-run.
```

In `{mode} = all` mode, STOP the entire run immediately — do not process remaining files.

### 6n — Skip

Print `Skipped.` and continue to the next file in `{mode} = all` mode. In `{mode} = file` mode, STOP.

### 6o — Per-file cleanup

After each file (regardless of outcome), delete the per-file temp files inside `{WORK_DIR}`:

```bash
rm -f -- "<WORK_DIR>/file-{i}-src.md" "<WORK_DIR>/file-{i}-norm.md" "<WORK_DIR>/file-{i}-out.yaml"
```

The `{WORK_DIR}` itself is removed in Step 7.

## Step 7 — Final summary and cleanup

Print:

```
---
Terse-MD run complete.
  Processed: N
  Approved:  N
  Rejected:  N
  Skipped:   N

Written files:
  - <path>
  - <path>
```

If no files were written (dry-run or all rejected/skipped), omit the "Written files:" block.

Then remove the work directory:

```bash
rm -rf -- "<WORK_DIR>"
```

## Constraints

- Never modify, move, or rename the source file.
- Never write any output file outside the source file's realpath-resolved parent directory.
- Never write a file without an explicit `Approve` (and `Overwrite` if the target exists) from the user.
- In `{mode} = all` mode, a Reject stops the entire run; it does not merely skip that file.
- The compress subagent is retried at most once per file. Do not loop beyond two attempts.
- Temp files live inside `{WORK_DIR}` (a `mktemp -d` directory); never use bare `/tmp/terse-md-*` paths.
- Do not pass the `{SOURCE_HASH}` to the compress subagent — inject it after validation in Step 6f.
- In dry-run mode, print what would be written but write nothing to disk.
- Files larger than 1 MiB are skipped in Step 3.
- No more than 250 files per invocation.
