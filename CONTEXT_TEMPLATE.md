<!--
  SKELETON for `<lib>.context.md` — the adopter-facing integration guide that
  ships with a published JS library. Copy this to `<lib>.context.md`, fill it in,
  delete sections that don't apply, and delete this comment.

  What it is: the single file an adopter (or their AI agent) reads to wire the
  library WITHOUT reading the source. README is the pitch; this is the contract.
  See LIBRARY_CONVENTIONS.md §3 for why it exists and what belongs here.

  Rule of thumb: if an integrating agent would otherwise have to open your source
  to answer a question, the answer belongs in this file.
-->

# <lib> — Integration Guide

## What this is
<!-- One paragraph: the problem it solves and the one-line mental model. -->

## What <lib> is and is not
<!-- State the boundary up front. What it deliberately does NOT do, so an adopter
     doesn't reach for the wrong tool. -->

## Which mode / components do I need?
<!-- Only if there's more than one entry point or usage mode. A short decision
     guide: "want X → use A; want Y → wire B + C". Omit for single-purpose libs. -->

## Minimal usage
<!-- The smallest working example — the "6-line" wiring. Real, copy-pasteable. -->
```js
```

## All options
<!-- Every config option: name, type, default, what it does, and any constraint
     validated at startup. This is the table adopters live in. -->

## Public API
<!-- Each exported function/class: signature, what it returns, when to use it,
     and its contract (errors thrown, idempotency, side effects). -->

## <Extension> contract
<!-- If the lib accepts a custom store / mailer / transport / provider / etc.,
     document the interface the adopter must implement. One section per seam. -->

## Architecture
<!-- The shape in a few sentences + how the pieces fit. Enough to reason about
     it; not a re-derivation of the source. -->

## What's NOT in <lib>, and why
<!-- The refusals, each with its reasoning and where the responsibility lives
     instead (adopter / perimeter / MTA / etc.). This is the durable design record
     adopters and agents relitigate — answer it once, here. -->

## Threat model summary
<!-- Security-relevant libs only: the properties it guarantees, the attacker it
     assumes, and what is explicitly out of scope. -->

## Gotchas
<!-- The sharp edges adopters actually hit. Ordered by how often they bite. -->

## Constraints
<!-- Hard limits and non-negotiables (Node floor, ASCII-only, localhost-only,
     single-writer, etc.) and the one-line reason each is non-negotiable. -->
