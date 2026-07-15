# ESDM Extension Proposal 0002 — FEEL Rule Expressions

- **Status:** Draft / proposed (not part of upstream ESDM) — implemented (subset), see *Implementation status*
- **Target:** a *convention* over existing core string fields — **no schema change**
- **Author:** Ralf Süss ([r-sw-eet](https://github.com/r-sw-eet))
- **Affects:** `aggregate.invariants[].rule`, `process-manager.{invariants,constraints}[].rule`,
  `process-manager.endsWhen[].condition`, `policy.constraints[].rule`, and any future guard
  `condition` (see [0001](0001-aggregate-state-machine.md)) — **by interpretation only**
- **Related:** complements [0001 — Aggregate State Machine](0001-aggregate-state-machine.md)

> This document is a proposal written in ESDM's idiom. It is **not** a description of shipping
> ESDM behaviour. Where it makes claims about existing ESDM, those are marked.

## Summary

ESDM already has fields for rules — `invariants`, `constraints`, `endsWhen.condition` — but
they are **prose**: `type: string`, and ESDM's docs state outright that invariants are
*"descriptive, not executable."* This proposal adopts a **subset of FEEL** (the expression
language of OMG's **DMN** standard) as the agreed content of those existing string fields, so
the same descriptive rule a domain expert reads can also be **compiled to enforcement code**.

Crucially this is a **convention, not a schema change**: the fields stay `type: string`,
`esdm lint` stays green, and nothing in ESDM breaks. The new machinery lives entirely in the
generator — a model-aware FEEL validator (a second lint gate) and a FEEL→code compiler.

## Motivation

When generating code from an ESDM model, the same seam keeps appearing: **ESDM models structure
formally, but behaviour only as prose.** Value-shape rules already have a formal home (JSON
Schema constraints — `minimum`, `enum`, `pattern`). What has *no* formal home is
cross-field / stateful / temporal logic:

- "a quote can only be accepted while it is sent and not past its validity date"
- "the quoting process ends once the quote is accepted, rejected or expired"
- "expire the quote 14 days after it was sent"
- "a customer may not hold more than 5 open quotes"

Today these live in `rule`/`condition` as English. A generator can repeat the sentence as a
doc-comment but cannot *enforce* it. To turn ESDM into a **lossless intermediate** for a
domain-expert authoring pipeline (BPMN/DMN → **ESDM** → code), the one missing construct is a
machine-readable condition. This proposal supplies it without inventing a language: **FEEL is
that language**, it is a published OMG standard, it is small, and it reads almost like the
prose it replaces.

## Why FEEL, and why a convention (not a new kind)

- **Don't invent.** A bespoke rule DSL throws away a settled, tool-supported standard and
  trends toward reinventing a programming language. FEEL is the established rule/expression
  language (the "F" is literally *Friendly Enough Expression Language*).
- **Don't ingest BPMN/DMN XML.** Heavyweight tool formats fight ESDM's lightweight, git-native
  ethos. We borrow FEEL's *syntax and semantics*, expressed inside ESDM's existing fields.
- **Don't change the schema.** Unlike [0001] (which proposes a new `state-machine` *kind*),
  0002 needs **no** new kind and **no** new field. The `rule`/`condition` strings already
  exist; we only fix their interpretation. That is the lightest possible extension — it rides
  entirely on convention.

## Specification

### The convention

A `rule` / `condition` string MAY be a FEEL expression. When it is, it is interpreted against
the surrounding document's data:

- in an `aggregate` invariant/guard → against the aggregate's `state` fields;
- in a `process-manager` invariant/constraint/`endsWhen` → against the process `state` and the
  correlated events;
- in a `policy` constraint → against the handled event's data.

Identifiers are field names; the expression evaluates to a boolean (for guards/invariants/end
conditions) or a value (for decisions/timers).

### Declaring the language (optional, lint-clean)

Because `rule` is just a string, *"this string is FEEL"* is the generator's convention, not
ESDM's knowledge. Make it explicit one of two ways (both leave `esdm lint` green):

- **Generator config (recommended)** — the generator's own config file, keeps ESDM docs
  pristine:
  ```yaml
  options:
    rules:
      lang: feel        # interpret all rule/condition strings as FEEL
  ```
- **Document-level annotation** — ESDM-native, for models that mix languages per document:
  ```yaml
  metadata:
    annotations: { "esdm-extensions.io/lang": "feel" }
  ```

> Per-rule marking is **not** available: `invariants[]` / `endsWhen[]` entries are
> `additionalProperties: false`, so no metadata can attach to a single entry. If a single
> document must mix FEEL and prose, prefix the string (`rule: "feel: amount >= 0"`); otherwise
> declare the language at document or project level.

### Supported FEEL subset (v1)

The generator compiles a bounded subset — the regular-shaped ~80% that covers real invariants
without becoming a general-purpose language:

| Category            | Supported                                                                                     |
|---------------------|-----------------------------------------------------------------------------------------------|
| comparison          | `=`, `!=`, `<`, `<=`, `>`, `>=`                                                               |
| boolean             | `and`, `or`, `not(...)`                                                                       |
| membership / ranges | `x in ["a","b"]`, `x in [1..10]`                                                              |
| conditional         | `if … then … else …`                                                                          |
| temporal            | `today()`, `now()`, `duration("P14D")`, `date(...)`, date ± duration                          |
| collections         | `every x in xs satisfies …`, `some x in xs satisfies …`, `count(xs[pred])`, `sum(xs[].field)` |
| access              | path `a.b`, list filter `xs[pred]`                                                            |
| literals            | `"string"`, numbers, `true`/`false`, `null`                                                   |

Anything outside the subset is **rejected at lint time** (see below) — never silently
miscompiled.

## Validation — two layers, both before codegen

Putting logic in a string means `esdm lint` can only confirm the string *exists*, not that it
is correct. A second, **model-aware** gate restores the safety net:

```
esdm lint (raw files, structure)          ← existing gate, pre-parse
   → load documents
   → ModelFactory.create  (resolved model)
   → FEEL validate        ← NEW gate, post-parse (needs the model to bind identifiers)
   → adapter.generate
```

The FEEL validator does three escalating checks — the last two are things `esdm lint`
*cannot* do, because it does not hold the model:

1. **Parse (syntactic).** Valid FEEL grammar — catches `status == "sent"` (FEEL uses `=`),
   unbalanced parens, bad operators.
2. **Bind + type-check against the model.** Every identifier resolves to a real field
   (`statuss` → error); types are compatible (`amount >= "0"` → error; `validUntil >= today()`
   requires `validUntil` be a date); `in [...]` values respect the field's `enum`.
3. **Subset check.** Reject valid-but-unsupported FEEL with a clear *"unsupported FEEL
   feature"* rather than failing at codegen.

Result: **lint-green again means both structure *and* logic are valid** — a bad rule is caught
at generate time, with a location, not at runtime or never.

## What the generator compiles it to

FEEL conditions become **compile-time** predicates in the event-sourcing *decide* step — not a
runtime rule engine. Illustrative:

```yaml
# ESDM (a guard on accept-quote)
condition: status = "sent" and validUntil >= today()
```
```
# generated, in the aggregate's decide step (pseudocode — any target language)
acceptQuote(today):
    if not (status = "sent" and validUntil >= today):
        reject GuardViolation("quote-not-acceptable")
    record QuoteAccepted(id)
```

Other targets: `endsWhen.condition` → the process-manager's completion check; a `timers`
`firesAt: sentAt + duration("P14D")` → a scheduled deadline; a decision `if … then … else …`
→ a branch value. The temporal functions (`today()/now()/duration()`) give ESDM the **time**
its prose fields could only describe.

## Examples

```yaml
# guard / precondition — "can only be accepted while sent and still valid"
condition: status = "sent" and validUntil >= today()

# aggregate invariant — "the total can never be negative"
rule: amount >= 0

# process-manager end — "ends once accepted, rejected or expired"
condition: status in ["accepted", "rejected", "expired"]

# timer / deadline — "expire 14 days after it was sent"
firesAt: sentAt + duration("P14D")

# decision / branch — "orders over 10,000 € need approval"
decision: if amount > 10000 then "needs-approval" else "auto-approve"

# collection rules
rule: every item in lineItems satisfies item.quantity > 0
rule: count(quotes[status = "sent"]) <= 5
```

## Relationship to 0001 and to ESDM kinds

- **0001 (state machine)** handles *lifecycle transitions* — which command is admissible from
  which state. **0002 (FEEL)** handles *data and temporal predicates* — the conditions over
  field values and time. They are complementary: a guard often combines both (*"admissible
  from `sent`"* is 0001; *"and `validUntil >= today()`"* is 0002). Together they cover most of
  the behaviour gap ESDM leaves as prose.
- **`invariants` / `constraints` / `endsWhen`** keep their schema; their string content simply
  graduates from prose to FEEL where declared.
- **JSON Schema constraints** (`minimum`, `enum`, …) still cover single-field value rules;
  FEEL covers cross-field, stateful and temporal ones.

## Backwards compatibility

Fully additive and reversible. A `rule` left as prose is untouched (the validator skips it
unless the document/project declares FEEL). No core schema changes; `esdm lint` behaviour is
unchanged. A model with zero FEEL behaves exactly as today.

## Out of scope (v1)

- **Full FEEL** — only the subset above; unsupported features are a lint error, not silent.
- **A runtime rule engine** — rules compile to code; they are not evaluated from data at
  runtime (that is a different, heavier integration; see DMN engines like Camunda).
- **DMN decision *tables*** — the tabular rule format is a natural follow-up (very
  business-friendly), but v1 is FEEL *expressions* only.
- **Editing rules without regeneration** — out of band for a compile-time approach.

## Implementation status

The reference implementation (the **ESDM toolchain**, see the repository [README](../README.md)) ships
FEEL as a lexer / parser / compiler / model-aware validator:

- **Implemented syntax:** comparisons (`=`, `!=`, `<`, `<=`, `>`, `>=`), `and` / `or` /
  `not(…)`, membership over literal lists (`x in ["a", "b"]`), parentheses,
  string/number/boolean literals, field identifiers, and `today()` / `now()` (compiled to
  injected clock values).
- **Validation:** a model-aware gate runs after parse and before generation — parse errors
  and unknown-field binding errors abort generation with the offending expression; syntax
  outside the subset is rejected, never silently miscompiled. (Type-checking identifiers
  against their declared field types is open.)
- **Compilation:** the same AST compiles to native decide-step guards in each of the
  generator's target languages — no runtime rule engine.
- **Where it is wired:** `admits[].when` of [0001](0001-aggregate-state-machine.md)
  state-machine documents. Extending the pipeline to `invariants` / `endsWhen` / `timers`
  is open, as are ranges (`[1..10]`), `if … then … else`, durations/date arithmetic and
  the collection quantifiers.

## Prior art

- **DMN + FEEL** — OMG's Decision Model and Notation and its *Friendly Enough Expression
  Language*: the standard, small, business-readable rule/expression language this proposal
  adopts wholesale (subset).
- **SBVR** — the descriptive-rules lineage ESDM's prose `invariants` already echo; FEEL is the
  executable counterpart.
- **Event sourcing "decider"** (Chassaing) — FEEL conditions compile into exactly the *decide*
  half (`decide(command, state) → events|reject`).
- The audience framing (BPMN/DMN authored by domain experts → ESDM intermediate → code) makes
  FEEL the single construct needed to keep the **decision logic** from being lost at the ESDM
  layer — the one place that pipeline otherwise leaks.

## References

- DMN / FEEL specification (OMG): https://www.omg.org/spec/DMN/
- FEEL overview (Camunda docs): https://docs.camunda.io/docs/components/modeler/feel/what-is-feel/
- ESDM: https://www.esdm.io/  · the native web: https://www.thenativeweb.io/
- Proposal 0001 — Aggregate State Machine: ./0001-aggregate-state-machine.md
