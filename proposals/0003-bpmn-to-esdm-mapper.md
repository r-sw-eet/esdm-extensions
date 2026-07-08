# ESDM Extension Proposal 0003 — BPMN → ESDM Mapper

- **Status:** Draft / proposed (not part of upstream ESDM) — implemented (subset), see *Implementation status*
- **Target:** a *top-down transformation* — BPMN authored by domain experts → ESDM (the
  intermediate) → code. Produces nothing new in ESDM itself; it *emits* existing ESDM.
- **Author:** Ralf Süss ([r-sw-eet](https://github.com/r-sw-eet))
- **Affects:** nothing in ESDM core. Emits **ESDM core docs** + **[0001] state-machine docs**
  + **[0002] FEEL conditions**.
- **Related:** [0001 — Aggregate State Machine](0001-aggregate-state-machine.md),
  [0002 — FEEL Rule Expressions](0002-feel-rule-expressions.md)

> This document is a proposal written in ESDM's idiom. It is **not** a statement of shipping
> ESDM behaviour.

## Summary

If the people who model the domain are **non-programmers** (business/domain experts), YAML is
the wrong authoring surface — but **BPMN** (with **DMN** for rules) is purpose-built for them
and has free, embeddable visual editors (bpmn-js / dmn-js). This proposal defines a **top-down
mapper** that ingests a domain-expert-authored **BPMN 2.0** model and **decomposes** it into
three streams of ESDM:

1. **ESDM pure (core)** — bounded-contexts, aggregates, commands, events, read-models,
   process-managers, actors, external-systems, context-mappings;
2. **[0001] state machines** — the lifecycle slice (an entity walking through its states);
3. **[0002] FEEL** — the condition slice (gateway predicates, timer durations, guards).

ESDM is the **intermediate representation** ("ESDM has to come between them"); from there the
existing generator compiles ESDM → runnable code unchanged. The principle throughout:
**author in orchestration (how humans think), compile to choreographed event sourcing (what
the system wants).**

## Motivation

Every prior analysis converged on the same shape:

- ESDM-as-YAML serves *developers*, not non-programmer domain experts.
- **BPMN + DMN** serve exactly that audience (OMG's stated goal: "readable by all business
  stakeholders"), with mature open-source modelers.
- ESDM should sit **between** the visual model and the code — as a canonical, lint-checked
  IR.
- The one place that pipeline *leaks* is **decision logic** — and [0002] (FEEL) plugs it.

What's missing is the transformation itself: *given a BPMN diagram, which ESDM artifacts does
each element become?* That decomposition is this RFC.

## Pipeline

```
domain expert
   │  draws in a visual editor (bpmn-js / dmn-js / Camunda Modeler)
   ▼
BPMN 2.0 XML  +  DMN XML            ← the human source of truth
   │  0003 mapper (this proposal)
   ▼
ESDM core  +  0001 state-machines  +  0002 FEEL     ← generated IR
   │  esdm lint (structure)  +  FEEL validate (logic)     ← existing gates
   ▼
runnable event-sourced code         ← existing adapter
```

**Source-of-truth recommendation:** for a non-programmer audience, **BPMN/DMN is the human
source of truth**; ESDM is a *generated* IR (re-emitted on each change), and code is generated
from ESDM. Reverse rendering (ESDM → a BPMN diagram for viewing) is possible but **lossy on
layout** — regenerate a diagram, don't try to preserve the drawing.

## Authoring surface — reuse bpmn.io, don't build an editor

The visual editor is **not** something this project builds. The **bpmn.io** toolkit (Camunda,
open-source, MIT) already provides production-grade, embeddable modelers — the same ones the
Camunda Modeler desktop app is built from:

- **`bpmn-js`** — a complete BPMN 2.0 modeler as a JS component: palette, drag-drop, in-canvas
  validation, import/export of standard BPMN 2.0 XML.
- **`dmn-js`** — a DMN editor for decision tables **with a built-in FEEL expression editor**.
- **`bpmn-js-properties-panel` + `@bpmn-io/properties-panel`** — a side panel for attaching
  named properties to any diagram element (our hook for mapper hints; see below).

All three are MIT-licensed and embed as drop-in components in any web application — no new
application to build, no bespoke diagramming code.

### Why this is structural, not just convenient: the FEEL alignment

dmn-js's expression language **is FEEL**, and [0002] already ships a FEEL subset with a parser,
compiler and validator. So the box a domain expert types a condition into and the guard that
later enforces it **speak one language**: what they author in dmn-js is — modulo subset —
exactly what the [0002] parser reads and the [0002] compiler turns into a generated guard.
No translation layer, no second expression dialect to invent or reconcile.

> This is the single strongest argument for adopting BPMN/DMN over inventing a bespoke visual
> notation: **the rule half is already built and already wired into codegen.** 0003 only has to
> map *structure*; the *logic* an expert writes flows through the [0002] pipeline untouched.

### Storage & round-trip

bpmn-js/dmn-js import and export plain **`.bpmn` / `.dmn` XML** — text, diff-able, git-friendly.
They sit beside the model as the human source of truth:

```
apps/<app>/
  authoring/    # .bpmn / .dmn — drawn in the embedded modeler (human source of truth)
  model/        # ESDM IR — emitted by the 0003 mapper, then lint + FEEL gated
  generated/    # code — generated from model/ (disposable)
```

Flow: draw in the modeler → save XML to `authoring/` → 0003 mapper emits ESDM into `model/`
→ the existing generator runs. bpmn.io owns the *drawing*; ESDM owns the *meaning*. (ESDM → a
fresh diagram is possible but lossy on layout — regenerate, don't preserve the picture.)

### Cutting mapper guesswork with the properties panel

The mapper's hardest calls — *is this XOR gateway on entity **state** (→ [0001]) or on **data**
(→ [0002])? which aggregate does this lane act on?* — don't have to stay heuristic. A small set
of custom properties in the panel lets the author tag elements explicitly:

| Property                                      | Meaning                                           |
|-----------------------------------------------|---------------------------------------------------|
| `esdm:context`                                | the bounded-context a pool/lane belongs to        |
| `esdm:aggregate`                              | the aggregate a task/lane mutates                 |
| `esdm:kind = state \| data`                   | force a gateway to the [0001] or [0002] slice     |
| `esdm:lifecycle = create \| mutate \| delete` | the command's lifecycle (else the verb heuristic) |

When a tag is present it **overrides** the detection heuristics in *The decomposition*; when
absent, the heuristics still run. Simple diagrams stay zero-config; precise control is available
exactly where ambiguity bites. (This mirrors how the existing generator already accepts an
`esdm-extensions.io/lifecycle` annotation to override its verb heuristic.)

## The decomposition

Each BPMN construct routes to one of the three streams. The spine of the mapper:

### Participants & data → ESDM pure

| BPMN                              | ESDM core                                                                    |
|-----------------------------------|------------------------------------------------------------------------------|
| Pool (participant / organisation) | `bounded-context` (or `external-system` if third-party)                      |
| Lane (role)                       | `actor`                                                                      |
| Data object / data store          | `read-model` (or `value-object` / aggregate `state`)                         |
| Message flow across pools         | `context-mapping` + the message → an `event`/`command` crossing the boundary |

### Activities → commands (ESDM pure), with rule tasks → FEEL

| BPMN                               | Routes to                                                                |
|------------------------------------|--------------------------------------------------------------------------|
| User task                          | `command` issued by the lane's `actor` (ESDM core)                       |
| Service task                       | `command` + `domain-service` (ESDM core)                                 |
| Send / Receive task                | `emits` to / `reactsTo` from an `external-system` (ESDM core)            |
| **Business-rule task** (calls DMN) | **[0002] FEEL** decision (or a DMN decision table, future 0002)          |
| Script task                        | a `command`/`domain-service`; bespoke body stays **code** (out of scope) |
| Call activity / sub-process        | a nested `process-manager` (ESDM core)                                   |

### Events → triggers, timers, terminals

| BPMN                                 | Routes to                                                                 |
|--------------------------------------|---------------------------------------------------------------------------|
| Start event                          | `process-manager.startsWhen` (ESDM core); timer-start → a scheduled start |
| End event                            | `process-manager.endsWhen` (ESDM) + the recorded domain `event`           |
| Intermediate message (catch / throw) | `reactsTo` / `emits` (ESDM); cross-boundary → `external-system`           |
| **Timer / boundary timer**           | `process-manager.timers` (ESDM core) **+ [0002]** for the `duration(...)` |
| Conditional event                    | a reaction whose trigger predicate is **[0002] FEEL**                     |
| Error / boundary error               | a failure `event` + compensating `command` (ESDM, *partial* — see gaps)   |
| Signal event                         | a broadcast `event` everyone `reactsTo` (ESDM)                            |

### Gateways → the three-way split (this is where 0001 vs 0002 vs core is decided)

| BPMN gateway                                                 | Routes to                                                                                |
|--------------------------------------------------------------|------------------------------------------------------------------------------------------|
| Exclusive (XOR) on **entity state** ("if quote is `sent` …") | **[0001]** transition / `admits`                                                         |
| Exclusive (XOR) on **data** ("if amount > 10000 …")          | **[0002]** FEEL condition on the branch; branches → alternative `command`/`event` (core) |
| Parallel (AND) split                                         | fan-out: a `policy`/`command` that `emits` several commands (ESDM `emits` is a list)     |
| Parallel (AND) join                                          | `process-manager.state` waits for both; join condition → **[0002]**                      |
| Event-based gateway ("accept OR reject OR timeout")          | `process-manager` that `reactsTo` whichever event; timeout branch → **[0002]** timer     |
| Inclusive (OR)                                               | multiple branches, each guarded by **[0002]** FEEL                                       |

### The three streams, named

- **ESDM pure (core)** — the structural + orchestration skeleton: `bounded-context`,
  `aggregate`, `command`, `event`, `read-model`, `actor`, `external-system`,
  `context-mapping`, and — for stateful/timered/correlated flow — `process-manager` (or a
  `policy` for a simple stateless reaction).
- **[0001] state machines** — emitted when the BPMN process is essentially **walking one
  entity through its states** (a status field advancing via XOR-on-state gateways). That
  lifecycle is extracted into a `state-machine` document over the aggregate.
- **[0002] FEEL** — every gateway condition, conditional/timer-event predicate and flow guard
  becomes a FEEL string in the relevant `condition`/`rule`/`firesAt` field.

## Detection heuristics (which slice a construct goes to)

The mapper needs rules to disambiguate. Proposed:

- **XOR gateway** → if its conditions test an entity's **status/state field**, emit a [0001]
  transition/`admit`; if they test **other data**, emit a [0002] FEEL guard; if both, emit
  both (0001 for the state part, 0002 for the data part — exactly how a real guard composes).
- **Sequence of status-mutating tasks on one aggregate** → collapse into a [0001]
  state-machine (the "process" *is* that aggregate's lifecycle).
- **Stateful / long-running / timered / multi-aggregate flow** → `process-manager`
  (startsWhen / reactsTo / emits / timers / endsWhen / correlatedBy / state).
- **Simple stateless "on event X do command Y"** → a `policy` (no process-manager needed).
- **Business-rule task / DMN reference** → [0002] FEEL (decision expression), with the
  DMN decision-table form noted as future 0002 work.

## What does *not* map (honest gaps)

- **Diagram geometry / layout** — x/y, colours, lanes' visual order. Dropped; ESDM stores
  meaning, not pictures. Regenerate a diagram from the model rather than preserve the drawing.
- **The flow as a single drawn gestalt** — ESDM is declarative/choreographed, so the process
  decomposes into reactions + process-manager wiring. You keep the *edges*, not the *picture*;
  round-tripping back to the exact BPMN drawing is therefore lossy by design.
- **Bespoke script-task logic** — a pricing algorithm, a fraud check. Stays hand-written code;
  the mapper emits a stub/command, not the body.
- **Full compensation / error semantics** — BPMN compensation events and complex error
  boundaries map only *partially* (as failure-event → compensating-command reactions), not as
  first-class transactional compensation.

The mapper MUST **report unmapped elements** (a coverage log), never silently drop them —
silent loss would read as "fully converted" when it wasn't.

## Validation

The mapper's output is plain ESDM (+ 0001/0002), so it flows through the **existing gates**
unchanged: `esdm lint` (structure) and the [0002] FEEL validator (logic). Two mapper-specific
checks on top:

- **Coverage** — every BPMN element is either mapped or explicitly listed as unmapped.
- **Round-trippable identity** — names/ids are stable so re-running the mapper on an edited
  diagram produces a minimal ESDM diff (not a churn of renamed artifacts).

## Scope / out of scope (v1)

- **In:** the common BPMN subset — pools/lanes, user/service/send/receive/business-rule tasks,
  start/end/message/timer/conditional events, XOR/AND/event-based gateways, call activities,
  DMN business-rule references.
- **Out:** full BPMN (multi-instance markers, ad-hoc sub-processes, BPMN *choreography*
  diagrams, complex compensation), bespoke script bodies, layout preservation, and a runtime
  BPMN engine (this compiles to ES code; it does not embed Camunda/Zeebe).

## Implementation status

The reference implementation (**BPAG**) ships this mapper as a `bpmn:map` command, exercised
by three example applications (a single-pool order flow; a two-pool commerce flow with a
cross-pool policy; a five-pool factory flow with a quality-gate rework loop and a
four-policy chain):

- **Implemented:** namespace-agnostic BPMN 2.0 parsing (diagram interchange ignored);
  pool/process → `bounded-context` + `aggregate`; task → `command` + `event`; the
  [0001](0001-aggregate-state-machine.md) state machine is **graph-derived** —
  `admits[].from` from a backward walk over sequence flows, final states from a forward
  walk; sequence-flow `conditionExpression` → [0002](0002-feel-rule-expressions.md) FEEL
  `admits[].when`; `messageFlow` across pools → a `policy` (source task's event → target
  task's command); a default tabular `read-model` per aggregate.
- **Authoring hints:** what BPMN cannot express rides on a small `esdm:` extension
  namespace — `<esdm:meta domain|context|aggregate|state|lifecycle …/>` and
  `<esdm:field name type/>`. Every decision the mapper makes that a downstream generator
  would otherwise guess (e.g. a command's lifecycle) is pinned explicitly as an annotation
  on the emitted document.
- **Authoring surface:** an embedded bpmn-js modeler (with a properties-panel group for
  FEEL rules on sequence flows); `.bpmn` files under `authoring/` are the human source of
  truth, `model/` is re-emitted on every mapper run.
- **Not yet implemented:** DMN / dmn-js decision tables, timer/conditional events →
  `process-manager` mapping, compensation/error boundaries, and the coverage report of
  unmapped elements.

See [examples/0003-bpmn-to-esdm](../examples/0003-bpmn-to-esdm/) for a real input/output
pair.

## Prior art

- **BPMN + DMN (OMG)** — the authoring standards; **Camunda / Zeebe** the reference engines
  (this proposal compiles *instead of* runtime-executing them).
- **bpmn-js / dmn-js (Camunda, open-source)** — the embeddable editors that make this a
  realistic non-programmer authoring surface.
- **Event Modeling** (Dymitruk) — the *other* business-visual technique for event sourcing:
  BPMN covers orchestration; Event Modeling covers the whole event-sourced blueprint. A
  sibling future RFC could map an Event-Modeling board to ESDM the same way this maps BPMN.
- **MDA** (OMG) — model-to-code; this whole pipeline is small-scale MDA.
- **The decider** (Chassaing) — BPMN gateways/tasks ultimately compile into decide/evolve.

## References

- BPMN specification (OMG): https://www.omg.org/spec/BPMN/
- DMN specification (OMG): https://www.omg.org/spec/DMN/
- bpmn-js: https://bpmn.io/  · dmn-js: https://bpmn.io/toolkit/dmn-js/
- Proposal 0001 — Aggregate State Machine: ./0001-aggregate-state-machine.md
- Proposal 0002 — FEEL Rule Expressions: ./0002-feel-rule-expressions.md
