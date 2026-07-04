---
name: domain-ontology
description: Use when building or loading a compressed world-model ("ontology") of a feature domain — before implementing in an unfamiliar area, when a spec and the code disagree, or when a state bug appears. An ontology captures a domain's Intent, Concepts, Relationships, States, and Rules, and — most importantly — the GAPS where spec/PRD intent and code reality diverge. Loading one lets a weaker model skip re-deriving the domain from raw specs + code. Use for "build an ontology for X", "map this domain", "why does the spec and code disagree", "onboard me to feature Y before I change it". Portable and repo-agnostic; ontologies are stored OUTSIDE the product repo.
---

# Domain Ontology (ODPM world-model)

> "ODPM" is the user's named method. Its structure is defined operationally below
> (Intent → Concepts → Relationships → States → Rules → Gaps → Open Questions).
> Do NOT invent an expansion of the acronym — if terminology matters, ask the user.

## Why this exists

A large codebase always offers enough surface to build a *plausible* mental model of a
domain — which is exactly how a weaker model goes confidently wrong. An ontology is a
**compressed, evidence-backed world-model** you load *instead of* re-deriving the domain
from raw PRD + code each session. Its single highest-value content is the set of **Gap
callouts**: the places where the spec's intent and the code's reality diverge. Those are
the landmines. A feature added without knowing them is silently wrong.

This is the context-compression layer referenced by the repo harness (AGENTS.md §5-style
"Context Compression via Ontology"): load the ontology, apply the **Observed State vs
Actual State** lens, and cite `file:symbol` for every claim.

## When to use

- **Loading (cheap, common):** an ontology already exists for the domain → read it
  instead of scanning raw specs/code; verify its Gaps still hold before relying on them.
- **Building (authoring):** onboarding to an unfamiliar feature area before changing it;
  a spec and the code disagree; a state/lifecycle bug appears; before adding a feature so
  it doesn't break an un-modeled case (e.g. a role, a terminal state, a missed event).

## Storage — OUTSIDE the product repo

Ontologies are **not** committed into the product repo (they drift from code and couple
docs to one repo). Store them in a workspace-external location:

```
~/.claude/ontology/<repo-slug>/<domain>.md          # default
~/.claude/ontology/<repo-slug>/CANDIDATES.md         # unreviewed new concepts
```

PRD↔code escalations do **not** get their own file — they live inline in the domain doc
(see "Drift vs Gap"). Triage across domains on demand with `grep -rn ESCALATE`.

Never add an ontology file to the product repo unless the user explicitly asks.

## Loading an existing ontology

1. Read the ontology file for the domain. Treat **Concepts / Relationships / Rules** as
   the compressed map; treat **Gaps / Open Questions** as active warnings. Honor any
   `[verified: <date>]` section tags — an untagged section is unverified, treat with
   the same suspicion as a claim from memory.
2. Check the `Snapshot` date and `Sources`. If code has moved since, re-verify before
   relying, in this priority order:
   a. the **State mapping table** (highest-leverage, most drift-prone — the raw→display
      status mapping against the actual mapper `file:symbol`);
   b. any **Gap** you're about to act on (`grep`/read both cited sides);
   c. any **count** you're about to quote (recompute from source — do not trust the
      written number).
3. Apply **Observed State vs Actual State** (below) for any state-related work.
4. New concept discovered while working → append to `CANDIDATES.md`; do **not** edit a
   reviewed ontology directly (that is a higher-tier / review action).

## Building an ontology (procedure)

Work from **both** sources and label every divergence:

1. **Intent** — 2-4 sentences: what the domain is *for*, from the PRD/spec. Name the
   app's actual role (e.g. "the app is a passive observer of vehicle state").
2. **Concepts** — one short paragraph per noun. For each, map to code: `file:symbol` or
   "not modeled in code". A concept the PRD names but code lacks → that's a Gap.
3. **Relationships** — bullet list of "A has/makes/contains B" edges.
4. **States** — if the domain has lifecycle, model it. Watch for **two-layer state**
   (raw backend/vehicle state vs the app's displayed interpretation) — conflating them
   causes bugs. Give the mapping table and flag any backend state with no distinct
   display state.
5. **Rules** — the business rules (access/roles, preconditions, retry, expiry,
   scheduling, notifications). Each should be traceable to a PRD line or a `file:symbol`.
6. **Gaps** (the point — see below).
7. **Open Questions** — things neither PRD nor code answers; resolve before new features.
8. **Sources** — table of PRD link + the code paths the model was built from.
9. **Verify before publishing (mandatory).** Before saving the ontology for others to
   load, run the `fact-check` skill over it. Claim inventory = concepts (`file:symbol`),
   status codes, **counts (recompute from source — do not read the written number)**, the
   **state-mapping table**, and each Gap's **two cited sides**. Prioritize counts and the
   mapping table (the drift-prone, high-value targets). Resolve every OVERSTATED /
   UNSUPPORTED by re-reading source — never by hedging. **Classify each finding as
   ontology drift (fix the doc) or a PRD↔code Gap (escalate — never doc-edit to match
   code); see "Drift vs Gap" below.** Only apply a `[verified: <date>]` tag to a section
   after it passes this pass.

### Evidence discipline (non-negotiable)

- Every concept/rule → cite the PRD location or `file:symbol`. No claims from memory.
- A **Gap** is a divergence between a **cited** intent and a **cited** code reality —
  not a vibe. If you can't cite both sides, it's an Open Question, not a Gap.
- **Quantitative claims must be computed, never asserted.** Any "N values / states /
  types / codes" figure is derived by counting the source at authoring time and pasting
  the count basis — never estimated from memory. Re-check it on load. (Real failure: an
  OTA ontology stated "UpdateStatus has 23 values" when the enum had 18 — every code was
  correct but the total was invented. A weak model then loads "23" as compressed truth.)
- Building an ontology is novel-judgment work → route to a strong model (Opus/Fable).

## The Gap callout — highest-value output

```
> ⚠️ Gap: <PRD/spec intent, cited> vs <code reality, file:symbol>.
> <Why it's a landmine — what future change will be silently wrong.>
```

Example (from a real OTA ontology):
> ⚠️ Gap: "Purged" has no dedicated display state. When the manufacturer cancels a
> campaign (status 40), the user sees "Update Error." Purged is semantically different
> from a failure — the vehicle didn't fail, Honda cancelled it. Any messaging keyed on
> `UPDATE_ERROR` will mislabel a cancellation.

## Drift vs Gap — never launder a divergence

Verification turns up two findings that look alike but must be handled oppositely:

- **Ontology drift** — the ontology misreads/miscounts the code, and the PRD is *not* in
  conflict (or agrees with the code). → **Fix the ontology document.** (e.g. "enum has 18
  values, doc said 23"; "doc said timer 5s, code and PRD both say 30s".)
- **PRD ↔ code Gap** — the PRD says A and the code does B, and they *genuinely differ*.
  → This is a ⚠️ Gap. **Do NOT rewrite the ontology to match the code and call it
  resolved** — that hides a real product/engineering divergence behind a tidy doc. Record
  it as a Gap and **escalate**, using structure the doc already has — no separate file:
    1. an inline `⚠️ Gap (PRD↔code) — ESCALATE` callout at the concept (full context), and
    2. a matching **ESCALATE** entry in the doc's **Open Questions** section (that section
       is the doc's escalation surface — both "unanswered" items and PRD↔code conflicts),
    3. surface it to the user for filing (Jira/PR/decision).
  The ontology's job is to *expose* the divergence, not to reconcile it into false
  agreement. Triage all escalations across domains on demand: `grep -rn ESCALATE`.

**Litmus test before "fixing" any finding:** *is the PRD side in conflict with the code?*
- No (pure doc-vs-code) → update the doc.
- Yes → escalate; the doc records the gap, it does not close it.

A numeric near-equivalence ("code uses `> 9`, PRD says `≥ 10%` — same for integer %") is a
**hypothesis for the escalation to resolve**, never a license to silence the gap. When in
doubt, escalate — a false "resolved" is more expensive than a redundant escalation.

## Observed State vs Actual State (state-bug lens)

When something holds ground truth elsewhere (a device, a server, a vehicle) and the app
reflects it via events:

1. Which state is the code **reading** — the observed (app-cached) one or the actual
   (ground-truth) one?
2. Which one does the **requirement** mean?
3. If an event can be missed/delayed, observed ≠ actual → is there reconciliation/re-poll?

Document the answer in the fix commit. This lens is what turns "the screen froze" into
"the app trusted a stale observed state with no reconciliation path."

## Output template

Tag each section that was checked against code with `[verified: <YYYY-MM-DD>]`. An
untagged section signals "asserted, not yet code-checked" to future loaders.

```markdown
# <Domain> Ontology

> Snapshot: <YYYY-MM-DD>
> Sources: <PRD ref> + <repo> implementation
> Method: ODPM — concepts/relationships/rules/states from PRD intent AND code reality;
> gaps between them named explicitly.

## Intent
...

## Concepts  [verified: <date>]
**<Concept>** — <definition>. In code: `<file:symbol>` (or "not modeled").
> ⚠️ Gap: ...

## Relationships
- <A> —has→ <B>

## States  [verified: <date>]
### Two-layer model (if applicable)
<!-- Count basis: <enum file:symbol> has <N> values (counted <date>) -->
| PRD concept | Backend state(s) | Display state |
### State flow
```<diagram>```

## Rules
### <group>
- <rule> (cite PRD / file:symbol)

## Open Questions   (also the escalation surface)
- <question neither PRD nor code answers>
- **ESCALATE** — <PRD↔code conflict>: PRD says A / code does B [verified: date] · ticket: — · found: date

## Source references
| Artifact | Location |
```

## Anti-patterns

- **Glossary with no Gaps.** A list of definitions isn't an ontology — the Gaps and the
  two-layer state model are the value. If it has no ⚠️ Gap and no Open Questions, you
  haven't compared intent to reality.
- **Asserted counts.** "N states/values/types" written without counting the source. Every
  individual item can be right and the total still invented — recompute, don't estimate.
- **Committing it into the product repo.** Store it outside (see Storage).
- **Inventing concepts** not present in either PRD or code.
- **Editing a reviewed ontology in place** with a freshly-discovered concept — use
  CANDIDATES.md and escalate for review.
- **Building it with a weak model.** Authoring requires novel judgment (Opus/Fable);
  *loading* one is what lets a weaker model then execute safely.
```
