# ESDM Extension Proposal 0001 — Aggregate State Machine

- **Status:** Draft / proposed (not part of upstream ESDM) — implemented (subset), see *Implementation status*
- **Target:** an ESDM *extension* (api group `schema.esdm.io/state-machine/v1`), not a core change
- **Author:** Ralf Süss ([r-sw-eet](https://github.com/r-sw-eet))
- **Affects:** `aggregate` (by reference only — core schema unchanged)

> This document is a proposal written in ESDM's idiom. It is **not** a description of
> shipping ESDM behaviour. Where it makes claims about existing ESDM, those are marked.

## Summary

ESDM models an aggregate's *data* precisely (`state` is a JSON Schema) but its *lifecycle*
only in prose (`invariants[].rule` is a free-text sentence). This proposal adds a small,
declarative **state-machine** document that pins down an aggregate's lifecycle as states +
transitions, referencing the aggregate's existing commands and events by name. It is purely
descriptive — like the rest of ESDM — but, being structured, it is machine-checkable by
`esdm lint` and mechanically consumable by code generators.

## Motivation

An aggregate is, in the DDD/event-sourcing tradition, a state machine: commands are inputs,
events are transitions, and the aggregate's job is to reject illegal transitions (see *Prior
art*). ESDM already captures the *vocabulary* of that machine — the `command` and `event`
catalogue, and `command.publishes` wiring — but the *machine* itself is implicit:

- **Which command is admissible in which state** is not expressed. Today nothing in the model
  says "you cannot rename a deleted task" except an English `invariant`.
- **Which state an event moves the aggregate into** is not expressed.
- Consequently a generator (or a reader) cannot tell a legal lifecycle from an illegal one,
  and a generated aggregate has **no guards** — every command just records its event.

Concretely, in the `todo` example app, all of the following are currently un-modelled and
therefore ungenerated, so they silently succeed:

- `rename-task` / `set-completion` on an already-**deleted** task (delete is a soft event;
  the stream survives, so the aggregate happily records more events);
- `delete-task` on an already-deleted task (double delete);
- redundant `set-completion` toggles.

These are *lifecycle* invariants — exactly the class a state machine formalises.

## Why an extension, not a core field

The natural shape would be an optional `lifecycle:` block on the `aggregate` kind. That is
**not** viable without an upstream core-schema change: ESDM's core schema is strict
(`unevaluatedProperties: false`), so an unknown `lifecycle:` key on an `aggregate` document
fails `esdm lint`. ESDM's own extension philosophy resolves this — *"extensions add document
kinds around the core without injecting into it; core docs never validate against an
extension and vice versa."* (existing ESDM design). So this proposal defines a **new kind in
an extension api group**, in a separate document that references the aggregate by name —
mirroring how the Given-When-Then extension's `scenario` references core commands/events.

If upstream later adopts it, the same grammar can graduate into an optional
`aggregate.lifecycle` core block; the design below is written to make that migration trivial.

## Specification

A new document kind `state-machine`, scoped to one aggregate.

```yaml
apiVersion: schema.esdm.io/state-machine/v1   # extension group (illustrative; to be assigned)
kind: state-machine
name: task-lifecycle
scope:
  domain: todo
  boundedContext: tasks
  aggregate: task            # references an existing core `aggregate` by name
initial: open                # state after the creating event
states:
  - name: open
  - name: completed
  - name: deleted
    final: true              # no transition may leave a final state
transitions:                 # "evolve": events move the machine
  - on: task-added      to: open
  - on: task-deleted    to: deleted
admits:                      # "decide": which commands are valid in which states
  - command: rename-task     from: [open, completed]
  - command: set-completion  from: [open, completed]
  - command: delete-task     from: [open, completed]
```

Two complementary facets, matching the functional **decider** decomposition (see *Prior
art*):

- **`transitions` (evolve)** — each entry maps an aggregate `event` to the state it produces.
  This is the lifecycle projected onto the event stream.
- **`admits` (decide)** — each entry lists the states from which a `command` is admissible.
  This is what a generator turns into a guard.

### Fields

| Field             | Type                                                | Required | Notes                                               |
|-------------------|-----------------------------------------------------|----------|-----------------------------------------------------|
| `scope.aggregate` | string                                              | yes      | Must reference a declared `aggregate`               |
| `initial`         | state name                                          | yes      | State entered by the creating event                 |
| `states[]`        | `{ name, final? }`                                  | yes      | `final: true` ⇒ terminal                            |
| `transitions[]`   | `{ on: event, to: state }`                          | yes      | `on` must be an event the aggregate emits           |
| `admits[]`        | `{ command, from: [state, …], when? }`              | no       | Omitted command ⇒ admissible in any non-final state |
| `admits[].when`   | FEEL string ([0002](0002-feel-rule-expressions.md)) | no       | Data/temporal guard, AND-composed with `from`       |

### The prose→formal boundary (explicit non-goal)

Some transitions depend on **event payload**, not just on the event type. In `todo`,
`task-completion-changed` should move the machine to `open` *or* `completed` depending on its
`completed` field. v1 deliberately supports only **unconditional** transitions. Two honest
ways to stay inside v1:

1. **Split the event** into `task-completed` / `task-reopened`, so each transition is
   unconditional (the recommended, most event-sourcing-idiomatic option); or
2. model completion as a plain `state` data field and keep only existence states
   (`active`, `deleted`) in the machine — already enough to generate "no edits after delete".

**Guarded transitions** (a predicate over event/command data) are where a real expression
language begins — that language is [0002](0002-feel-rule-expressions.md) (FEEL). The two
proposals compose in exactly one place: an `admits[]` entry MAY carry a `when:` FEEL guard,
AND-composed with `from` — covering the common *"admissible from `sent` **and**
`validUntil >= today()`"* shape. *Transitions* themselves remain unconditional in v1.

## `esdm lint` rules this enables

Because the machine is structured, validation is mechanical (and slots straight into a
generator's pre-codegen gate):

- every `transitions[].on` / `admits[].command` references an event/command that exists on
  the scoped aggregate;
- every `to` / `from` references a declared state; `initial` ∈ `states`;
- no transition has a `final` state as its source;
- **reachability** — every non-initial state is reachable via some transition;
- **coverage** (warning) — every event the aggregate emits appears in `transitions` or is
  declared state-neutral;
- **determinism** (warning) — at most one unconditional transition per `(state, event)`.

## What a code generator can derive

From the table alone, deterministically, no prose and no LLM:

1. a **status enum** (`TaskStatus { Open, Completed, Deleted }`);
2. a `status` field on the aggregate, maintained inside the evolve step;
3. a **guard** at the top of each command's decide step (pseudocode — any target language):

   ```
   deleteTask():
       if status not in [Open, Completed]:
           reject IllegalTaskTransition("delete-task", status)
       record TaskDeleted(id)
   ```

4. a typed **`IllegalTaskTransition`** exception, mapped by the API layer to **409 Conflict**
   (instead of a generic 500);
5. **negative GWT scenarios** ("given a deleted task, when delete-task, then it is rejected"),
   because illegal `(state, command)` pairs are now enumerable;
6. richer `esdm view` / glossary output and a Mermaid `stateDiagram` rendering.

## Relationship to existing ESDM kinds

- **`invariants`** — complementary, not replaced. *Lifecycle* invariants move from prose to
  structure here; *data* invariants ("title non-empty", "≤ 50 open tasks") remain prose until
  a guard-expression proposal exists.
- **`process-manager`** — already *is* a state machine in the sphere; this proposal gives the
  *aggregate* the same explicitness a process manager implicitly has.
- **`dynamic-consistency-boundary`** — orthogonal; DCB concerns the consistency boundary,
  this concerns the lifecycle within one.

## Backwards compatibility

Fully additive. An aggregate with no `state-machine` document behaves exactly as today.
Core documents are unaffected and continue to validate against the unchanged core schema.

## Prior art

The "aggregate as state machine" framing is foundational in this sphere; this proposal only
makes it declarative. Verified references:

- **Decider pattern** — Jérémie Chassaing: an event-sourced aggregate as `decide(command,
  state) → events` and `evolve(state, event) → state`. The `admits`/`transitions` split here
  is decide/evolve made declarative.
- **Statecharts** — David Harel, *Statecharts: A Visual Formalism for Complex Systems*,
  Science of Computer Programming 8(3):231–274, 1987 — the formal root of declarative state
  machines; a future "guarded transitions" proposal would draw on its guard/condition model.
- **XState / Stately** — David Khourshid — the modern "declare states + transitions, run the
  guards" tooling this grammar resembles.
- **Event Modeling** — Adam Dymitruk — the example-driven methodology (its GWT slices already
  appear in ESDM) that treats the write model as a walked-through state machine. *(That ESDM
  itself descends from Event Modeling is this project's inference, not an ESDM claim; what is
  documented is only that ESDM ships a Given-When-Then extension.)*

ESDM is by **the native web** (Golo Roden) — confirmed in the vendor's own docs.

## Implementation status

A reference implementation exists in **BPAG** (Business Process App Generator), an
ESDM→code generator (see the repository [README](../README.md)). Of this proposal it
implements:

- `states` / `transitions` / `admits` compile to a status field maintained in the evolve
  step and a guard at the top of each decide step; an illegal `(state, command)` pair
  raises a typed exception the HTTP layer maps to **409 Conflict**.
- Transitions are unconditional, as specified; `admits[].when` guards go through the
  [0002](0002-feel-rule-expressions.md) FEEL pipeline (validated pre-codegen, compiled
  into the same generated guard).
- The `esdm lint` rule set above (reachability, coverage, determinism) is the design
  target and still open.

See the [examples](../examples/) for real documents consumed by the generator.

## References

- thinkbeforecoding — Functional Event Sourcing Decider: https://thinkbeforecoding.com/post/2021/12/17/functional-event-sourcing-decider
- Harel, Statecharts (1987): https://www.sciencedirect.com/science/article/pii/0167642387900359
- Stately / XState: https://stately.ai/docs
- Event Modeling: https://eventmodeling.org/
- ESDM: https://www.esdm.io/ · the native web: https://www.thenativeweb.io/
