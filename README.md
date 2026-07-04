# claude-skills

Personal/shared Claude Code skills.

## Skills

### `bug-triage`
Diagnose a reported bug from the evidence you have (logs, a ticket, a crash report) when
you may not be able to reproduce it locally, then hand the assigned developer a fact-based
head start. The deliverable is a **diagnosis + a developer-facing comment**, not a merged
fix. It separates verified fact from inference, proposes a fix only when the evidence
supports it, and always stops for approval before posting or committing.

Distinct from a reproduce-and-fix debugging loop: this is for field bugs, device/embedded
bugs, and timing/race bugs handed off via tickets.

## Using a skill

Claude Code discovers skills from `~/.claude/skills/` and from a project's
`.claude/skills/`. To use a skill from this repo, make it visible to Claude Code, e.g.:

```bash
# symlink an individual skill into your personal skills dir
ln -s "$PWD/bug-triage" ~/.claude/skills/bug-triage

# or symlink into a specific project
ln -s "$PWD/bug-triage" /path/to/project/.claude/skills/bug-triage
```

Then invoke it with `/bug-triage` (or let Claude pick it up from the description).

## Layout

```
claude-skills/
├── README.md
└── bug-triage/
    └── SKILL.md
```
# codex-working-rules
