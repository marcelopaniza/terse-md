---
description: Idempotence check. Runs the pipeline twice, diffs canonical YAML, fails if they differ.
argument-hint: <file>
---

Run the normalize + compress pipeline on a single file twice in sequence, canonicalize both outputs, and diff them. PASS if identical; FAIL with the diff if they differ.

## Step 1 — Parse arguments

The user invoked this command with: $ARGUMENTS

Parse `{file}` from `$ARGUMENTS`. `{file}` is required. If `$ARGUMENTS` is empty, print `Usage: /terse-md:test <file>` and STOP.

Resolve to an absolute path, passing the path through an environment variable:

```bash
P="<file>" realpath -- "$P"
```

Store as `{SOURCE_PATH}`.

Confirm `{SOURCE_PATH}` is a regular file and not a symlink:

```bash
P="<SOURCE_PATH>" bash -c 'test -f "$P" && test ! -L "$P" && echo regular || echo bad'
```

If the result is `bad`, print `Error: <SOURCE_PATH> is not a regular file (or is a symlink).` and STOP.

## Step 2 — Read source, hash, create work dir, generate nonces

Read `{SOURCE_PATH}` via the Read tool. Store as `{SOURCE}`.

Run one Bash call to hash the file:

```bash
P="<SOURCE_PATH>" sha256sum -- "$P" | head -c 64
```

Capture as `{SOURCE_HASH}`.

Create a private work directory for this invocation:

```bash
mktemp -d /tmp/terse-md-test.XXXXXXXX
```

Store the resulting path as `{WORK_DIR}`. Assert it exists and is writable:

```bash
D="<WORK_DIR>" bash -c '[ -d "$D" ] && [ -w "$D" ]'
```

If the assertion fails, print `Error: failed to create a usable work directory under /tmp.` and STOP.

Confirm `python3` + `PyYAML` are available before starting the pipeline, since Step 3c, 4c, and Step 5 all require them:

```bash
python3 -c 'import yaml, json, sys; print("ok")'
```

If this fails with `ModuleNotFoundError` or `command not found`, print `Error: /terse-md:test requires python3 + PyYAML.` and STOP. Leave `{WORK_DIR}` in place for the user to inspect.

Generate four independent 16-character alphanumeric nonces — one per subagent invocation — each via its own Bash call:

```bash
head -c 32 /dev/urandom | base64 | tr -dc 'a-zA-Z0-9' | head -c 16
```

Store them as `{NONCE_NORM_A}`, `{NONCE_COMP_A}`, `{NONCE_NORM_B}`, `{NONCE_COMP_B}`. Every subagent gets a fresh nonce — do not reuse across runs A and B.

## Step 3 — First pipeline run (A)

### 3a — Normalize A

Read `${CLAUDE_PLUGIN_ROOT}/prompts/normalize.md`. Substitute `{NONCE_NORM_A}` for every occurrence of `{NONCE}`.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted normalize prompt, followed by a blank line, then the literal line `---BEGIN SOURCE {NONCE_NORM_A}---`, then a newline, then `{SOURCE}` verbatim, then a newline, then the literal line `---END SOURCE {NONCE_NORM_A}---`.

Capture output as `{NORMALIZED_A}`.

### 3b — Compress A

Read `${CLAUDE_PLUGIN_ROOT}/prompts/compress.md` and `${CLAUDE_PLUGIN_ROOT}/schema.yaml`. Substitute the verbatim `schema.yaml` in place of `{SCHEMA}`, and `{NONCE_COMP_A}` for every occurrence of `{NONCE}`.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted compress prompt, followed by a blank line, then the literal line `---BEGIN INPUT {NONCE_COMP_A}---`, then a newline, then `{NORMALIZED_A}` verbatim, then a newline, then the literal line `---END INPUT {NONCE_COMP_A}---`.

Capture output as `{COMPRESSED_A}`.

### 3c — Inject `meta.source_hash` into A

Write `{COMPRESSED_A}` to `{WORK_DIR}/yaml-a.yaml` via the Write tool. Then run Bash to parse, inject, and re-serialize safely:

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
' "<WORK_DIR>/yaml-a.yaml" "<SOURCE_HASH>"
```

## Step 4 — Second pipeline run (B)

Wait for Step 3 to fully complete before starting this step. Use fresh subagents with no shared context from Step 3.

### 4a — Normalize B

Read `${CLAUDE_PLUGIN_ROOT}/prompts/normalize.md`. Substitute `{NONCE_NORM_B}` for every occurrence of `{NONCE}`.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted normalize prompt, followed by a blank line, then the literal line `---BEGIN SOURCE {NONCE_NORM_B}---`, then a newline, then `{SOURCE}` verbatim, then a newline, then the literal line `---END SOURCE {NONCE_NORM_B}---`.

Capture output as `{NORMALIZED_B}`.

### 4b — Compress B

Read `${CLAUDE_PLUGIN_ROOT}/prompts/compress.md` and `${CLAUDE_PLUGIN_ROOT}/schema.yaml`. Substitute `schema.yaml` for `{SCHEMA}` and `{NONCE_COMP_B}` for every occurrence of `{NONCE}`.

Invoke the Agent tool with:
- `subagent_type: general-purpose`
- `model: sonnet`
- `run_in_background: false`
- `prompt`: the substituted compress prompt, followed by a blank line, then the literal line `---BEGIN INPUT {NONCE_COMP_B}---`, then a newline, then `{NORMALIZED_B}` verbatim, then a newline, then the literal line `---END INPUT {NONCE_COMP_B}---`.

Capture output as `{COMPRESSED_B}`.

### 4c — Inject `meta.source_hash` into B

Write `{COMPRESSED_B}` to `{WORK_DIR}/yaml-b.yaml` via the Write tool. Run the same parse-and-inject Bash as Step 3c, passing `<WORK_DIR>/yaml-b.yaml` as the first argument.

## Step 5 — Canonicalize both outputs

```bash
python3 -c '
import sys, yaml, json
doc = yaml.safe_load(open(sys.argv[1]))
print(json.dumps(doc, sort_keys=True, indent=2))
' "<WORK_DIR>/yaml-a.yaml" > "<WORK_DIR>/test-a.json"

python3 -c '
import sys, yaml, json
doc = yaml.safe_load(open(sys.argv[1]))
print(json.dumps(doc, sort_keys=True, indent=2))
' "<WORK_DIR>/yaml-b.yaml" > "<WORK_DIR>/test-b.json"
```

If `python3` or `PyYAML` are unavailable, print `Error: python3 + PyYAML required for canonicalization.` and STOP — leave `{WORK_DIR}` in place.

## Step 6 — Diff and report

```bash
diff -u -- "<WORK_DIR>/test-a.json" "<WORK_DIR>/test-b.json"
```

Capture the diff output and exit code.

**If diff exit code is 0 (identical):**

Print:

```
PASS: pipeline is idempotent for <SOURCE_PATH>.
```

Remove the work directory:

```bash
rm -rf -- "<WORK_DIR>"
```

**If diff exit code is non-zero (differ):**

Print the full diff output, then:

```
FAIL: pipeline drifted. See diff above.
Temp files preserved for inspection:
  <WORK_DIR>/test-a.json
  <WORK_DIR>/test-b.json
```

Do NOT delete `{WORK_DIR}` on failure.

## Constraints

- `{file}` is required. Do not default to any path if the argument is missing.
- The two pipeline runs must be sequential, not parallel — the second subagent must not receive any context from the first.
- Never write outside `{WORK_DIR}`. All output goes into the `mktemp -d` directory.
- On PASS, delete the work directory. On FAIL, leave it in place and print its path.
- Do not retry either pipeline run. If a subagent returns empty output or malformed YAML, treat canonicalization failure as a FAIL, print the error, and leave the work dir in place.
- This command never writes an `.approved.yaml` file.
