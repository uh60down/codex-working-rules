---
name: bug-triage
description: >-
  Use when triaging or root-causing a bug from logs / a bug report you may not be
  able to reproduce locally — especially a field bug handed off via a ticket (Jira,
  GitHub issue). Produces a fact-based diagnosis and a developer-facing comment,
  separating verified fact from inference, and proposes a fix only when the evidence
  supports it. Always stops for approval before posting or committing. Use for
  "what's the root cause of this ticket / log", "is this our bug", "triage this
  crash report", "diagnose this from the logs". NOT a reproduce-and-fix-locally
  loop — for a bug you can reproduce and fix in one sitting, use a normal debug flow.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
---

# Bug Triage

Diagnose a reported bug from the evidence you have (logs, a ticket, a crash report),
hand the assigned developer a fact-based head start, and stop. The deliverable is a
**diagnosis + a developer-facing comment**, not a merged fix.

## Reliability rules
- Never state specifics (names, versions, flags, paths, values, behaviors) you cannot confirm from provided context. State the assumption explicitly instead.
- If something is uncertain, say so before proceeding — do not bury the caveat after the answer.
- Do not fill gaps with plausible-sounding details. An explicit "I don't know" is always better than a confident guess.
- Reference only what exists in: this SKILL.md, user-provided files/docs, or content retrieved in the current session.
- If a claim depends on an assumption, label it: "Assuming X — please verify."

## When this fits vs. when it doesn't

- **Fits:** field bugs you can't reproduce on your machine, device/firmware/embedded
  bugs, timing/race bugs, anything where the artifact is a log + a ticket and the next
  step is "tell the right person what's actually wrong."
- **Doesn't fit:** a bug you can reproduce and fix in one sitting. Use a normal
  debug-and-fix loop for that.

## The one rule

**Base every claim on verified fact, show the raw evidence, and label every inference
as an inference.** A confident-but-wrong conclusion costs the next person more time than
no conclusion. Do not trust the ticket's stated root cause — verify it.

## Don't manufacture a plausible cause (read before you theorize)

The most common way this goes wrong is being *confidently wrong*: producing a fluent,
plausible-sounding mechanism instead of the true one, because a large codebase always
offers enough surface to build a plausible story. Plausibility is **not** evidence, and
it reads identically to evidence in your output — so a guess gets dressed up as a
finding. Four guardrails, in priority order:

1. **Get the discriminating datum first, or declare blocked on it.** For "*some* X is
   missing / failing Y" bugs, the very first question is: *what distinguishes the X that
   work from the X that don't?* (record `type`/`source`, build, code path, device, time
   window). Until you have that fact, every path theory is a guess. The correct output is
   "I need an example of each kind to tell which path this is" — **not** a ranked menu of
   possible mechanisms.
2. **Trace backward from the failing artifact, not forward from the architecture.** Start
   at the actual broken record/event/log line and walk *upstream* to its emitter.
   Forward enumeration ("what could cause this?") invites invention; backward tracing from
   a real artifact constrains you to reality.
3. **Complete the chain to the sink before concluding** (see step 4). A single true-looking
   fact ("this map doesn't add the field") is not the conclusion — keep going and check
   whether something downstream adds or fixes it.
   - **Generalize to the rule; don't stop at the first matching path.** If one path is
     missing the field, ask *what is the general condition that produces this?* Usually
     it's a class ("any record that doesn't pass through decorator X"), not one path. A
     fix aimed at the single path you found patches one leak and leaves the rest.
4. **Label plausible vs verified in the output itself.** Any mechanism you haven't traced
   to evidence is a "candidate, not checked against real data" — say so. Never present a
   hypothesis with the confidence of a finding.

Blunt version: **when you don't have the fact that decides the answer, stop and name what
you're missing.** Do not fill the gap with the most plausible story.

## Judgment rules (apply before/while running the steps)

Codified from real triage runs — these are the calls the written steps don't make for you:

1. **Classify the ticket archetype first, and adapt.** Field bug with a log → full
   procedure. Story/design change sitting in QA → verify the implementation against the
   acceptance criteria; a "bug" mentioned in comments may be a separate report, not the
   ticket. Spec-tuning question → compare against the spec, don't hunt code defects.
   Possible duplicate → search first. Restate the triage question in one sentence for
   the archetype you picked.
2. **Negative search ≠ negative fact.** An empty `git log --grep=<ticket-id>` does not
   mean "not implemented" — PRs often omit the ticket key. Also search symptom keywords,
   the affected file paths, and PR titles; state which queries you ran before concluding
   absence.
3. **Establish the version discriminant early:** the fix's merge date vs. the build the
   reporter tested. This single fact decides whether a QA observation applies to the
   current code or predates it.
4. **"Not found" is a result.** If an artifact (PR number, log, code path) cannot be
   located, report exactly that with the search trail. Never reconstruct its contents.
5. **One analysis per reported symptom.** Enumerate the symptoms at intake; do not let a
   second symptom ride along unanalyzed.
6. **Batch mode: keep an accountability table** (ticket | fact-check status | presented/
   held | reason). Every ticket gets the full step 9 — rigor must not thin with volume;
   no sampling.
7. **Verify inputs too.** Ticket-supplied PR numbers, the reporter's stated repro steps,
   and the ticket's own "root cause" line are claims, not facts — check them like any
   other claim.

## Procedure (single ticket: one todo per phase; batch: one task per ticket plus the accountability table from Judgment rule 6)

### 1. Intake & restate
- Capture: ticket id, the log/artifact path, the build/version it reproduced on.
- Restate the symptom in **one factual sentence** (what was observed, not what's assumed).

### 2. Check for an existing fix FIRST
Before constructing any diagnosis — it may already be fixed.
```bash
gh pr list --search "<ticket-id>" --state all
git log --oneline --all --grep="<ticket-id>"
git log --oneline -10 -- <affected files/dirs>
```
If a merged fix exists, your job is mostly done: report it (PR + file:symbol) and stop.

### 3. Establish facts from raw data
- Read the **raw log**. Build a timeline of the relevant window with **timestamps**.
- Quote the exact lines you rely on. Read the actual code path (`Grep`/`Read`), not your
  memory of it.
- Convert timezones explicitly if the log and report differ.

### 4. Trace the WHOLE chain (for "event lost" / dropped-effect / missing-attribute bugs)
- **Trace backward from the failing artifact.** Start at the actual broken thing (the
  record missing the field, the dropped event, the symptom log line) and walk *upstream*
  to its emitter. Do not start from the architecture and reason forward about what could
  go wrong — that generates plausible guesses, not the real path.
- Map every hop from producer to final consumer (every async boundary, queue, stream
  operator, cancellation point, subscription, **and every alternate path to the same
  sink** — e.g. a value added on the log path but not the trace path).
- **Enumerate ALL drop points — do not stop at the layer that logged the symptom**, and
  do not stop at the first true-looking fact. "This layer doesn't add X" is not the
  conclusion until you confirm nothing downstream adds X either. The cause is often the
  layer *furthest* from the symptom line.
- For each hop ask: *what does it depend on, and can that be null / stale / absent under
  the failure scenario* — even if it succeeded in the one captured log? One log is one
  sample; the bug is the edge. Multiple independent causes can coexist.

### 5. Separate fact from inference
Write two explicit buckets:
- **Facts** — each backed by a quoted raw line / `file:symbol` / data value.
- **Inference** — what you deduce, clearly labeled, with how confident and why.
When the evidence proves a *symptom* but not the *mechanism*, say exactly that.

### 6. Race / no-repro ceiling
If it's a timing/race/cancellation bug you can't reproduce: recognize that a single log
usually proves the **symptom, not the mechanism** (silent cancellations, sub-ms windows,
unlogged triggers). Don't over-claim. To *prove* the mechanism, recommend the
disambiguator: targeted instrumentation at the suspect points, or an on-device trace
(e.g. Android Perfetto / `androidx.tracing`). Present candidate mechanisms as
branch-conditional, not a single confident pick.

### 7. Pick the exit
- **App-side, evidence-supported** → propose a **fact-grounded fix**: concrete change +
  `file:symbol`. Mark it "potential" if supported-but-unproven.
- **Off-app** (vehicle/ECU/firmware/SDK/third-party/infra) → do **not** invent an app
  fix. Package the evidence, name the data that would confirm it (e.g. "need the
  device/ECU-side log for <time>"), and recommend reassigning.
- **Inconclusive** → say so plainly and name exactly what data closes the gap.

### 8. Draft the developer-facing comment
Goal: save the assigned dev's initial-investigation time.
- Lead with **verified facts + the evidence trail** (raw lines, `file:symbol`).
- Conclusions secondary; inferences labeled.
- Include the fix / potential fix, or the data needed.
- Keep it scannable (~20-40 lines). A brief "credit the reporter / apologies for the
  earlier wrong root cause" line is fine; no groveling.

### 9. Fact-checking review (default — do not skip)
This step is also available as the standalone `/fact-check` skill — use that skill for
ANY analytical deliverable, not just triage. The full flow is kept inline here so this
skill stays self-contained when shared.

Two passes. The diagnosis does not reach the user until both are clean.

**9a. Self-audit.** Re-verify **every** factual claim in the draft against the raw data.
This catches overstatements (e.g. "value was constant" when it actually changed). If a
claim isn't backed by something you can quote, downgrade it to inference or cut it.

**9b. Independent fact-check (subagent).** Dispatch a separate agent to adversarially
re-verify the diagnosis — *you wrote it, so you are the worst person to catch your own
rationalizations.* Give it the draft, the raw log/artifact path, and the affected
`file:symbol`s. Instruct it to, for each factual claim, re-read the source and return a
verdict:
- **SUPPORTED** — quote the exact backing line / code / value.
- **OVERSTATED** — true but claimed more strongly than the evidence (e.g. an inference
  presented as fact, or "always" drawn from one sample).
- **UNSUPPORTED** — no backing evidence, or the evidence contradicts it.

It must also flag any drop point in the chain (step 4) left unchecked, and any proposed
fix whose `file:symbol` it could not confirm exists. Tell it to default to OVERSTATED /
UNSUPPORTED when uncertain — its job is to refute, not to agree.

Then **act on the verdicts** — and acting means verifying, not labeling:
- **OVERSTATED**: go find the actual evidence (grep, read the file, trace the call). If
  evidence exists, quote it and upgrade to SUPPORTED. If it does not exist, cut the claim
  or replace it with the actual finding. **Adding `[Inference]` or `[unverified]` to an
  OVERSTATED claim is not resolution — it is deferral. A false claim with a disclaimer is
  still a false claim.**
- **UNSUPPORTED**: same — verify the underlying fact first. Only after checking do you
  know whether to cut, rewrite, or promote.
- Re-run 9b if the rewrite introduced new claims.
- Do not present a draft with unresolved OVERSTATED / UNSUPPORTED findings.
- Distinguish observations, symptoms, contributing factors, and root-cause
  claims. Root-cause assertions require stronger evidence than symptom
  descriptions.
- Never upgrade confidence without upgrading evidence. Increasing certainty
  requires additional verification, not stronger wording.

Model routing (encode it — do not decide per call): code-mapping / "where is X" /
log-extraction subagents → `model: "haiku"` or `"sonnet"`; the adversarial fact-check
subagent → the strongest available model (the one dispatch NOT to downgrade).

Example dispatch:
```
Agent(subagent_type: "general-purpose", description: "Fact-check bug diagnosis",
  model: "<strongest available>",
  prompt: "Adversarially fact-check this bug diagnosis. For EACH factual claim,
  re-read the cited source (raw log at <path>, code at <file:symbol>) and return
  SUPPORTED (with the exact quote) / OVERSTATED / UNSUPPORTED. Default to refuting
  when uncertain. Flag unverified chain drop-points and any fix file:symbol you
  cannot confirm exists.\n\n<draft + evidence trail>")
```

### 10. STOP — present, do not post
Present the draft and the diagnosis. **Do not post to the ticket, comment, commit, or
push** without an explicit, in-the-moment go-ahead from the user.

## Anti-patterns
- **Producing a ranked menu of plausible mechanisms when you lack the fact that decides
  between them.** That's guessing dressed as analysis. Get the discriminating datum
  (which records/cases fail vs. don't) first, or say you're blocked on it.
- **Reasoning forward from the architecture ("what could cause this?") instead of
  backward from the actual failing artifact.** Forward enumeration invents causes.
- **Stopping at the first true-looking fact.** "This layer doesn't add the field" feels
  like a hit but isn't the conclusion until you've traced to the sink.
- **Naming one failing path as the root cause when it's a class.** If one path lacks the
  field, the cause is usually the general condition (e.g. "any record bypassing decorator
  X"), not that path. Fixing the single path you found leaves every sibling path broken.
- Proposing a fix before tracing the full data flow.
- Anchoring on the layer that logged the symptom.
- Over-anchoring on someone's phrasing (ticket, reviewer, lead) and jumping to the most
  salient symbol they named without re-deriving which code path actually matters.
- Treating "it worked in this one log" as "this link is sound."
- Stating an inferred mechanism as fact.
- Skipping the independent fact-check (9b) because the self-audit "felt" clean — the
  self-audit and the independent pass catch different errors; run both.
- **Resolving an OVERSTATED / UNSUPPORTED verdict by adding a hedge label (`[Inference]`,
  `[unverified]`) instead of actually verifying.** The subagent flags what's wrong; you
  must go find the truth. A labeled false claim is still a false claim.
- Presenting a draft with unresolved OVERSTATED / UNSUPPORTED findings.
- Posting/committing without explicit approval.

## Output

Two artifacts:
1. **Diagnosis** — Facts / Inference / Exit (fix proposal, off-app handoff, or
   data-needed), with raw evidence.
2. **Draft comment** — the dev-facing write-up, ready to post once approved.

Both must have passed the step 9 fact-checking review (self-audit + independent
subagent) with all OVERSTATED / UNSUPPORTED findings resolved before you present them.
