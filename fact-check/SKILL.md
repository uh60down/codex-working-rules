---
name: fact-check
description: >-
  Use before presenting ANY external-facing analytical deliverable built on factual
  claims — a Jira comment, a root-cause diagnosis, a report, a code-review verdict, a
  recommendation. Runs a two-pass verification: (a) self-audit of every claim against
  the raw sources, then (b) an independent adversarial subagent that returns
  SUPPORTED / OVERSTATED / UNSUPPORTED verdicts, which must be resolved by actual
  verification (grep/read/trace), never by adding hedge labels. Extracted from
  bug-triage step 9 so it can run on any deliverable, whether or not /bug-triage was
  invoked. Use for "fact-check this", "is this draft accurate", and by default before
  presenting analysis results. Also the mandatory gate before **publishing a domain
  ontology** (see the domain-ontology skill) — a reused world-model is a high-blast-radius
  deliverable: claim inventory = concepts (file:symbol), status codes, counts (recompute,
  don't read the number), the state-mapping table, and each Gap's two cited sides.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
---

# Fact-Check

Verify a draft deliverable claim-by-claim before it reaches the user (and, through
them, other people). The most expensive failure mode in this workspace is being
*confidently wrong*: a fluent, plausible-sounding claim reads identically to a verified
one in the output. This flow is the mechanism that separates them.

## The one rule

**A claim ships only with its evidence.** Every factual claim either gets a quotable
backing (raw log line, `file:line`, data value, command output) or gets downgraded to a
labeled inference — and if it was flagged by the checker, it gets *verified or cut*,
never merely relabeled.

## Procedure

### 1. Self-audit (author pass)

Re-verify **every** factual claim in the draft against the raw data it cites:
- Re-read the quoted log lines / code at the cited `file:line`. Do not trust your own
  earlier extraction — go back to the primary artifact.
- Catch overstatements: "always/never/constant" drawn from one sample; an inference
  phrased as fact; a semantic upgrade the evidence doesn't license (e.g. calling an
  unknown `warningCode` an "error").
- If a claim isn't backed by something you can quote, downgrade it to inference or cut
  it now — cheaper than having the subagent catch it.

### 2. Independent adversarial check (subagent pass)

Dispatch a separate agent to re-verify the draft — *you wrote it, so you are the worst
person to catch your own rationalizations.* Give it:
- the draft (or the enumerated list of factual claims),
- the raw artifact paths (logs, data files),
- the affected `file:symbol`s.

Instruct it to, for each claim, re-read the source and return a verdict:
- **SUPPORTED** — quote the exact backing line / code / value.
- **OVERSTATED** — true but claimed more strongly than the evidence (e.g. an inference
  presented as fact, or "always" drawn from one sample).
- **UNSUPPORTED** — no backing evidence, or the evidence contradicts it.

It must also flag: any reasoning-chain step left unchecked, and any referenced
`file:symbol` / PR / artifact it could not confirm exists. Tell it to **default to
refuting when uncertain — its job is to refute, not to agree.** Do not lead it toward
your conclusion; phrase claims neutrally ("report what the code actually shows").

Model note: run the checker on the **strongest available model**. This is the one
dispatch not to downgrade — finder/mapper agents can be haiku/sonnet, but the
adversarial pass is where multi-layer judgment pays.

Example dispatch:
```
Agent(subagent_type: "general-purpose", description: "Fact-check draft",
  model: "<strongest available>",
  prompt: "Adversarially fact-check this draft. For EACH factual claim, re-read the
  cited source (raw log at <path>, code at <file:symbol>) and return SUPPORTED (with
  the exact quote) / OVERSTATED / UNSUPPORTED. Default to refuting when uncertain.
  Flag any unchecked reasoning step and any file:symbol/PR you cannot confirm exists.
  \n\n<draft + evidence trail>")
```

### 3. Act on the verdicts — acting means verifying, not labeling

For every OVERSTATED or UNSUPPORTED verdict:
1. Go find the actual evidence (grep, read the file, trace the call, re-pull the log).
2. If evidence exists → upgrade to SUPPORTED and quote it.
3. If evidence does not exist → **cut the claim entirely**, or replace it with the
   actual finding.
4. **Never resolve a verdict by adding a label like `[Inference]` or `[unverified]` —
   that is deferral, not resolution. A false claim with a disclaimer is still a false
   claim.** (This exact failure happened: a flagged claim kept as `[Inference]` turned
   out to be entirely false — the code path never existed.)

Additional discipline:
- Re-run step 2 if the rewrite introduced new claims.
- Distinguish observations, symptoms, contributing factors, and root-cause claims —
  root-cause assertions require stronger evidence than symptom descriptions.
- Never upgrade confidence without upgrading evidence; stronger wording requires
  additional verification, not additional conviction.

### 4. Gate

Do not present a draft with unresolved OVERSTATED / UNSUPPORTED findings. If a claim
cannot be verified and cannot be cut (the user must decide), present it explicitly as
"blocked on <the missing datum>" — top of the deliverable, not buried after it.

## Anti-patterns

- Skipping the subagent pass because the self-audit "felt" clean — the two passes catch
  different errors; run both.
- Resolving a verdict with a hedge label instead of verification.
- Leading the checker ("confirm that X causes Y") instead of asking it to refute.
- Re-verifying from your own earlier notes/extraction instead of the raw artifact.
- Presenting a draft with unresolved verdicts.
- Treating a plausible mechanism as verified because it is coherent — plausibility
  reads identically to evidence in output; only citations distinguish them.
