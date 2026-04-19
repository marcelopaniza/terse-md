---
description: Scan instruction files under a path, count tokens, sample-compress the largest file, report estimated savings. Writes nothing.
argument-hint: [<path>] [--include-all]
---

Scan candidate instruction files under a path, estimate token counts, run one real compression on the largest file, and report projected savings. Writes nothing to disk.

## Step 1 — Parse arguments

The user invoked this command with: $ARGUMENTS

Tokenize `$ARGUMENTS` on whitespace. Let the token list be `{TOKENS}`.

If any token in `{TOKENS}` is exactly equal to the string `--include-all` (no prefix or suffix variation), set `{include_all}` to `true` and remove that token from the list; otherwise `{include_all}` is `false`. Substring matches like `--include-all-the-things` or `--include-all=true` do NOT count and are passed through to the path-parsing step (where they'll fail as bad paths).

Parse `{path}` from the remaining tokens. If none remain, default to `.`. If more than one remains, print `Usage: /terse-md:analyze [<path>] [--include-all]` and STOP.

Resolve `{path}` to an absolute path via Bash, passing the path through an environment variable to avoid any shell metacharacter injection:

```bash
P="<path>" realpath -- "$P"
```

Store the resolved path as `{SCAN_ROOT}`.

## Step 2 — Discover candidate files

Use the Glob tool to find files matching each of these patterns under `{SCAN_ROOT}`:
- `**/CLAUDE.md`
- `**/SKILL.md`
- `**/user_*.md`
- `**/feedback_*.md`
- `**/reference_*.md`
- `**/project_*.md`
- `**/MEMORY.md`

Combine all results and deduplicate by absolute path (one absolute path appears at most once in the final candidate list).

Filter out:
- Any path containing `/.git/`, `/node_modules/`, `/venv/`, `/.venv/`, `/__pycache__/`, or `/target/` as a path component.
- Any file whose name ends in `.approved.yaml`.
- Any symlink. Check via `test -L "$P"` (pass the path via env var as in Step 1); skip symlinks and do not read them.
- Any candidate whose real path is outside the scan root. Run one Bash call per candidate with paths passed via env vars:

  ```bash
  C="<candidate-absolute-path>" R="<SCAN_ROOT>" bash -c '
    REAL_C=$(realpath -- "$C") || exit 1
    R="${R%/}"
    if [ -z "$R" ]; then
      case "$REAL_C" in /*) exit 0 ;; *) exit 2 ;; esac
    else
      case "$REAL_C" in "$R"|"$R"/*) exit 0 ;; *) exit 2 ;; esac
    fi'
  ```

  Non-zero exit means the candidate's real path escapes `{SCAN_ROOT}` (typically via a symlinked directory component). Drop the candidate and continue.

This plugin does not interpret `.gitignore` globs. If you want a path excluded, don't include it under `{SCAN_ROOT}`.

### Step 2a — Default-skip filter

Unless `{include_all}` is `true`, partition the remaining files into `{CANDIDATES}` (to analyze) and `{SKIPPED_BY_DEFAULT}` (reported but not compressed). A file goes to `{SKIPPED_BY_DEFAULT}` if its basename matches either:

- Exactly `MEMORY.md` — reason: `index file; already one-liners`.
- Matches the shell glob `project_*.md` — reason: `narrative scratchpad; short half-life`.

Both checks are **case-sensitive on the basename**. `Project_foo.md`, `MEMORY.MD`, and `memory.md` are NOT default-skipped; only the exact-case forms shown above match.

All other files go to `{CANDIDATES}`.

If `{include_all}` is `true`, `{CANDIDATES}` holds every remaining file and `{SKIPPED_BY_DEFAULT}` is empty.

If both `{CANDIDATES}` and `{SKIPPED_BY_DEFAULT}` are empty, print `No candidate files under {SCAN_ROOT}.` (with the resolved path wrapped in backticks) and STOP.

If `{CANDIDATES}` is empty but `{SKIPPED_BY_DEFAULT}` is not, let `{SKIP_COUNT}` be the count of `{SKIPPED_BY_DEFAULT}` and print `All {SKIP_COUNT} file(s) under {SCAN_ROOT} are skipped by default (project_*.md or MEMORY.md). Re-run with --include-all to override.` (substituting the literal count and resolved path) and STOP.

## Step 3 — Size and count caps

Count the files in `{CANDIDATES}`. If the count exceeds **250**, print a message with the real count and the resolved scan root — for example: `Refusing to scan 312 candidate files under /home/me/project — exceeds the 250-file cap. Narrow the scan path and retry.` Then STOP.

For each file in `{CANDIDATES}`, measure byte count via Bash, passing the path through an env var:

```bash
P="<absolute-path>" wc -c -- "$P"
```

If any single file exceeds **1,048,576 bytes (1 MiB)**, exclude it from `{CANDIDATES}` and note it in the final report as `(skipped: {N} bytes exceeds 1 MiB cap)`.

Compute estimated tokens per remaining file: `round(bytes / 3.5)`, rounded to the nearest 100. Record `{path, bytes, est_tokens}` for every file.

Sum all estimated tokens across all remaining files in `{CANDIDATES}` as `{TOTAL_TOKENS}`. (Files in `{SKIPPED_BY_DEFAULT}` are not measured or summed.)

## Step 4 — Identify the largest file

Sort the remaining files in `{CANDIDATES}` by `bytes` descending. The first entry is `{SAMPLE_FILE}`. If `{CANDIDATES}` is empty after the 1 MiB cap filter in Step 3, skip Steps 5–10 and go straight to Step 11 (report whichever blocks have content — skipped-by-default and/or skipped-by-1-MiB — without a sample-compression line). If both `{CANDIDATES}` and `{SKIPPED_BY_DEFAULT}` are empty after Step 3 but the 1-MiB-skipped list is non-empty, still proceed to Step 11 to surface the 1-MiB-skipped block.

## Step 5 — Create an isolated work directory

Create a private temp directory in a single Bash call:

```bash
mktemp -d /tmp/terse-md-analyze.XXXXXXXX
```

Capture the resulting absolute path as `{WORK_DIR}`. Assert it exists and is writable:

```bash
D="<WORK_DIR>" bash -c '[ -d "$D" ] && [ -w "$D" ]'
```

If the assertion fails, print `Error: failed to create a usable work directory under /tmp.` and STOP.

Use this literal value in every subsequent temp path — do not re-call `mktemp` later in this invocation.

## Step 6 — Generate a per-invocation nonce

Generate a random 16-character alphanumeric nonce via one Bash call:

```bash
head -c 32 /dev/urandom | base64 | tr -dc 'a-zA-Z0-9' | head -c 16
```

Store the result as `{NONCE}` (a concrete 16-character string). Use this literal value when substituting the `{NONCE}` placeholder in the compress prompt and in the BEGIN/END markers around the sample source.

## Step 7 — Sample compression

Read `${CLAUDE_PLUGIN_ROOT}/prompts/compress.md` and `${CLAUDE_PLUGIN_ROOT}/schema.yaml`. Substitute the verbatim contents of `schema.yaml` in place of the literal string `{SCHEMA}` in the compress prompt. Substitute the generated `{NONCE}` in place of every occurrence of the literal string `{NONCE}` in the compress prompt.

Read `{SAMPLE_FILE}` via the Read tool. Store its contents as `{SAMPLE_SOURCE}`.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted compress prompt, followed by a blank line, then the literal line `---BEGIN INPUT {NONCE}---`, then a newline, then `{SAMPLE_SOURCE}` verbatim, then a newline, then the literal line `---END INPUT {NONCE}---`.

Capture the subagent's output as `{SAMPLE_YAML}`.

Note: this step skips the normalize pass intentionally — analyze is a quick peek, not a full pipeline run.

## Step 8 — Write sample to work dir and validate

Write `{SAMPLE_YAML}` to `{WORK_DIR}/sample-out.yaml` using the **Write tool** (not a bash heredoc — heredocs can mangle content containing backticks or shell metacharacters).

Attempt schema validation via Bash. Pass both paths as positional arguments so they are never interpolated into the Python source string:

```bash
python3 -c '
import sys, yaml, jsonschema
doc = yaml.safe_load(open(sys.argv[1]))
schema = yaml.safe_load(open(sys.argv[2]))
jsonschema.validate(doc, schema)
print("ok")
' "<WORK_DIR>/sample-out.yaml" "${CLAUDE_PLUGIN_ROOT}/schema.yaml"
```

If `python3`, `yaml`, or `jsonschema` are unavailable (ImportError or command-not-found), fall back to the same structural closure check used by `run.md`: read `${CLAUDE_PLUGIN_ROOT}/schema.yaml` and confirm (a) every top-level key in `{SAMPLE_YAML}` is one of `meta`, `rules`, `transforms`, `triggers`, `thresholds`, and (b) every key inside those sections exists in the schema's `properties` definition for that section.

If validation fails, note the failure in the report (step 11) as `(validation failed: <error>)` beside the sample file's row. Continue to step 9 using the raw byte size of the work file as the output measurement.

## Step 9 — Measure sample output

```bash
wc -c -- "<WORK_DIR>/sample-out.yaml"
```

Compute `{SAMPLE_OUT_TOKENS}` = `round(output_bytes / 3.5)`, rounded to the nearest 100.

Compute `{SAMPLE_IN_TOKENS}` = the estimated token count for `{SAMPLE_FILE}` from Step 3.

## Step 10 — Extrapolate savings

```
savings_ratio = (SAMPLE_IN_TOKENS - SAMPLE_OUT_TOKENS) / SAMPLE_IN_TOKENS
estimated_total_after = round(TOTAL_TOKENS * (1 - savings_ratio))
savings_pct = round(savings_ratio * 100)
```

If `savings_ratio` is negative (output is larger than input — can happen on very short files), report `0%` savings and a note that the sample file may be too short to compress efficiently.

## Step 11 — Report

Print exactly this format, substituting real values:

```
Scanned N files, X,XXX tokens total.
Sample compression on <filename>: A,AAA → B,BBB tokens (P% saved).
Estimated total after compression: ~C,CCC tokens.

Top candidates by size:
  1. path/to/file        NNN tokens
  2. path/to/file        NNN tokens
  3. path/to/file        NNN tokens

Next: /terse-md:run --all <SCAN_ROOT>  or  /terse-md:run <file>
```

- Show up to 5 top candidates (by estimated tokens, descending). If fewer than 5 files total, show all.
- Use comma-separated thousands for all token counts.
- `<filename>` in the sample line is the basename only; the top-candidates list uses paths relative to `{SCAN_ROOT}`.
- If validation failed in Step 8, append `(validation failed)` after the sample compression line.
- If any files were skipped for exceeding the 1 MiB cap, append a `Skipped (>1 MiB):` block listing each.
- If `{SKIPPED_BY_DEFAULT}` is non-empty, append a block titled `Skipped by default (re-run with --include-all to include):` listing each path relative to `{SCAN_ROOT}` followed by its one-line reason:
  - `MEMORY.md` files → `index file; already one-liners`
  - `project_*.md` files → `narrative scratchpad; short half-life`

  Before printing any path, **sanitize the displayed string**: replace any byte less than `0x20` or equal to `0x7f` with the literal character `?`. This prevents a filename containing control characters (e.g. `\r`, `\n`, ANSI escape sequences) from rewriting earlier lines or forging fake entries in the rendered report. Sanitization is display-only; the actual path used internally is unchanged.

  Indent each entry by 2 spaces. Left-align the sanitized paths in a single column; pad each with spaces so the reason column begins 2 spaces past the longest path in this block.

  **Cap:** if `{SKIPPED_BY_DEFAULT}` contains more than **10** files, print only the first 10 (sorted alphabetically) and append a final summary line in the same indent: `(... and {EXTRA} more — {project_count} project_*.md, {memory_count} MEMORY.md)` where `{EXTRA}` = total minus 10, and the per-type counts are over ALL skipped files (not just the omitted tail). This keeps the report readable for auto-memory directories that may default-skip 100+ project files.

  Example (small case, no cap):
  ```
  Skipped by default (re-run with --include-all to include):
    MEMORY.md                index file; already one-liners
    project_foo_20260419.md  narrative scratchpad; short half-life
  ```

  Example (large case, capped):
  ```
  Skipped by default (re-run with --include-all to include):
    MEMORY.md                       index file; already one-liners
    project_aaa.md                  narrative scratchpad; short half-life
    project_bbb.md                  narrative scratchpad; short half-life
    ... (8 more shown) ...
    (... and 184 more — 193 project_*.md, 1 MEMORY.md)
  ```
- If `{CANDIDATES}` was empty (Step 4 skipped the sample), omit the `Sample compression` and `Estimated total` lines and the `Top candidates by size:` block; still print the `Scanned N files, X,XXX tokens total.` header (with `N=0, X=0`), then whichever of the `Skipped by default` and `Skipped (>1 MiB):` blocks have content. At least one will, given Step 4 only routes here when something was skipped somewhere.

## Step 12 — Cleanup

```bash
rm -rf -- "<WORK_DIR>"
```

## Constraints

- Never write any file outside `{WORK_DIR}`. The work directory is removed in Step 12.
- Never modify any discovered file.
- Never follow a symlink; skip it in Step 2.
- The sample compression skips the normalize step — this is intentional for speed.
- Do not retry the compress subagent on failure; report the raw size and continue.
- Token counts are estimates (bytes ÷ 3.5). State them as estimates if the user asks.
