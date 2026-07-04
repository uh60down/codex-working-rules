# Codex Working Rules

This repo carries shared Codex setup for local and online/cloud use.

## Evidence First

- Base factual claims on raw evidence: log line with timestamp, `file:line`, `file:symbol`, command output, data value, PR/ticket artifact, or cited source.
- Separate `Facts` and `Inference` in analytical work.
- Do not present correlation as proof; say `correlated, not proven` when timing or adjacency is the basis.
- If evidence proves a symptom but not the mechanism, say that and name the missing datum.
- Negative search is not negative fact. Vary the query and report what was searched before concluding absence.
- `Not found` and `I don't know` are valid results.

## Required Flows

- Ticket, log, crash, field bug, or "is this our bug" -> use `$bug-triage`.
- Any analytical deliverable before presentation -> use `$fact-check`.
- Feature-domain onboarding, spec/code disagreement, or state lifecycle confusion -> use `$domain-ontology`.
- Use `$working-constitution` when a task involves claims, estimates, external-facing output, or side effects and no narrower skill fully covers it.

## Side Effects

- Do not post to Jira/GitHub, comment, commit, push, open a PR, or send external output without explicit in-the-moment user approval.
- Investigation tasks are read-only unless the user explicitly requests code edits.
- Use draft -> present -> wait for explicit approval -> act.
- Jira comments must start with `AI analysis`.
- Never put credentials in command strings; read them from config or environment at runtime.

## Communication

- Lead with what was found or changed.
- Keep developer-facing summaries evidence-first, conclusions second, and inferences labeled.
- Prefer concise outputs with specific references over exhaustive code maps.
