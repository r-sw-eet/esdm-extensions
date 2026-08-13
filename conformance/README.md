# C4 — cross-generator behavioral conformance (data + procedure)

This directory contains **no code**. It holds the language-agnostic conformance
artifacts, and this README specifies the procedure a runner must implement. Each
generator repo carries a **native runner** (its own language) that finds this
directory in the sibling checkout, builds and boots **its own** targets, executes
the scenario, and compares its normalized observations against the recorded golden
answers. Cross-stack conformance holds transitively: every stack that matches the
golden file matches every other.

```
conformance/
  scenarios/<app>.yaml            # what to fire (steps + checkpoints + applicable targets)
  registry.yaml                   # accepted divergences: (endpoint, field, reason[, targets])
  golden/<app>.observations.json  # the normative answers (recorded by the oracle runner)
```

**A scenario belongs here only if its `model:` resolves inside a public repo of the
family.** A scenario is a domain surface written out in full - endpoints, payloads and
recorded responses - so one pointing at a private model publishes that domain while
staying unrunnable for everyone else. Keep private models and their scenarios in their
own repo.

Runners: `esdm-2-nimbus/scripts/conformance.ts` · `esdm-2-symfony/scripts/conformance.php`
(the oracle — carries `--record`) · `esdm-2-python/scripts/conformance-c4.py` ·
`esdm-2-opencqrs/scripts/conformance.sh`.

## Runner contract

1. **Applicability** — run each of your repo's targets that appears in the
   scenario's `targets:` list; skip the rest silently.
2. **Generate** from the canonical model (`model:` is workspace-relative; pass it
   explicitly — never use your repo's own example copy, they drift).
3. **Boot** the emitted compose with only the `api` service published, on a
   host port that cannot collide with dev stacks or other runners
   (nimbus 1811x, symfony 1812x, python 1813x). Ready when
   `GET /_dev/catalog` returns 200 (timeout 300 s). Tear down (`down -v`) after.
4. **Steps**, in order:
   - POST step: resolve `$NAME` placeholders from captures, send JSON, record
     `{step, endpoint: "POST <path>", status, body}`. If the step has
     `capture: NAME` and the response carries a string `id`, store it.
   - GET step: with `poll: true`, re-read every 1 s until the array has
     `min_rows` rows (default 1) or `poll_timeout` (default 45 s) elapses. With
     `capture:`, sort rows by canonical JSON and capture `capture_field`
     (default `id`) of the first row whose value is not already captured.
5. **Converge** — poll all checkpoints until two consecutive identical reads
   (1 s interval, 90 s timeout), then record them like steps
   (`checkpoint` instead of `step`).
6. **Normalize** (exactly — golden files are byte-comparisons after this):
   - every captured value → `«NAME»` wherever it appears as a string;
   - object keys snake→camel (`_x` → `X`; keys without underscores unchanged);
   - list bodies sorted by their canonical JSON (lexicographically sorted keys);
   - the checkpoint named `events`: each row becomes
     `{aggregate: lowercased, aggregateId: masked, event: <last dot-segment,
     underscores→dashes, lowercased>, playhead: <as-is>, payload: <normalized>}`
     — the store event id and `recorded_on` are dropped; window order (newest
     first) is preserved.
7. **Compare** against `golden/<app>.observations.json`: flatten each record to
   `status` and `body...` field paths (`body.id`, `body[0].title`); the record
   key is `"<METHOD> <path>#<step-or-checkpoint-name>"`. Every differing field
   is a divergence `(endpoint, field, golden, got)`. Mask it iff a registry
   entry's `endpoint` and `field` globs match (fnmatch) and, when the entry has
   `targets:`, your target is listed. Unregistered divergences → exit 1.
8. **`--record`** (oracle runner only): instead of comparing, write your
   normalized observations to `golden/<app>.observations.json`. Golden files
   only change deliberately, with the diff reviewed like a spec change.

## Semantics notes

- Scenario steps never assert expected outcomes — rejection probes are ordinary
  steps; correctness is *agreement with golden*, not hardcoded expectations.
- Ids are server-minted and target-specific by design (ULID / UUIDv7 / UUID4);
  the capture placeholders are what make them comparable.
- Keep scenarios under 50 events total: `/_dev/events` (proposal 0004) serves a
  50-row window and is the event-sequence tap.
- The registry is for *inherent* differences only (currently one: `playhead` is
  null on EventSourcingDB-backed stores, an integer on Postgres-backed ones).
  Behavioral differences get fixed in generators, not registered — the
  2026-07-12 harmonization removed every behavioral entry.
