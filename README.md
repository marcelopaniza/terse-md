# Terse-MD

Compress Claude instruction files. Verified lossless by meaning, not by wording.

## What it does

Terse-MD is a Claude Code plugin that compresses human-written Markdown
instruction files (`CLAUDE.md`, memory files, `SKILL.md`) into dense YAML
conforming to a closed schema. The YAML is what Claude loads on subsequent
sessions — typically 50–70% fewer tokens than the source prose. Compression
is verified by round-tripping the YAML back to prose and asking you to
approve the reconstruction. Wording will shift. Meaning should not.

## Requirements

Terse-MD runs inside Claude Code. The validation step in `/terse-md:run` and the
canonicalization step in `/terse-md:test` shell out to Python. You need:

- `python3`
- `PyYAML` (`pip install pyyaml`)
- `jsonschema` (`pip install jsonschema`)

If `pyyaml` or `jsonschema` is missing, `/terse-md:analyze` falls back to a
structural closure check of the compressed YAML's top-level keys against the
schema. `/terse-md:run` and `/terse-md:test` both require PyYAML — `/terse-md:run`
uses it to inject `meta.source_hash` into the validated YAML, and
`/terse-md:test` uses it to canonicalize both runs before diffing. Both commands
error out cleanly at the first Python call if PyYAML is missing.

## Install

From inside Claude Code:

```
/plugin marketplace add marcelopaniza/terse-md
/plugin install terse-md@marcelopaniza-terse-md
/reload-plugins
```

Or for local development, clone the repo and start Claude Code with the plugin
directory attached:

```
git clone https://github.com/marcelopaniza/terse-md
claude --plugin-dir ./terse-md
```

## Use

```
/terse-md:analyze .
```

Scans `.` for candidate files, counts tokens (bytes ÷ 3.5 estimate), runs one
real compression on the largest file, and prints projected savings. Writes
nothing.

```
Scanned 4 files, 3,700 tokens total.
Sample compression on examples/larder_claude.source.md: 1,700 → 600 tokens (65% saved).
Estimated total after compression: ~1,300 tokens.

Top candidates by size:
  1. examples/larder_claude.source.md    1,700 tokens
  2. docs/SKILL.md                         900 tokens
  3. CLAUDE.md                             800 tokens
  4. memory/feedback_bar.md                300 tokens

Skipped by default (re-run with --include-all to include):
  MEMORY.md                  index file; already one-liners
  memory/project_foo.md      narrative scratchpad; short half-life

Next: `/terse-md:run --all .` or `/terse-md:run <file>`
```

```
/terse-md:run examples/larder_claude.source.md --dry-run
```

Runs the full pipeline on one file: normalize → compress → decompress, then
shows the normalized source alongside the prose reconstructed from the YAML.
You approve, reject, or skip. `--dry-run` performs every step but never
writes the `.approved.yaml` file.

```
/terse-md:test examples/larder_claude.source.md
```

Idempotence check. Runs the pipeline twice with fresh subagents, canonicalizes
both YAMLs (sorted keys, JSON form), and diffs them. Passes if identical.
Useful after changing prompts or schema.

## Which files to compress

Terse-MD's closed schema works best on files that are **stable** (you don't
edit them weekly) and **structured** (rules, facts, pointers — not flowing
narrative). By default, Terse-MD picks up files likely to fit that profile
and skips files that don't.

### Good fits (picked up by default)

- `CLAUDE.md` — global or project instructions; stable, high re-read frequency.
- `SKILL.md` — skill definitions; stable once written.
- `user_*.md` — Claude auto-memory "user" type; role/expertise/preferences.
- `feedback_*.md` — Claude auto-memory "feedback" type; rule + why + how-to-apply.
- `reference_*.md` — Claude auto-memory "reference" type; pointers to URLs or systems.

The `user_*.md`, `feedback_*.md`, and `reference_*.md` patterns also match
similarly-named files outside Claude's auto-memory directory (e.g. a
`user_guide.md` or `reference_api.md` in a docs folder). If that's not
what you want, narrow the scan path — the patterns are filename prefixes,
not directory-scoped.

### Skipped by default

- `MEMORY.md` — the auto-memory index. Already one-liners; nothing to compress.
- `project_*.md` — Claude auto-memory "project" type. These are narrative
  scratchpads with SHAs, version tags, TODO markers ("PICK UP HERE",
  "LANDED"), file paths, and mid-task edits. They decay fast (many are
  archived or deleted within days). Compressing then discarding is wasted
  effort, and the closed schema fights their loose narrative shape.

### Override

If you know what you're doing and want to compress everything Terse-MD
found, pass `--include-all` to either command:

```
/terse-md:analyze ~/.claude/projects/-mnt-data-game2/memory --include-all
/terse-md:run --all ~/.claude/projects/-mnt-data-game2/memory --include-all
```

Naming a single file explicitly with `/terse-md:run <path>` always bypasses
the default-skip filter — if you point Terse-MD at a specific file, it
trusts you.

## How it works

Three Sonnet subagents run in sequence. The first rewrites your prose in
canonical form, stripping filler without altering meaning. The second emits
YAML conforming to `schema.yaml` — a closed vocabulary of `rules`,
`transforms`, `triggers`, `thresholds`, and `meta`. Any key outside the
schema fails validation, and the compress step is retried once with the
error attached. The third expands the validated YAML back to prose. You
review source-vs-reconstruction side-by-side and approve only if meaning
survived. The approved YAML is written next to the source as
`<name>.approved.yaml` — that file is what Claude loads on future sessions.

## What it's not

- Not a token-optimizer that guarantees savings (it measures, then asks).
- Not lossless at the wording level (it's lossless at the meaning level — you approve the diff).
- Not affiliated with Anthropic.

## License

MIT. See [LICENSE](LICENSE).
