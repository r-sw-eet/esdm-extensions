# ESDM Extensions

Proposals — with working examples — for making more of an [ESDM](https://www.esdm.io/)
model's *behaviour* formal, and therefore generatable, without giving up what makes ESDM
good: lightweight YAML, `esdm lint`, git-native, descriptive.

> [ESDM](https://www.esdm.io/) (Event-Sourced Domain Modeling) is created by
> [the native web](https://www.thenativeweb.io/). This repository is independent and
> unaffiliated: the proposals are written in ESDM's idiom, but nothing here is a statement
> of shipping ESDM behaviour.

## The gap

ESDM expresses domain *structure* formally — aggregates, commands, events and read models
are schema-checked YAML. *Behaviour*, however, lives in prose fields (`invariants[].rule`,
`endsWhen[].condition`): a human can read them, but a generator cannot enforce them.

| Concern                                                           | Formal in ESDM today? | Covered by                                            |
|-------------------------------------------------------------------|-----------------------|-------------------------------------------------------|
| domain structure (aggregates, events, commands, read models)      | **yes**               | core ESDM                                             |
| value-shape rules (range, enum, format, required)                 | **yes**               | JSON Schema in `state`/`data`                         |
| long-running flow, timers, correlation                            | **yes**               | core `process-manager`                                |
| **lifecycle transitions** (which command is valid in which state) | no — prose            | **[0001](proposals/0001-aggregate-state-machine.md)** |
| **data & temporal predicates** (guards, conditions, decisions)    | no — prose            | **[0002](proposals/0002-feel-rule-expressions.md)**   |
| **reaction payloads** (what an emitted command carries)           | no — *no field at all* | **[0005](proposals/0005-reaction-payload-mapping.md)** |
| bespoke logic (a pricing algorithm, a fraud check)                | no                    | out of scope — stays code                             |

Together, 0001 and 0002 convert most of that prose-behaviour gap into formal, generatable
model — the regular-shaped ~80%. The genuinely bespoke rest stays hand-written code, by
design.

## The proposals

| #                                                 | Title                   | Shape                                               | Status                                                  |
|---------------------------------------------------|-------------------------|-----------------------------------------------------|---------------------------------------------------------|
| [0001](proposals/0001-aggregate-state-machine.md) | Aggregate State Machine | new ESDM *extension kind* (`*.statemachine.yaml`)   | draft — implemented (subset) in the reference generator |
| [0002](proposals/0002-feel-rule-expressions.md)   | FEEL Rule Expressions   | *convention* over existing string fields            | draft — implemented (subset); arithmetic amended in, not yet built |
| [0003](proposals/0003-bpmn-to-esdm-mapper.md)     | BPMN → ESDM Mapper      | *top-down transformation*, emits core + 0001 + 0002 | draft — implemented (subset)                            |
| [0004](proposals/0004-domain-console-contract.md) | Domain Console Contract | *runtime HTTP contract* exposed by generated apps   | draft — implemented                                     |
| [0005](proposals/0005-reaction-payload-mapping.md) | Reaction Payload Mapping | *convention* over an annotation, reusing 0002's FEEL | draft — implemented in all four generators, 8/8 targets conform |

Three different layers live here:

- **0001, 0002 and 0005 extend the ESDM *model*** — they add (or formalize) what the intermediate
  representation can express: lifecycles, conditions, and what a reaction carries.
- **0003 targets ESDM from above** — it lets non-programmer domain experts author in BPMN
  and *emits* ESDM. It adds nothing to ESDM itself.
- **0004 sits after code generation** — a small dev-time HTTP surface every generated app
  exposes (model catalog, authoring BPMN, raw event stream), so **one** stack-agnostic
  console/viewer can drive apps from *any* generator target. It, too, adds nothing to ESDM.

```
BPMN / DMN  ──0003──▶  ESDM (core + 0001 + 0002)  ──generator──▶  app  ──0004──▶  console
(authoring)            (intermediate representation)            (runtime)      (one UI, all targets)
```

## How 0001 and 0002 compose

A single real guard usually needs both — *"a quote can be accepted only from `sent`*
**(0001 — a lifecycle transition)** *and while `validUntil >= today()`"* **(0002 — a FEEL
predicate)**:

```yaml
admits:
  - { command: accept-quote, from: [sent], when: 'validUntil >= today()' }
```

## Design stance

- **Borrow, don't invent.** 0001 takes its semantics from UML statecharts and the decider
  pattern; 0002 adopts a subset of DMN's FEEL; 0003 maps from OMG BPMN. No new language.
- **Stay lightweight.** Everything is ESDM's own YAML idiom (or standard `.bpmn` XML) —
  hand-writable, diff-able, git-friendly.
- **Keep `esdm lint` green.** 0001 is a separate extension kind (core never validates
  against extensions); 0002 is a pure convention over existing `type: string` fields.
  Nothing changes the core schema.
- **Validation stays a gate.** Everything formal added here is checked *before* codegen —
  0002 pairs `esdm lint` (structure) with a model-aware FEEL validator (logic).

## Examples

Working documents, lifted from the reference generator's example apps — see
[examples/](examples/):

- [`examples/0001-state-machine/`](examples/0001-state-machine/) — the smallest useful
  lifecycle: a todo task that admits nothing once deleted.
- [`examples/0002-feel/`](examples/0002-feel/) — a quote lifecycle whose `accept-quote`
  admit carries a temporal FEEL guard.
- [`examples/0003-bpmn-to-esdm/`](examples/0003-bpmn-to-esdm/) — a full input/output pair:
  `order.bpmn` in, ESDM core + state machine (including a FEEL guard) out. This one
  exercises all three proposals at once.
- [`examples/0004-console-contract/`](examples/0004-console-contract/) — what a conforming
  app answers on the wire: a real `catalog.json` and an `events.json` slice.

## Reference implementation

The proposals are implemented (to the subset noted per proposal) by the **ESDM toolchain**,
a family of standalone code generators: draw the business process in BPMN, the 0003 mapper
compiles it to ESDM and a generator emits a complete, running event-sourced application.
Which languages and frameworks are targeted is the generators' concern, not this spec's:
everything here is target-agnostic.

The generators: [esdm-2-symfony](https://github.com/r-sw-eet/esdm-2-symfony) (PHP · Symfony),
[esdm-2-nimbus](https://github.com/r-sw-eet/esdm-2-nimbus) (TypeScript · Nimbus),
[esdm-2-python](https://github.com/r-sw-eet/esdm-2-python) (Python · Django) and
[esdm-2-opencqrs](https://github.com/r-sw-eet/esdm-2-opencqrs) (Java · Spring Boot + OpenCQRS),
each with a PostgreSQL or EventSourcingDB + MongoDB event-store target - four generators, eight
targets, one conformance suite.

## Status & discussion

All proposals are drafts, not upstream ESDM. Issues and discussion are welcome — the
point of publishing is to find out whether any of this is worth carrying upstream.

New proposals: number sequentially (`NNNN-kebab-title.md`), follow the existing structure
(Status, Motivation, Specification, Backwards compatibility, Out of scope, Prior art,
References), and add a row to the table above.

## License

[MIT](LICENSE) © 2026 Ralf Süss
