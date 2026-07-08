# 0004 example — what a conforming app answers

Real wire payloads from the reference generator's `todo` app
(see [0004 — Domain Console Contract](../../proposals/0004-domain-console-contract.md)):

- [`catalog.json`](catalog.json) — the answer to `GET /_dev/catalog`, generated from the
  todo ESDM model by the `symfony-patchlevel-postgres` target. Note the column names are
  *this app's* row keys (snake_case, because that is what its list queries return).
- [`events.json`](events.json) — a `GET /_dev/events` slice, newest first: one task added,
  then renamed. On a store without per-aggregate sequences, `playhead` would be `null`.

A console needs nothing else: the catalog says which commands/queries exist and how to render
their forms and tables; the event stream shows the log those commands append to.
