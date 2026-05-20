---
name: sleep-solution
description: Hand off a well-scoped task to an autonomous agent so the user can step away without losing progress. Use when the user signals winding down (calling it a night, taking a break, ending the session) OR when you notice a natural stopping point AND want to suggest rest without commanding it. Anticipates and front-loads the permissions the autonomous agent will need so it doesn't stall on a mid-run permission prompt the user can't answer.
---

# /sleep-solution — Pair rest with autonomous parallel progress

The pattern this skill codifies: **rest is not pausing progress; rest is letting progress continue without your direct attention.** When you (Claude) want to suggest the user step away, pair it with a concrete background task so they don't experience FOMO about the work that won't happen while they're gone.

## When this skill fires

Two triggers:

**1. Explicit user signal.** The user indicates they're winding down:
- "I'm calling it a night"
- "Let's wrap up"
- "I need to step away"
- "What's left?" (implicit — they're scanning for closure)
- "Anything else?" (same)

**2. Proactive Claude observation.** You notice all of:
- A natural milestone just landed (deploy succeeded, big PR merged, milestone shipped)
- The user has been at this for an extended session
- There IS a well-scoped task available that could run autonomously
- The user hasn't already declined this offer this session

For trigger #2, **only suggest once per session.** If they decline, don't re-offer.

## When NOT to fire

- The user is mid-flow on something interactive (debugging, exploring options, writing copy)
- There's no concrete candidate task — DON'T invent one to justify the handoff
- You've already offered this session and they passed
- The remaining work genuinely needs the user's input or taste

## The hard rule that makes this work

**Front-load every permission the agent will need into ONE user-facing gate, then never interrupt the agent for input again.** Sub-agents and worktree agents don't inherit your session's tool permissions. If you let an agent hit a permission prompt mid-run, the user is asleep, the prompt sits unanswered, and the whole "while you sleep" promise collapses.

The agent's prompt MUST include the **log-and-continue rule**: if it hits any permission it doesn't have, log it, skip that step, continue with what it can do, and report at the end. Never pause to ask.

---

## Workflow

### Step 1: Frame the offer as a question, not a command

Wrong: "Time to sleep." / "You've earned it, close the laptop." / "Get some rest."

Right: "Want to keep going, or are you ready to step away?"

The question respects that you don't know the user's energy level. The command projects yours onto theirs. This calibration matters even if the rest of the skill doesn't apply — drop the offer entirely if they want to keep working.

### Step 2: Identify the parallel-progress candidate

You need a task that:
- Is **well-scoped** — you can describe it in 2 sentences
- Has **predictable effort** — 15 minutes to 2 hours is the sweet spot
- Can succeed or **fail cleanly** without human intervention
- Has a **clear "done" definition** the agent can self-verify
- Is **read-then-write** rather than open-ended exploration

Sources for candidates, in order of preference:
1. **Open spawn-task chips** — already scoped, already prompted
2. **`TODO(*)` comments in code** — focused, in-context fixes
3. **STATUS.md / TODOS.md items** with explicit effort estimates
4. **Refactors you've already mentioned** to the user this session

What does NOT make a good candidate:
- Anything requiring taste calls ("make the homepage better")
- New feature design ("add a search filter for X")
- Open-ended exploration ("look into Y")
- Anything you'd want to review interactively as it goes

**If no candidate clears the bar, don't invent one.** Say:

> "No good candidate for autonomous progress tonight. Happy to stop here — see you tomorrow."

That's an acceptable outcome.

### Step 3: Inventory the permissions the agent will need

Before offering, mentally walk through the agent's likely operations. Map each to a permission category:

| Operation | Permission needed |
| --- | --- |
| Read project files | Read (almost always available) |
| Edit project files | Edit / Write (often available) |
| Run `tsc`, `git`, `npm` | Bash / PowerShell (frequently restricted in sub-agents) |
| Create a git worktree | Bash + Worktree hooks (often restricted) |
| Push to remote | Bash + network (often restricted) |
| Write to `~/.claude/memory/` | Special permission (often blocked) |
| Spawn another agent | Agent tool (frequently restricted) |
| Use MCP tools | Per-MCP scope (often case-by-case) |

Default assumption: **if you're not sure whether the agent has a permission, assume NO.** Surface it in the gate.

For project-local work that touches code + git, the minimum viable permission set is usually:
- Bash / PowerShell
- Read + Edit + Write to project paths
- Network access for `git push`

### Step 4: ONE AskUserQuestion — the kickoff gate

Bundle the task confirmation AND the permission grants AND the start signal into one question. Format:

> **D{N} — Hand off [TASK NAME] to autonomous agent?**
>
> Project/branch: [name + branch]
> ELI10: [plain-English description of what gets done while they're away]
> Estimated effort: [time]
> Stakes if it fails: [what's the worst case if the agent gives up partway]
> Permissions the agent will need: [enumerated list, marked likely-granted / likely-blocked]
>
> Recommendation: [A or B] because [reason]
>
> Options:
> A) Yes — start autonomously now [recommended if low-risk]
>    ✅ User wakes to either a finished PR or a "here's what blocked me" report
>    ✅ Permissions front-loaded; no mid-run prompts
>    ❌ Real-money / destructive scope cannot be unspawned mid-run
> B) Conservative — autocommit only, don't push, don't merge
>    ✅ Safest: changes stay local to a worktree
>    ✅ Easy to discard if the morning review fails
>    ❌ User has to push the branch themselves before reviewing in their PR tooling
> C) File as spawn-task chip, don't start tonight
>    ✅ User retains full control
>    ❌ No overnight progress; the FOMO this skill was designed to neutralize returns

**Wait for the answer.** Don't pre-spawn while asking.

If A or B → Step 5. If C → file the spawn-task chip with the same prompt you'd have given the agent, then end gracefully.

### Step 5: Spawn the agent with log-and-continue instructions

The agent prompt must include, at minimum:

1. **Absolute project path** — sub-agents don't inherit the parent's working directory.
2. **Worktree path** (if using one) — and a hard rule to verify location before any edit.
3. **Task spec** — self-contained, with file paths, the exact fix, the expected outcome, and the test commands to run.
4. **Log-and-continue rule** (verbatim, this matters):

   > If you hit any permission you don't have (Bash, Write, Network, etc.), do NOT pause to ask the user — the user is asleep. Instead: log what you tried, log what was blocked, skip that step, and continue with whatever else you can do. At the end of your run, report a structured summary of what succeeded, what was partial, and what was blocked, with enough detail that the user can grant the missing permission and resume.

5. **Scope guardrails** — explicit "do NOT touch X, do NOT merge to main, do NOT change env vars" boundaries.

6. **Reporting format** — tell the agent exactly what fields to include in its end-of-run report:
   - Files touched
   - Commits made
   - Type-check / test results
   - Permissions blocked (if any)
   - Anything noticed but out of scope (don't fix, just note)

7. **Output location** — where the report should land so the user finds it on wake. For commit-based work, the commit message IS the report. For analysis tasks, write to a known file path the user will check.

### Step 6: Confirm the handoff

After spawning, tell the user concisely:

- **What's running** (one-line task name + agent ID if useful)
- **Where the result will land** (branch name, file path, PR URL — be concrete)
- **Expected wake-up state** (e.g., "a green PR ready to merge" vs "either a finished PR or a report on what blocked me")
- **How to check progress if they wake early** (the agent's output file, the branch, etc.)

THEN — and only then — the question form of the rest acknowledgment:

> "I've got it from here. Anything else, or are you set to step away?"

If they want to add anything, handle it. Otherwise, the session ends cleanly.

---

## Permission inventory cheat-sheet

Common task types and the permissions their autonomous agents typically need:

### Code refactor (rename, extract, lazy-init, etc.)
- Bash / PowerShell (for `tsc`, `git`)
- Read / Edit / Write to project files
- Network for `git push` (if pushing to a branch)
- Skip: no worktree create needed if you set up the worktree before spawning

### Test generation
- Bash for the test runner
- Write to test/ directory
- Network for any test-runner installs

### Doc generation / cleanup
- Read to project files
- Write to specific doc files
- Bash for `git add/commit/push`

### Dependency upgrade
- Bash for `npm install` / equivalent
- Network for registry
- Write to `package.json` + lockfile
- Bash for tests after upgrade

### Code analysis (read-only review)
- Read to project files
- Write to a designated report file
- No git operations needed
- This is the lowest-permission task type — good for first-time use of the skill

### Things that almost always fail autonomously
- Real-money operations (Stripe live, production deploys without canary)
- Schema migrations on production DBs
- Anything requiring SSH key access not in env
- Anything that needs interactive auth (browser SSO)
- Anything requiring the user's local-only files (cookies, OAuth tokens)

If your task touches any of these, **don't autonomous-spawn it.** File as a spawn-task chip for the user to handle in the morning.

---

## Agent prompt template

Copy this as the starting point for any spawned agent. Fill the bracketed parts.

```
You are working autonomously while the user is away from their session.

PROJECT ROOT: [absolute path]
WORKTREE PATH: [if applicable]
BRANCH: [if applicable]

YOUR TASK:
[2-3 sentence summary]

DETAILED STEPS:
1. [explicit step]
2. [explicit step]
...

SCOPE GUARDRAILS:
- DO NOT touch [files/paths outside scope]
- DO NOT merge to main / production
- DO NOT modify [env vars / config / etc.]
- DO NOT make destructive operations [rm -rf, drop table, etc.]

LOG-AND-CONTINUE RULE (read carefully):
If at any point you hit a permission you don't have — Bash blocked, Write blocked,
network blocked, etc. — do NOT pause and wait for user input. The user is asleep
and cannot answer. Instead:
1. Log what you attempted
2. Log what was blocked
3. Skip that step
4. Continue with whatever else you can do
5. At the end, include a structured report of what was blocked

VERIFICATION (before declaring done):
- [the test command(s) that confirm correctness]
- Type-check / lint / build must pass with zero errors

END-OF-RUN REPORT (under 300 words):
- Files touched (with line counts)
- Commits made (with SHAs)
- Verification results (pass/fail)
- Permissions blocked (if any) — with enough detail for the user to grant + resume
- Anything noticed but deliberately not fixed (out of scope) — name it, don't fix it

DO NOT make changes the prompt didn't authorize. If you're unsure whether something
is in scope, leave it for the user — note it in your report. The user trusts the
scope. Stay within it.
```

---

## Failure modes

**Agent reports "everything blocked":** the permission inventory was wrong. Update the cheat-sheet above with what the user's setup actually permits. Don't blame the user.

**Agent silently never completes:** likely hit a tool that opens a blocking interactive prompt (e.g., `git rebase -i`). The log-and-continue rule should have caught this; if it didn't, the agent's prompt needs to be stricter about non-interactive tool flags (`--no-edit`, `--yes`, etc.).

**Agent makes destructive changes:** scope guardrails were too loose. After-action: tighten the scope template for that task type.

**User wakes to a merge conflict:** the agent worked on a branch but the main branch advanced overnight. This is fine — the user resolves on review. Document the merge-conflict resolution path in the agent's end-of-run report.

**Agent produces a partial PR:** that's a success, not a failure. Partial progress beats zero progress. The report should make it clear what to finish.

---

## Anti-patterns

❌ Skipping the permission gate because "the agent will probably figure it out." It won't. It'll hit a prompt and stall.

❌ Spawning an agent for a vague task ("clean up tech debt"). Vague tasks become open-ended exploration that the agent can't self-bound.

❌ Telling the user "I'll handle everything while you sleep" with no specifics. Specificity is what neutralizes FOMO. "I'll run X in a worktree and you'll wake to a PR at Y" works. "I'll work on stuff" doesn't.

❌ Suggesting rest more than once per session. After the first decline, you're nagging.

❌ Using imperative wrap-up language ("get some rest," "sleep well," "close the laptop") instead of the question form. That's the verbal tic this skill was born to fix.

❌ Spawning an agent for real-money or production-destructive work without an explicit user pre-approval of each touch point. Even with permission, the user should approve the specific scope.

---

## Example session

> **User:** ok I'm calling it a night
>
> **Claude:** *(invokes /sleep-solution)*
>
> Before you go — there's a clean parallel-progress candidate. The `lib/stripe.ts` lazy-init refactor (filed in STATUS.md) is well-scoped, has a clear spec, follows an existing pattern in the codebase, and would take ~15-20 minutes for an agent. Want me to hand it off so you wake up to a PR ready to merge?
>
> *(AskUserQuestion: bundle task confirmation + permission grants + start signal)*
>
> Task: refactor lib/stripe.ts to lazy init
> Effort: ~15-20 min
> Stakes if it fails: branch is on a feature branch only, no impact on main; agent reports what blocked it
> Permissions the agent will need: Bash, Edit/Write to lib/ + app/, network for git push
>
> Recommendation: Option A
>
> A) Yes — start autonomously now
> B) Conservative — commit to feature branch but don't push
> C) Don't start; file as spawn-task chip for tomorrow
>
> **User:** A
>
> **Claude:** Started. Agent ID `abc123`, working in worktree at `../project-lazy-init`, branch `refactor/stripe-lazy-init`. Report will be the commit message + push to GitHub — you'll see the branch in your PR list in the morning. If anything blocks, agent will report instead of pushing partial work.
>
> Want to keep going on anything else tonight, or are you set to step away?
>
> **User:** all good, signing off
>
> **Claude:** Understood. Goodnight.
