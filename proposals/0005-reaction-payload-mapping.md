# ESDM Extension Proposal 0005 — Reaction Payload Mapping

- **Status:** Draft / proposed (not part of upstream ESDM) — **not implemented**, see *Implementation status*
- **Target:** a *convention* over the generator's config and the `esdm-extensions.io/*` annotation
  channel — **no schema change**
- **Author:** Ralf Süss ([r-sw-eet](https://github.com/r-sw-eet))
- **Affects:** `policy` (what the commands in `emits[]` carry) and `event-handler.sideEffects[]`
  (what the effect is handed) — **additively**; absent a mapping, nothing changes
- **Related:** builds directly on [0002 — FEEL Rule Expressions](0002-feel-rule-expressions.md)
  (same language, same evaluator, same validation gate); composes with
  [0001](0001-aggregate-state-machine.md); observable through
  [0004](0004-domain-console-contract.md)

> This document is a proposal written in ESDM's idiom. It is **not** a description of shipping
> ESDM behaviour. Where it makes claims about existing ESDM, those are marked **MEASURED** or
> **DOCUMENTED**.

## Summary

A `policy` document states, with real structure, *which event* it handles and *which command* it
emits. It does not state **what that command carries**. Nothing in core ESDM does: there is no
mapping, no payload, no field-correspondence construct anywhere in the `policy` block.

Every generator therefore invents one. **MEASURED:** two independent implementations converged on
the *same* unwritten three-rule cascade, which is the good news, and neither can express a rename
or a computed value, which is the bad news — those fall through to a silent type default rather
than an error.

This proposal fills that gap without inventing a language and without touching the schema: **a
reaction's payload mapping is a FEEL context expression**, evaluated against the handled event's
data, carried as a string in generator config or in an `esdm-extensions.io/mapping` annotation.

## Motivation

### The construct stops one field short

**DOCUMENTED**, from the core schema. A policy carries `scope`, `deliveryGuarantee`,
`idempotency`, `handles[]` (event reference triples), `emits[]` (full command reference triples)
and `constraints[]` (prose rules). Real, closed, checkable structure — the schema even requires
`idempotency` when `deliveryGuarantee: at-least-once`.

Here is a complete, valid, lint-clean policy:

```yaml
kind: policy
name: draft-quote-on-request
scope: { domain: manufacturing }
deliveryGuarantee: at-most-once
handles: [{ boundedContext: intake,  aggregate: request, event: request-submitted }]
emits:   [{ boundedContext: quoting, aggregate: quote,   command: draft-quote }]
```

`draft-quote` declares `data: { requestId: string }`, `required: [requestId]`. The document does
not say that `requestId` comes from the handled event's `id`. It does not say whether the new
`quote` gets a fresh identity or inherits the request's. Both are decisions a code generator
**must** make, and the model is silent on both.

### What the generators actually do, measured

**MEASURED**, by reading the reaction emitters of two independent generators (a TypeScript one and
a Java one) and by generating the same model through both. Each fills the emitted command field by
field, with the identical cascade:

| # | Rule | Result |
|---|------|--------|
| 1 | the field is named `<handledAggregate>Id` | the handled event's identity |
| 2 | the field is the reacting aggregate's identity and the command does not declare it | a minted id |
| 3 | the handled event has a field of **exactly** the same name | passthrough |
| 4 | anything else | **a type default literal, silently** |

Generated from the same model, they emit the same mapping:

```java
// Java target
new DraftQuoteCommand(UUID.randomUUID().toString(), event.id(),
                      event.customerName(), event.product(), event.quantity())
```
```ts
// TypeScript target
data: { requestId: event.data.id, customerName: event.data.customerName,
        product: event.data.product, quantity: event.data.quantity }
```

So the situation is **not** four generators disagreeing. It is four generators quietly agreeing on
a rule that no document states, that no reviewer can see, and whose rule 4 turns *"this field is
not expressible"* into *"this field is `0`"* with no diagnostic. A model can look complete and
correct while a reaction invents data.

That also bounds this proposal honestly: it does not fix a divergence, it makes an existing
convention **declared**, and adds the two cases the convention cannot reach.

### The silence is not harmless: two apps need opposite answers

- A quoting policy must **mint** a new `quote` id and carry the request id as a *reference*.
  Reusing the handled id keys every quote by its request, which is a different domain model.
- A metering policy in another application does the reverse **on purpose**: the usage record *is*
  keyed by the job it meters, so one terminal job can only ever produce one usage row. That
  identity choice is what makes its deduplication work.

Same construct, opposite semantics, and nothing in either document distinguishes them. Both were
discovered by a generator producing the wrong one and a conformance suite catching it.

### Conformance only pins what a fixture happens to touch

**MEASURED**, over the published conformance data and example models:

| | |
|---|---|
| policies across all example models | 6 |
| of those exercised by a conformance scenario | **1** |
| properties in that policy's emitted command | **1** (`requestId`, required) |
| example models declaring `policy.constraints` | **0** |
| example policies with `deliveryGuarantee: at-least-once` | **0** |

Where a fixture does reach the mapping, the golden pins it well — captured identifiers are
normalized to placeholders, so `{"id": "«Q1»", "requestId": "«R1»"}` encodes the *relationship*
and not just the values, and a generator that reused the request id as the quote id fails. But
that covers a one-field mapping and one identity decision. Multi-field mappings, renames,
constants, computed values and conditional firing are exercised **nowhere**, so four
implementations may already disagree about them without any signal.

The remedy is not more fixtures alone. A shared convention that no document states is a
convention that drifts; conformance can only report the drift after the fact.

## Why a FEEL context expression, and why not a new field

- **Don't invent.** [0002](0002-feel-rule-expressions.md) already adopts FEEL for the *predicate*
  half of behaviour, and FEEL already has a context literal — `{ a: x, b: y.z }` — whose entire
  purpose is building a structured value out of an input. A mapping is that, exactly. Using it
  means the "language" for mappings needs no design at all.
- **The evaluator already exists, four times over.** Every generator that implements 0002 has a
  FEEL lexer, parser, binder and compiler. This proposal reuses them for a value-typed expression
  instead of a boolean-typed one.
- **The binding rule already exists.** 0002 already states that in a `policy` constraint,
  identifiers bind *against the handled event's data*. 0005 evaluates in the same context, so an
  author who has learned one has learned both.
- **A string is the only carrier available, and it is enough. MEASURED**, on a `policy` document:
  `metadata.annotations: { "esdm-extensions.io/mapping": "{ requestId: id }" }` lints **identical
  to baseline** (8 findings, 0 errors); the same key with a structured YAML value is **rejected**
  with `esdm/structure/type-mismatch: expected [string], got object`. Annotations are pinned to
  strings, and a FEEL context literal is a string. No schema change is needed or possible.

## Specification

### The convention

A reaction MAY carry a **mapping expression**: a FEEL context literal whose keys are properties of
the emitted command's `data`, and whose values are FEEL expressions evaluated against the handled
event's payload.

```yaml
kind: policy
name: draft-quote-on-request
metadata:
  annotations:
    esdm-extensions.io/mapping: "{ requestId: id }"
```

Where a document must not be touched, the same expression may live in generator config, keyed by
policy name (recommended for models shared across generators with differing rollout):

```yaml
options:
  reactions:
    draft-quote-on-request: "{ requestId: id }"
```

Config wins over annotation when both are present, so a generator can be pinned without editing
the model.

### Identity, the rule that makes both applications expressible

The emitted command targets an aggregate with a declared identity field.

- If the mapping **assigns** that field, the reaction uses the assigned value.
  `"{ id: id, accountId: accountId, seconds: durationSeconds }"` keys the new aggregate by the
  handled event's id — the metering case.
- If the mapping **omits** it, the generator mints a fresh identity — the quoting case.

That single rule is the whole difference between the two applications above, and it is now stated
in the document rather than chosen by the generator.

### Defaults, so nothing changes for existing models

With **no** mapping present, generators keep today's behaviour, which this proposal writes down
rather than alters: mint a fresh identity for the reacting aggregate, map the handled event's
identity onto the emitted command's `<aggregate>Id` field when one is declared, and pass through
properties whose names match on both sides. Existing models therefore emit byte-identical output.

### Event handlers

`event-handler.sideEffects[]` takes the same expression with the same binding. The result is not a
command but the payload the effect is handed — the data given to an `external-call` on the named
`external-system`, or to the `other` effect. A handler mapping never assigns an identity, since a
handler creates no aggregate.

### A mapping says *which* value, not *how* the target reads it

A generator must apply its own accessor and coercion conventions to a mapped value, exactly as it
does to a value the default convention supplies. **MEASURED** while implementing this in four
generators, where three got it wrong in three different ways and the byte-identity check below
caught each one:

- one target reads an untyped payload map into a typed constructor, so the mapped value needs the
  same cast the fallback applies;
- another supplies a per-field zero-literal default on every payload read, and the mapped value
  must too;
- on a third, the handled aggregate's identity is not a payload attribute at all but the stream
  id, so an identifier naming it resolves differently from every other field.

A mapping is a statement about *which* value belongs in a field. Rendering it is still the
target's business.

### The supported subset

Exactly [0002](0002-feel-rule-expressions.md)'s v1 subset, plus the context literal `{ k: expr }`
and nothing else. Values may be any 0002 expression: a field access, a literal, a conditional, an
arithmetic or temporal expression. Anything outside the subset is rejected at validation time,
never silently miscompiled.

## Validation — the same two layers as 0002

```
esdm lint (raw files, structure)        ← existing gate, unchanged, stays green
   → load documents
   → ModelFactory.create (resolved model)
   → FEEL validate (0002)               ← existing gate
   → mapping validate                   ← NEW gate, same place, same failure mode
   → adapter.generate
```

The mapping gate, which needs the resolved model because it compares two documents against each
other, rejects:

1. a key that is not a declared property of the emitted command's `data`;
2. a `required` property of that `data` that the mapping does not produce;
3. an identifier that binds to no property of the handled event's payload;
4. a value expression outside the subset;
5. a mapping on a policy whose `emits[]` names more than one command, unless the mapping is itself
   a context keyed by command name (v1 may simply reject this case, see *Out of scope*).

All are errors and all abort before the adapter is fetched, upholding the rule the generators
already state: an invalid model never reaches the adapter.

## What the generator compiles it to

```yaml
# ESDM
handles: [{ intake, request, request-submitted }]
emits:   [{ quoting, quote, draft-quote }]
metadata: { annotations: { "esdm-extensions.io/mapping": "{ requestId: id }" } }
```

```ts
// generated reaction (pseudocode — any target language)
onRequestSubmitted(event) {
    const command = {
        id:        newId(),          // identity not assigned by the mapping → minted
        requestId: event.payload.id, // from the mapping
    };
    commandRouter.route('quoting.quote.draft-quote', command);
}
```

## Examples

```yaml
# mint a new aggregate, carry the source as a reference
esdm-extensions.io/mapping: "{ requestId: id }"

# key the new aggregate BY the handled event, and carry three fields
esdm-extensions.io/mapping: "{ id: id, accountId: accountId, seconds: durationSeconds }"

# rename across the boundary
esdm-extensions.io/mapping: "{ jobId: id, startedAt: occurredAt }"

# a constant, for a reaction that always reports the same channel
esdm-extensions.io/mapping: "{ orderId: id, channel: \"email\" }"

# a conditional, which IS in 0002's subset
esdm-extensions.io/mapping: "{ id: id, tier: if amount >= 1000 then \"priority\" else \"standard\" }"
```

## How it composes

- **0002 decides whether a reaction fires** (`policy.constraints[].rule`, already in 0002's
  scope); **0005 decides what it carries.** Neither is useful alone for a code generator.
- **0001** governs whether the *emitted* command is admissible in the target aggregate's current
  state. A mapping produces a command; the state machine may still refuse it, and that refusal is
  the correct behaviour, not a mapping error.
- **0004** is how this becomes checkable across generators. A reaction's declared mapping is model
  data, so a generated app can list its reactions in `/_dev/catalog` alongside aggregates and read
  models. That makes the seam **observable**, and therefore something the conformance suite can
  compare, instead of a convention four implementations hold privately.

## Backwards compatibility

Total. The annotation is optional and measured lint-clean; absent it, generators emit exactly what
they emit today, which this document specifies as the default. A model that adopts a mapping keeps
linting with the unmodified upstream `esdm` binary, because the carrier is a string in a free-form
annotation map.

## Out of scope (v1)

- **`process-manager` reaction logic and timers.** A process manager has state, correlation and a
  lifecycle; its reactions map from *accumulated state* rather than from one event, and its
  `timers[].at` and `endsWhen` are a separate design. Worth its own proposal.
- **Fan-out with divergent payloads.** A policy emitting several commands that each need a
  different mapping. v1 rejects the case rather than guessing; the natural v2 is a context keyed by
  command name.
- **Arithmetic, and therefore most "computed" values.** **MEASURED while implementing this:**
  0002's v1 subset has no arithmetic operators at all - its lexer admits only `<= >= != = < >` -
  so `durationSeconds * rate / 3600` does not parse. A mapping value can be a field, a literal, a
  conditional, a temporal expression or a membership test, and nothing else. Genuine arithmetic
  needs 0002's subset to grow first, and that belongs in an amendment to 0002 rather than here.
  An earlier draft of this document used an arithmetic example; it was wrong.
- **Anything requiring data the handled event does not carry.** A mapping is a pure function of one
  event. Enrichment by lookup is a process manager or an application concern, deliberately.
- **Cross-store references.** Where an application keeps two event stores, the reference between
  them is a modelled field like any other and gets its value from a mapping like any other. The
  rules governing *which direction* such a reference may point are an application's design
  decision, not this proposal's.

## Implementation status

**None. Specification only** — deliberately, and this is the one place 0005 differs from its
predecessors. 0001 and 0002 were written after a generator implemented them; the strongest thing
about them at review time was that several independent implementations already agreed.

0005 should earn the same footing before it is proposed upstream:

1. **Extend a public conformance scenario first** so the surface this proposal formalizes is
   actually exercised. **Done (2026-08-10):** `manufacturing`'s `draft-quote` now takes four
   fields instead of one, so the golden pins the minted identity, the cross-reference and
   same-name passthrough together, and `customerEmail` is deliberately excluded so a generator
   that copies the handled event wholesale fails. The policy also carries the family's first
   `constraints[].rule`, written as FEEL per 0002. A rename and a computed value are **not** in
   the fixture on purpose: no implementation can express them, so recording them would enshrine a
   silently defaulted value as the normative answer.
2. **Record the golden**, confirming the generators already agree on the convention.
   **Done (2026-08-10):** re-recorded by the oracle runner, then verified against **all eight
   targets** — `symfony-esdb`, `nimbus`, `nimbus-postgres`, `python`, `python-esdb` and
   `opencqrs` green with only the registered `playhead` divergences, `opencqrs-postgres` green
   with **zero** divergences. Four independent implementations, three languages, produce the same
   four-field mapping from the same policy document. **The convention is therefore established as
   fact, not as intent** — which is precisely the footing 0001 and 0002 had when they were
   proposed.
3. **Then implement the annotation** in one generator and confirm the golden does not move — which
   is the proof that the default in this document describes reality. **Done (2026-08-10)** in the
   Java generator: a `Mapping` parser over FEEL context literals, the model carrying the raw
   annotation, a validation gate beside the existing FEEL one, and the emitter preferring an
   assigned field over the cascade. Verified three ways - a mapping stating the default emits
   **byte-identical** output; a deliberately swapped mapping **does** change the emitted reaction
   (so the annotation is not being ignored); and C4 still reports `16 records match` against the
   **unchanged** golden. The gate rejects an undeclared key, an unassigned required field and an
   identifier that binds to nothing.
4. **Then the remaining three**, with the same golden as the gate. **Done (2026-08-10).** All four
   generators implement the annotation. Three of the four needed the accessor-convention fix
   described in the specification above, every one of them found by the byte-identity check rather
   than by review.

5. **Put the annotation in the fixture, or the suite never runs the code.** **Done (2026-08-11),
   and it is the step that nearly went missing.** On 2026-08-10 the implementation was reported as
   verified because "all eight targets pass C4 against the unchanged golden" - but the fixture
   carried **no** annotation, so every target took the fallback path and the suite executed not one
   mapped reaction. The golden's stillness was evidence about the old code, not the new. The
   byte-identity proofs that *did* exercise it were run by hand over temporary copies: not
   committed, not repeatable, no use against a regression.

   Both halves are now fixed. Every generator carries a committed test that generates the example
   model twice, with and without the annotation, asserting byte-identity **and** that a
   deliberately swapped mapping reaches the emitted reaction; each was mutation-checked by
   disabling the mapping and confirming the second assertion fails while the first still passes.
   And `manufacturing`'s policy now carries the mapping in the model, so C4 exercises the declared
   path on all eight targets - with the golden unchanged, which is the property worth having.

**The generalisable rule**: a feature whose fixture does not use it is covered by nothing, however
green the suite looks. That is the same failure this proposal diagnoses in policies - a convention
no document states - reproduced one level up in the verification of the fix.

The claim "four independent implementations, one conformance suite" therefore holds for this
proposal the way it already holds for 0001 and 0002 - with the honest caveat that here the
implementations came *after* the document rather than before it, so the document has not yet been
tested by an author who did not write it.

## Prior art

- **DMN / FEEL context expressions** — the exact construct borrowed here, used in DMN for building
  structured decision output.
- **Camunda input/output mappings** — the same idea in a workflow engine: a small expression per
  target field, evaluated against the incoming payload.
- **EventStorming** — policies are drawn as "whenever X, then Y", and the payload correspondence is
  precisely what the sticky note leaves implicit and an implementation must supply.

## References

- [0002 — FEEL Rule Expressions](0002-feel-rule-expressions.md), whose subset, evaluator and
  validation gate this proposal reuses wholesale
- [0004 — Domain Console Contract](0004-domain-console-contract.md), for the catalog surface that
  would make reactions observable
- ESDM core schema, `policy` and `event-handler` blocks
