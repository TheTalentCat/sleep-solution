# /sleep-solution

A Claude Code skill that turns "time to wrap up" into "I'll have a PR for you in the morning."

## The problem this fixes

If you use Claude Code (or any agentic assistant) for long sessions, you've probably noticed two annoying patterns:

1. **The assistant tells you to sleep instead of asking.** "Time for some rest." "Close the laptop, you've earned it." It's well-meaning, but it's projecting an energy state onto you that the assistant has no way to actually know.

2. **You don't want to stop because there's still momentum.** Each thing you finish reveals another thing you could finish. Stopping feels like wasting the runway.

This skill addresses both:

- It reframes the rest suggestion as a **question** ("ready to step away, or want to keep going?"), not a command.
- It pairs the suggestion with a **concrete autonomous task** so stepping away no longer means stopping progress. You wake up to a finished PR or a clear "here's what blocked me" report.

## The non-obvious insight

Most "autonomous overnight work" attempts fail the same way: the agent hits a permission prompt three minutes in, you're asleep, the prompt sits unanswered, and the whole promise collapses.

The skill's core mechanic is **front-loading every permission decision into one user-facing gate before spawning**, then instructing the agent to **log-and-continue** rather than ever pause to ask. The user is asleep. The agent cannot wait for input. Bake that into the agent's prompt as a hard rule and the autonomous run actually completes (or fails cleanly with a structured report).

## What's in here

- [`SKILL.md`](./SKILL.md) — the full skill spec: when to fire, when not to, the permission-inventory cheat-sheet, the spawned-agent prompt template, failure modes, anti-patterns, and an example session.

## Installation

### Per-project

Drop `SKILL.md` into your project at:

```
your-project/.claude/skills/sleep-solution/SKILL.md
```

Commit it. Claude Code will discover it automatically. Invoke with `/sleep-solution`, or let Claude invoke it on its own when the trigger conditions hit.

### Global (across all your projects)

```
~/.claude/skills/sleep-solution/SKILL.md
```

(On Windows: `C:\Users\<you>\.claude\skills\sleep-solution\SKILL.md`.)

## When the skill fires

Two triggers:

1. **You signal you're winding down** — "calling it a night," "let's wrap up," "what's left?"
2. **Claude proactively notices** a natural stopping point AND has a well-scoped task that could run autonomously. (Capped at one suggestion per session — no nagging.)

If there's no good autonomous candidate, the skill explicitly tells Claude **not to invent one** just to justify the offer. "No good candidate tonight, see you tomorrow" is a valid outcome.

## What a good autonomous candidate looks like

- Well-scoped (describable in 2 sentences)
- 15 minutes to 2 hours of agent work
- Can succeed or fail cleanly without human input
- Has a self-verifiable "done" definition
- Read-then-write, not open-ended exploration

Bad candidates: anything requiring taste calls, new feature design, real-money operations, or production-destructive changes. These get filed as spawn-task chips for the user to handle later, not autonomous-spawned.

## Credit

This was extracted from a real session with [Claude Code](https://claude.com/claude-code). The user flagged the "sleep" verbal tic, proposed both the question-form recalibration AND the parallel-progress reframe, and explicitly identified the permission-prompt problem that motivated the log-and-continue rule.

The skill exists because they said: *"take this and share it with your brethren."*

## License

MIT. Use it, fork it, improve it, ship better versions.

## Contributing

PRs welcome. Especially:

- New entries for the **permission inventory cheat-sheet** (`SKILL.md` section). If you've run autonomous agents and hit permissions that aren't documented, add them.
- New entries for the **failure modes** section. The skill gets better as more people stress-test it.
- Better **agent prompt template** wording — the log-and-continue rule in particular benefits from real-world hardening.
