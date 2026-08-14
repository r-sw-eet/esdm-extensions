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
without becoming a general-purpose language.

**Every row below was run through the reference parser on 2026-08-14**, and each is shown as an
expression rather than a category, because a category hides the difference between "the parser
takes this" and "the proposal wishes it did". A reader must be able to tell what a generator
accepts without reading a generator.

**Live** - parses, binds, compiles:

| Expression | |
|---|---|
| `amount >= 100` · `status != "sent"` | the six comparisons |
| `amount > 1 and status = "sent"` · `not(amount > 1)` | boolean |
| `(a or b) and c` | grouping |
| `status = "sent"` · `amount = 12.5` · `a = true` | string, number, boolean literals |
| `validUntil >= today()` · `now()` | clock, compiled to injected values |
| `status in ["sent","draft"]` · `qty in [1,2,3]` | membership over a literal list |

**Not live, though small and worth having** - in rough order of cost:

| Expression | What happens today | |
|---|---|---|
| `amount > -1` | rejected at the lexer | negative literals are unwritable at all: one token |
| `a = null` | **parses, then binds wrong**: `unknown field "null"` | `null` lexes as an identifier, so the error blames the model for the parser's gap |
| `customer.name` | rejected, no `.` token | nested payloads are ordinary |
| `qty between 1 and 10` | rejected | sugar for two comparisons, and the most readable form for a domain expert |
| `qty in [1..10]` | rejected, no `..` token | ranges |
| `amount + 1` · `amount * qty` | rejected | the arithmetic amendment below |
| `if qty > 1 then "bulk" else "single"` | rejected | real parser work, not a token |

**Blocked on one thing.** All of these fail for the same reason - the AST's call node carries a
function name and **no arguments**, so any call taking one is unrepresentable:

`date("2026-01-01")` · `duration("P14D")` · `count(items)` · `sum(items)` ·
`starts with(s, "x")` · `contains(s, "en")` · `every i in items satisfies …` ·
`some i in items satisfies …` · `items[qty > 1]`

That single AST change unlocks all of them at once, including the temporal row this proposal has
claimed since v1 and which has never been reachable: `validUntil + duration("P14D")` needs both a
`+` token and call arguments.

Anything not *live* is **rejected at parse time**, never silently miscompiled. The one exception is
`null`, which is not rejected but misattributed, and is a bug rather than a design.

## Amendment (2026-08-14): arithmetic

### Why now

[0005](0005-reaction-payload-mapping.md) makes a reaction's payload a FEEL context literal, so a
mapping can say *which* value a field takes. It cannot say `durationSeconds * rateCentsPerHour`,
because this subset has no arithmetic at all - a gap found by writing that example into 0005 and
then discovering it does not parse. Every computed value therefore stays hand-written, which is
precisely the seam these proposals exist to close.

### Operators

`+`, `-`, `*`, `/`, and unary `-`. No exponent, no modulo: both are rare in domain rules and each
adds a semantics argument (negative exponents, sign of the remainder) that buys nothing here.

Unary minus also fixes an accidental hole: **negative literals are not expressible today**, so
`amount > -1` cannot be written at all.

### Precedence and associativity

Tightest to loosest, all left-associative:

```
unary -        →  * /        →  + -        →  comparison        →  not(…)  →  and  →  or
```

Parentheses already group and already parse, so `(a + b) * c` needs no new syntax. This ordering is
the conventional one; it is written down because four hand-written recursive-descent parsers have to
agree, and a table is cheaper to agree on than four implementations.

### The number domain, and the trap that makes this worth specifying

**All arithmetic evaluates in the real-number domain. Integer division must never occur.**

This is not pedantry, and it is measured rather than assumed: of the four implementation languages
only Java divides integers as integers (`7 / 2` on two `long` fields is `3`, against `3.5`
everywhere else - see the table below). A model that says `total / count` would then mean different
things on different targets while every generator looked correct - the identical failure mode 0002
already documents for comparisons, where
`Objects.equals(Long, Integer)` silently answers false. The remedy is the same one the reference
toolchain already applies to comparisons: **arithmetic compiles through the emitted per-target
helper**, not to bare operators, so the coercion lives in one place per target and is testable
there.

**Division by zero.** A literal zero divisor is a validation error. At runtime a zero divisor
yields FEEL's `null`, and the two contexts differ deliberately: in a predicate, `null` is false, so
a guard refuses; in a 0005 mapping, `null` must **not** be written into a command field - the
reaction fails loudly and dispatches nothing.

**Precision, and what actually goes wrong.** DMN specifies decimal arithmetic; all four reference
targets use binary64. This amendment does **not** claim decimal semantics, because none of the four
implement it. But the usual warning that follows - *don't compute money* - is too blunt, and the
interesting part is which half of it survives measurement.

**MEASURED (2026-08-14)**, the same expressions in all four implementation languages. Numeric
semantics belong to the language rather than the target, so this is four rows and not eight:

| | PHP | TypeScript | Python | Java |
|---|---|---|---|---|
| `0.1 + 0.2` | `0.30000000000000004` | `0.30000000000000004` | `0.30000000000000004` | `0.30000000000000004` |
| `1234567 * 89` | `109876463` | `109876463` | `109876463` | `109876463` |
| `7 / 2`, integer operands | `3.5` | `3.5` | `3.5` | **`3`** |
| rendering of the value `93.0` | `93` | `93` | **`93.0`** | **`93.0`** |

So **arithmetic does not diverge**: binary64 is deterministic, and identical inputs in identical
order give identical bits in all four. Two things do diverge, and neither is the sum itself.

The first is the integer division above, and it is a single-language problem with a single-language
remedy - but the remedy has to be placed carefully.

**Where the coercion goes in Java.** Not `(double)` at each emission site: that is a cast a
generator can forget at one call site out of twenty, and nothing would catch it. It goes in the
emitted helper, exactly where comparisons already live for the same reason - `Guards.compare` and
`Guards.equal` exist because Java has no operator that spans FEEL's comparisons, and Java has no
operator that spans FEEL's division either. One place per target, tested there.

**And the whole expression, not the division node.** `a * b / c` would otherwise multiply two longs
and then divide two longs. So arithmetic evaluates in the double domain throughout, which means the
result arrives as a double and must be **coerced to the field's declared type on assignment** - the
same declared-type rule that settles the rendering split below, doing double duty.

**The boundary, admitted rather than discovered.** Binary64 is exact to 2^53. Inside that, the four
languages agree; outside it they genuinely differ, since Java `long` and Python `int` stay exact
where TypeScript and PHP floats do not. A model is expected to stay inside, and nine quadrillion is
a comfortable ceiling for quantities and for money in minor units alike.

**The second is rendering, and it splits two against two.** Java and Python print `93.0` where PHP and TypeScript
print `93`, so a conformance golden recorded from one oracle would flag the others on a value that
is numerically identical everywhere. This needs a rule, and it is not an arithmetic rule:

> A field's declared type governs its emitted representation. A non-integral result assigned to a
> field declared `integer` is a validation error, not a silent `93.0`.

That leaves the ordinary float-money problem, which is not special to ESDM. It has the ordinary
answer: **money in integer minor units**. In cents, `+`, `-` and `*` by an integer are *exact* in
binary64 up to 2^53 - about nine hundred billion euro - so a model may compute money that way with
no caveat at all.

**Division is the real exclusion, and not because floats are frightening.** A money quotient needs a
rounding rule - half-up, half-even, floor - and this subset cannot express one: `round(x, 2)` is a
call with arguments, and the AST's call node carries none. So the rule is not "no money" but:

> Do not divide money in the model until rounding can be stated. An unrounded monetary quotient is
> wrong in every language equally.

0005's own motivating example, `durationSeconds * rateCentsPerHour / 3600`, is exactly that case: the
multiplication is exact and unproblematic, and it is the `/ 3600` that has no defensible answer here
yet. A future amendment can add rounding (it needs call arguments) or decimal semantics (it needs
all four targets to change), and either would lift this restriction.

### What the gate must additionally reject

Beyond the existing binding check, two rules the model already has the information to enforce:

1. **Operands must be numeric.** A field declared `type: string` or `type: boolean` in `state` or
   `data` is a validation error under an arithmetic operator. The model knows every field's type;
   this is the first place 0002 would use it.
2. **A literal zero divisor** is a validation error, as above.
3. **A division assigned to an `integer` field** is a validation error, because a quotient is not
   generally integral and the two halves of the family render such a value differently (`93` against
   `93.0`). Either the field is a `number`, or the expression is not a division.

All three abort before the adapter, like every other gate.

### Still out of scope after this amendment

`date ± duration` stays blocked. It reads like arithmetic, but it needs `duration("P14D")` and
`date(...)` - calls with arguments - and the AST's call node carries no arguments at all. That is an
AST change and belongs in its own amendment, together with the collection quantifiers that are
blocked for the same reason.

### Before any of this is implemented

Add a conformance fixture that uses arithmetic, and record the golden, **before** writing the
compiler in any generator. 0005 shipped its implementation with a fixture that never exercised it,
so eight green targets proved nothing about the new path; the same mistake is available here and is
cheaper to avoid than to repair.

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
  state-machine documents, and - since [0005](0005-reaction-payload-mapping.md) - the value
  expressions of a reaction mapping. Extending the pipeline to `invariants` / `endsWhen` /
  `timers` is open, as are ranges (`[1..10]`), `if … then … else`, durations/date arithmetic and
  the collection quantifiers.
- **Arithmetic (amendment 2026-08-14): specified, not implemented.** No parser accepts `+`, `-`,
  `*`, `/` or a negative literal today; the amendment above says what they should mean and in what
  order they bind. Nothing in the toolchain has changed yet.

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
