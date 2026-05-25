# skill-telemetry

> I have 14+ installed skills and no signal on which ones actually fire, which produce outcomes the user accepts, and which quietly rot.

## Install

### One-liner

```sh
curl -fsSL https://raw.githubusercontent.com/j0yen/skill-telemetry/main/install.sh | bash
```

### Manual

```sh
git clone --depth 1 https://github.com/j0yen/skill-telemetry.git
cd skill-telemetry
./install.sh
```

Installs the `spool` binary via `cargo install --path . --locked`. Requires `cargo` / `rustc 1.85+` and `git`. Built binary lands in `~/.cargo/bin/`.

## Why

I have 14+ installed skills and no signal on which ones actually fire, which produce outcomes the user accepts, and which quietly rot. Spool adds a tiny on-disk telemetry layer: each invocation appends one JSONL line per start and end event to a monthly bucket. Reports answer 'which skills did I use this week,' 'which are stale,' and (after backfill) 'which got redirected.' This slice ships Phase 0 + the read commands (rank/report --stale) so /self-review can consume telemetry next session.

## Build

```sh
cargo build --release
```

Produces `target/release/spool`. Symlink into `~/.local/bin/` if you want it on `$PATH`.

## Usage

```sh
spool --help
```

## Audience

the author (and the agent) bracketing skill invocations with `spool log start <skill>` / `spool log end <id>`; the author reading `spool rank` weekly. Storage at ~/.claude/spool/<YYYY-MM>.jsonl; CLI invoked from shell or from SKILL.md instructions.

## Acceptance criteria

This project was scaffolded from a PRD via the `autobuilder` pipeline. The MUST-level acceptance criteria are:

- **AC1**: `spool log start <skill>` creates (or appends to) `<root>/<YYYY-MM>.jsonl` with one JSONL line containing `event: "start"`, `skill`, `invocation_id` (ULID), `started_at` (RFC3339). Prints just the invocation_id to stdout. `--root <dir>` ...
- **AC2**: Invocation IDs are ULIDs (26 chars, Crockford base32, lexicographically sortable). Two rapid `spool log start` invocations produce two different IDs.
- **AC3**: `spool log end <invocation_id> --outcome ok` appends a JSONL line with `event: "end"`, the same `invocation_id`, `ended_at` (RFC3339), `duration_ms` (computed from the matching start event's started_at), and `outcome` ∈ {ok, error, inter...
- **AC4**: `spool rank` reads all `<root>/<YYYY-MM>.jsonl` files, joins start+end pairs by invocation_id, emits a text table sorted by invocation count descending. Columns: skill, invocations, mean_ms, last_fired (RFC3339 or `never`).
- **AC5**: `spool rank --format json` emits a JSON array of `{skill, invocations, mean_ms_or_null, last_fired}` objects. Deterministic key ordering; same input always produces byte-identical output.
- **AC6**: `spool report --stale <duration>` (e.g. `30d`, `7d`) lists skills with zero invocations within the window. Stale skill discovery: comparing skills present in `<root>/skills.txt` (one-line-per-skill, optional) OR all skills ever seen in t...
- **AC7**: Robust to incomplete pairs: a `start` without matching `end` shows in `rank` with `mean_ms_or_null: null` and is still counted as an invocation. An `end` without start is reported on stderr as orphaned but does not crash.
- **AC8**: `spool log start <skill> --args <text>` truncates `args` to 200 chars (after which a `…` is appended). The truncation is recorded as `args_truncated: true` when applied.
- **AC9**: Exit codes: 0 = success; 1 = expected-failure (orphaned end event reported, stale skill found with --strict); 2 = invocation error (bad args, unreadable root, unwritable journal).

Each AC has a matching integration test under `tests/acceptance_ac<n>.rs`.

## Provenance

Built via the [`autobuilder`](https://github.com/j0yen/autobuilder) pipeline (PRD intake -> intent-card -> scaffold -> iterate-and-prove). Originally consolidated as a subdir of the [`wintermute`](https://github.com/j0yen/wintermute) monorepo; this standalone repo is a fresh-init snapshot for easier consumption and distribution.

## License

Licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.
