# ESDM Extension Proposal 0004 — Domain Console Contract

- **Status:** Draft / proposed (not part of upstream ESDM) — implemented, see *Implementation status*
- **Target:** a *runtime HTTP contract* exposed by generated applications — **no schema change,
  no new kind**; nothing in the ESDM model itself changes
- **Author:** Ralf Süss ([r-sw-eet](https://github.com/r-sw-eet))
- **Affects:** applications generated *from* an ESDM model (any target stack); the model files
  themselves are untouched
- **Related:** carries [0001](0001-aggregate-state-machine.md) lifecycles and
  [0002](0002-feel-rule-expressions.md) FEEL guards to the UI; the
  [0003](0003-bpmn-to-esdm-mapper.md) authoring diagram is served for round-tripping

> This document is a proposal written in ESDM's idiom. It is **not** a description of shipping
> ESDM behaviour. Where it makes claims about existing ESDM, those are marked.

## Summary

A generated event-sourced application is verified by *using* it: send a command, watch it become
an event, see the read models update. That calls for a small **domain console** UI — and the
console should not have to care which stack the generator targeted.

This proposal fixes the **wire contract between a generated application and a stack-agnostic
domain console**: the (already uniform) domain surface, plus three development-only endpoints —

| Endpoint            | Returns                                                                          |
|---------------------|----------------------------------------------------------------------------------|
| `GET /_dev/catalog` | the **catalog** — a JSON self-description of the app derived from the ESDM model |
| `GET /_dev/bpmn`    | the authoring BPMN diagram (0003), if the app was authored top-down              |
| `GET /_dev/events`  | the newest slice of the **raw event stream**, in a uniform row shape             |

— plus permissive CORS while in development mode, so the console can run on another origin.

Any conforming app can be driven by any conforming console. The reference consumer is
**esdm-vue-reader** (a Vue 3 viewer); the reference producers are the ESDM toolchain's
generator targets (Symfony, Nimbus and Django flavors, on PostgreSQL or EventSourcingDB).

## Motivation

Two forces meet here:

1. **The console must exist.** Domain experts and developers need to exercise the generated app
   — fire commands, inspect the append-only log, watch projections — *before* any end-user UI
   is built. The generator knows everything needed to render such a console from the model.
2. **The console must not be per-target.** A generator with N target stacks (possibly written in
   N languages) would otherwise embed N copies of the same UI as string templates — the worst
   kind of duplication. And the *stores* differ per target (PostgreSQL event-store table here,
   EventSourcingDB there), so a shared console cannot talk to storage directly.

The consequence: **stack independence has to live at the HTTP level.** Each generated app exposes
the same small, read-mostly surface; one console implementation consumes it. The generator emits
per-app *data* (the catalog, the diagram) instead of per-app *UI*.

This extends the pipeline picture by one layer at the far end:

```
BPMN / DMN ──0003──▶ ESDM (core + 0001 + 0002) ──generator──▶ app ──0004──▶ console / viewer
(authoring)          (intermediate representation)          (runtime)      (one UI, all targets)
```

## Specification

Conformance keywords **must** / **should** / **may** are used in the RFC-2119 sense.
A JSON value marked `T|null` is present but nullable; consumers **must** ignore unknown
extra fields anywhere in the contract, so producers can extend it additively.

### 1. Domain surface (normative recap)

The contract builds on the uniform route scheme generated apps already share:

- `POST /<context>/<command>` — JSON body per the command's fields. Success is 2xx; a
  creating command returns `{ "id": "<new aggregate id>" }`. A rejected domain rule
  (0001 transition, 0002 guard) is `409` with `{ "error": "<human-readable reason>" }`;
  other client errors are 4xx with the same error shape.
- `GET /<context>/<query>` — a list query returns a JSON array of rows; a parameterised
  query takes its parameters as query-string values (e.g. `?id=…`) and returns one row.

Row keys are **the app's own naming** — see the catalog rule below.

### 2. `GET /_dev/catalog` — the catalog

Returns `application/json`, describing everything a console needs to render itself:

```jsonc
{
  "domain": "todo",                    // the model's domain name
  "contexts": [
    {
      "name": "tasks",                 // bounded context (also the route prefix)
      "commands": [
        {
          "name": "accept-quote",
          "lifecycle": "create" | "mutate" | "delete",
          "path": "/tasks/accept-quote",     // POST here
          "fields": [
            {
              "name": "id",
              "type": "string" | "boolean" | "integer" | "number",
              "feel": null | {               // 0002-derived input hints for this field
                "temporal": null | "date" | "datetime",
                "values": ["sent", "42"],    // literals the guards compare against
                "rules":  ["validUntil >= today()"]
              }
            }
          ],
          "guard": null | {                  // 0001-derived admit guard, for display
            "from": ["sent"],
            "when": null | "validUntil >= today()"
          }
        }
      ],
      "queries": [
        {
          "name": "list-tasks",
          "path": "/tasks/list-tasks",
          "kind": "list" | "get",
          "params": [ { "name": "id", "type": "string" } ],
          "readModel": "tasks"
        }
      ],
      "readModels": [
        {
          "name": "tasks",
          "columns": [
            { "name": "title", "type": "string", "identity": false }
          ],
          "listPath": null | "/tasks/list-tasks",   // the list query feeding this table
          "stateMachine": null | {                  // 0001, when the rows carry a status
            "statusColumn": "status",
            "initial": "draft",
            "states": [ { "name": "draft", "final": false } ],
            "admits": [
              { "command": "accept-quote", "from": ["sent"],
                "when": null | "validUntil >= today()", "to": null | "accepted" }
            ]
          }
        }
      ]
    }
  ]
}
```

**The column-name rule:** `readModels[].columns[].name` **must** be spelled exactly as the app's
own query responses spell the row keys — the catalog describes *this app*, not the abstract
model. (The reference Postgres target returns snake_case columns, the Mongo target camelCase;
each app's catalog matches itself, and the console never needs to know.)

The `feel` hints exist so a console can offer *rule-conform* inputs: a field compared against
`today()`/`now()` in any 0002 guard is temporal; literals it is compared or `in`-tested against
become suggested values; `rules` carries the source expressions for display.

### 3. `GET /_dev/bpmn` — the authoring diagram

Returns the app's authoring BPMN 2.0 XML (`application/xml`) — the file the 0003 mapper consumed
— or an empty body when the app was not authored from BPMN. Consoles use it to show/edit the
diagram; writing changes back goes through the authoring workflow (export → `bpmn:map` →
regenerate), **not** through this endpoint.

### 4. `GET /_dev/events` — the raw event stream

Returns `application/json`: an array of the **newest events first**, bounded (the reference
implementations return at most 50). Each row:

```jsonc
{
  "id": "137",                  // store-assigned event id (string or integer)
  "aggregate": "task",          // the aggregate the event belongs to
  "aggregate_id": "0197…",      // the aggregate instance
  "playhead": 3 | null,         // per-aggregate sequence, when the store tracks one
  "event": "task-added",        // the stack's event-type identifier, display-only
  "payload": { … },             // the event's business data
  "recorded_on": "2026-07-07T09:15:00Z"   // store timestamp, display-only string
}
```

The shape is deliberately lowest-common-denominator: every event store has these facts, even if
it names them differently. `playhead` is `null` where the store has no per-subject sequence;
`event` formats differ per stack and consumers **must** treat them as opaque labels.

### 5. CORS

While the app runs in development mode it **must** answer cross-origin requests on the whole
contract surface (domain routes *and* `/_dev/*`): allow any origin, methods `GET, POST, OPTIONS`,
the `Content-Type` request header, and answer `OPTIONS` preflights with 2xx. Without this, a
console served from another origin (its own dev server) cannot connect.

### 6. This is a development window

`/_dev/*` exposes the raw event log and the model's inner structure. Producers **must not**
expose it in production deployments — gate it by environment or strip it from production builds.
The reference stacks only ship it in their dev compose setups.

## Backwards compatibility

Purely additive. The ESDM model is untouched — `esdm lint` never sees any of this. Apps that
predate the contract simply lack `/_dev/*`; a console should fail with a clear "not a 0004 app"
message when the catalog probe fails. Unknown-field tolerance (above) is the forward-compat
valve for evolving the catalog.

## Out of scope

- **Authentication / production hardening** — the contract is a dev tool; §6 forbids exposing
  it beyond dev.
- **Pagination or live streaming of events** — the console polls the bounded newest-slice; a
  richer history/subscription API is a different (production-grade) concern.
- **Writing BPMN back** — authoring round-trips through files and the 0003 mapper by design.
- **A UI specification** — this fixes the wire, not the console's look or features.

## Prior art

- **GraphQL introspection** — the catalog is introspection for a CQRS surface: the app
  self-describes, generic tooling renders.
- **EventSourcingDB's management UI** (`--with-ui`) and patchlevel's CLI tooling — per-store
  windows onto the log; 0004 is the store-agnostic equivalent one level up.
- **OpenAPI** — could describe the routes but not the ESDM semantics (lifecycles, guards,
  read-model/state-machine links); the catalog carries exactly those.

## Implementation status

- **Producers:** all six reference generator targets emit the contract:
  `symfony-patchlevel-postgres`, `symfony-eventsourcingdb`, `nimbus-eventsourcingdb`,
  `nimbus-postgres`, `django-eventsourcing-postgres` and `django-eventsourcingdb`.
- **Consumer:** **esdm-vue-reader**, the reference console — Vue 3, one build, no per-target
  code; point it at any conforming app's base URL.
