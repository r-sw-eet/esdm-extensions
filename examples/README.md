# Examples

Real documents from the reference generator's example applications — each is consumed by
the generator to produce a running event-sourced app.

## [`0001-state-machine/`](0001-state-machine/)

[`task.statemachine.yaml`](0001-state-machine/task.statemachine.yaml) — the smallest useful
lifecycle. A todo task is `open` until `deleted` (a final state); every command admits only
from `open`, so nothing can mutate a deleted task. The generator turns this into
decide-step guards that reject illegal transitions with **409 Conflict**.

## [`0002-feel/`](0002-feel/)

[`quote.statemachine.yaml`](0002-feel/quote.statemachine.yaml) — 0001 and 0002 composed. A
quote walks `drafted → sent → accepted | rejected`; `accept-quote` is admissible from
`sent` **and** only while `validUntil >= today()`. The `when:` string is a FEEL expression,
validated against the aggregate's declared fields before codegen and compiled into the
same generated guard.

## [`0003-bpmn-to-esdm/`](0003-bpmn-to-esdm/)

An input/output pair for the mapper:

- [`order.bpmn`](0003-bpmn-to-esdm/order.bpmn) — the human source of truth: place → pay →
  ship with a cancel path, a `paidAmount >= total` condition on the pay→ship flow, and
  `esdm:` hints carrying what BPMN cannot express (resulting state names, field types).
- [`emitted/`](0003-bpmn-to-esdm/emitted/) — what the mapper emits from it:
  [`orders.esdm.yaml`](0003-bpmn-to-esdm/emitted/orders.esdm.yaml) (ESDM core: domain,
  bounded context, aggregate, 4 events, 4 commands, read model, 2 queries) and
  [`order.statemachine.yaml`](0003-bpmn-to-esdm/emitted/order.statemachine.yaml) (the
  graph-derived 0001 lifecycle, with the flow condition as a 0002 FEEL guard on
  `ship-order`).

The emitted YAML is machine-written (hence the flow-style vs block-style difference to the
hand-written examples) and is re-emitted on every mapper run — `order.bpmn` is the file a
human edits.

## [`0004-console-contract/`](0004-console-contract/)

What a conforming generated app answers on the wire, taken from the reference `todo` app:
[`catalog.json`](0004-console-contract/catalog.json) (the `GET /_dev/catalog`
self-description a stack-agnostic console renders itself from) and
[`events.json`](0004-console-contract/events.json) (a newest-first `GET /_dev/events`
slice). The model files stay untouched — 0004 lives entirely at the runtime HTTP level.
