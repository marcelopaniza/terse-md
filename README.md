# Terse-MD

Compress Claude instruction files. Verified lossless by meaning, not by wording.

## What it does

Terse-MD is a Claude Code plugin that compresses human-written Markdown
instruction files (`CLAUDE.md`, memory files, `SKILL.md`) into dense YAML
conforming to a closed schema. The YAML is what Claude loads on subsequent
sessions. Compression is verified by round-tripping the YAML back to prose
and asking you to approve the reconstruction. Wording will shift. Meaning
should not — but see **[The honest caveats](#the-honest-caveats)** below,
because meaning *can* drift, and the whole design assumes you'll catch it.

Compression ratios vary by content shape. Rule-heavy files (CLAUDE.md,
SKILL.md, feedback memories shaped as "rule + why + how-to-apply") can
see 50–70% reduction. Reference files with paths/facts/rationale typically
see 15–25% at best, and they lose meaningful context in the process.
Short files (<300 tokens) may not shrink at all — the YAML scaffolding
costs more than the prose you're compressing.

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

- `CLAUDE.md` — global or project instructions. High re-read frequency
  (loaded every session), rule-shaped content, long-lived. Compression
  costs amortize over hundreds of reads. **This is the target Terse-MD
  was designed for.**
- `SKILL.md` — skill definitions. Similar shape and lifetime.
- `feedback_*.md` — Claude auto-memory "feedback" type. Already shaped
  as "rule + why + how-to-apply" which maps cleanly to the schema.

The `user_*.md`, `feedback_*.md`, and `reference_*.md` patterns also match
similarly-named files outside Claude's auto-memory directory (e.g. a
`user_guide.md` or `reference_api.md` in a docs folder). If that's not
what you want, narrow the scan path — the patterns are filename prefixes,
not directory-scoped.

### Weak fits (picked up but drift is likely)

- `reference_*.md` — reference material with paths, URLs, facts, and
  rationale. The closed schema has rules/transforms/triggers/thresholds
  but no native "fact" or "context" bucket, so standalone paths and
  "why this exclusion matters" explanations tend to get dropped or
  folded into rule directives. Expect 15–25% size reduction and
  meaningful context loss. **Read the review carefully.** Probably not
  worth compressing unless the file is very large and very stable.
- `user_*.md` — a single user-profile memory. If it's short and mostly
  facts (role, preferences, expertise), you'll save <200 tokens and the
  YAML overhead eats most of that. Compress only if the file is large
  and rule-heavy.

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

## The honest caveats

Read this before you run `/terse-md:run` on anything you care about.

### 1. The approval step is the entire safety model. Don't click through.

Terse-MD's promise of "lossless by meaning" is **not** a mechanical
guarantee — it's a human-approved guarantee. The tool compresses, then
decompresses, then shows you the original next to the reconstruction and
asks if meaning survived. If you click **Approve** without reading the
diff, you have bypassed the only thing that keeps drift out of your
compressed file.

This matters because **Claude will follow the drifted version on every
future session.** If the reconstruction dropped a rationale, a path, a
"why we do X", that context is now gone from what Claude sees. You may
not notice until Claude starts making choices you thought it wouldn't.

If you're tempted to approve reflexively because the diff is long: that's
the exact moment the tool is asking you to do its most important job.
Reject and edit the source to fit the schema better, or don't compress
that file.

### 2. Compression ratios vary more than "50–70%" suggests.

That number applies to rule-heavy CLAUDE.md-shaped files. Real ranges
I've seen in testing:

| Content shape                               | Typical savings |
|---------------------------------------------|-----------------|
| CLAUDE.md with many behavioural rules       | 50–70%          |
| feedback memory (rule + why + how-to-apply) | 40–60%          |
| Short reference memory (<500 tokens)        | 15–25%          |
| Dense reference with inline commands        | 20–30% with meaningful context loss |
| Very short files (<200 tokens)              | near zero or negative — YAML overhead |

### 3. The compression pipeline is not free.

Each file burns ~60k tokens of subagent context (3 Sonnet calls: normalize,
compress, decompress). A file has to be re-read many times to amortize that
cost. CLAUDE.md easily does (read every session × hundreds of sessions).
Individual feedback memories probably don't. Run `/terse-md:analyze` first
and think about re-read frequency before compressing a whole directory.

### 4. The closed schema is opinionated.

The schema has `rules`, `transforms`, `triggers`, `thresholds`, and `meta`
— five sections chosen for *behavioural instructions*. Reference material
(facts, paths, URLs, "here's the state of the world") fits awkwardly.
When in doubt, either: (a) rewrite the source into imperative rule shape
before compressing, or (b) don't compress that file.

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
