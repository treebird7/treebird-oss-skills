---
name: clodex
description: Two-model adversarial review under a negotiated contract — Claude and Codex each review, each verifies the other's claims, then fixes split by disjoint file set and land in one build. Use where being wrong is expensive and green CI proves little — crypto, auth, migrations, data-loss paths, backup/exfil surfaces. NOT ordinary PR review (use /code-review), not typo fixes. Costs ~3 dispatches plus a high-effort pass.
---

# /clodex — contract review

Two reviewers, different priors, each able to overrule the other, splitting the fix
work under a contract they negotiate first.

**The rule everything else serves:** *the reviewer that questions your premises beats
the reviewer that checks your logic.* Same method twice and the second reviewer is
wasted spend. Keeping them apart is the orchestrator's job.

## Don't run this for

- Ordinary PR review → `/code-review`.
- Typos, renames, mechanical refactors, doc edits.
- Confirmation. This manufactures disagreement on purpose.

Cost ≈ 3 Codex dispatches + your own high-effort pass. Earn it.

## Phase 0 — review it yourself first

Never dispatch against a change you haven't read. You need findings to *trade*, and
context to judge its claims.

Verify at source, not memory:

- Library behaviour → decompile it. `javap -p -c` on the cached jar, or read the
  pinned tag. "The docs say" is not verification.
- Platform behaviour → vendor docs, fetched now.
- Pinned constant, key derivation, magic string → mutate it, confirm the test goes
  red. A check that cannot fail proves nothing.

## Phase 1 — dispatch the second reviewer, aimed away from your method

Numbered, specific hunting grounds — never "review this". Bias toward premises,
environment, lifecycle: where a logic-checking pass structurally cannot reach.

- What does this *assume* that nobody re-checked? (manifest flags, build config,
  platform defaults, "we already handle that")
- What escapes the happy path? Interrupted mid-operation, process death between two
  writes, concurrent callers, the failure branch of the failure branch.
- What crosses the device / process / trust boundary, by routes *other* than the
  obvious one?
- What does the new code log, serialize, or persist that it did not before?

Aim it away from your *method*, never your *findings* — withhold nothing you found;
the full trade happens before negotiation.

If you send a known-issues list, scope each item by **mechanism and exact bounds**,
never by topic, and append: *"if something resembles a known item but the mechanism
or outcome differs, report it anyway."* A topic-shaped suppression ("TOFU protects
nothing before first enrolment") once absorbed a genuinely different concurrent-
enrolment race as a non-finding — it came back reading *checked and clear*. Worse
than not asking.

Demand in the reply:

| | |
|---|---|
| `file:line` | Not paraphrase. Citations get opened and checked. |
| Severity + failure scenario | Concrete inputs/state → wrong behaviour. |
| The fix | Scoped, not a redesign. |
| **Non-findings** | What it checked and found clear. Separates "clear" from "didn't look". |
| **Falsifier** | "What would show you're wrong?" Makes Phase 2 cheap and Phase 3 tractable. |

```bash
codex -a never exec --ignore-user-config --skip-git-repo-check \
  -m <model> --config model_reasoning_effort="high" \
  --sandbox read-only -C <worktree> "<prompt>" 2>/dev/null
```

Flag rationale: `/codex`. Model + effort in one `AskUserQuestion`; current slugs in
`/codex` → Models. Prefer a different model family than your own reasoning. Never
`codex-auto-review` — a guardian model, not a reviewer you dispatch to.

## Phase 2 — verify every claim before relaying one

**Where the value is.** Its findings are claims, not results. Treat them as you'd
treat your own.

- **Confirmed** — you checked. Say how.
- **Discounted** — you checked and it doesn't hold, *or* the finding is real but its
  fix is wrong. Say which.
- **Unproven** — not cheaply checkable. Say so, then decide whether the fix is cheap
  enough to take anyway. (A `Mutex` beats proving the race can't happen.)

Never relay unverified as fact. Never dismiss a finding because it contradicts your
earlier review. Both cost more than the check.

**Capitulation is not resolution — in either direction.** If it folds without
engaging your evidence, that's **Unproven**, not Discounted. If *you* start agreeing
without new evidence, stop and re-read the source. Verdicts move when the evidence
moves, not when the other side sounds certain.

**Non-findings are claims too.** Each must state scope, method, and the paths it
examined; sample-check them like findings. "Checked and clear" without a method is
"didn't look" wearing a verdict.

**Zero findings is a signal, not a pass.** Say so out loud, and treat it as evidence
you may have reviewed the summary instead of the code.

## Phase 3 — negotiate the contract. Do not dispatch it.

**First, the reciprocal ledger.** Send it your own findings and require it to classify
each with the same three verdicts, with evidence — exactly what Phase 2 did to its
claims. The frontmatter's "each verifies the other's claims" *is* this step; skip it
and the protocol is one-directional review with extra ceremony.

Then send scope-downs *and* disagreements. Demand objections:

> "Push back where you disagree — I want your objections, not agreement.
> Answer with: (a) accept or counter-proposal, (b) objections to my scope-downs,
> (c) anything in MY half you think I've mis-scoped."

Not ceremony. In the origin run the second reviewer overruled three orchestrator
decisions and was right on all three; two would have shipped silently broken.
**Assigning work instead of negotiating it throws away most of the benefit.**

Concede when it wins the argument — verify first, then concede plainly and move on.

**Cap: two exchanges.** Past that you're converging on agreement, not truth. Take the
remaining disagreement to the human with both cases stated at their strongest.

Contract must state:

| | |
|---|---|
| **File sets** | Disjoint. Both parties write the same worktree in parallel; overlap means collision. |
| **Scope per finding** | Named constraints, not a finding number. "Do X inside function Y, no wrapper, no caller changes" — otherwise you get 15 interface methods where 2 lines would do. |
| **Hands off** | Explicit files the other party must not touch. |
| **Tests** | Add only — a model-neutral invariant; any writer can drop cases by mistake. A genuinely invalid existing test gets flagged to the orchestrator, not rewritten. Add-only does not stop semantic disabling via fixtures or config — mutation-check those too. |
| **Builds** | Exactly one party builds. Two daemons in one worktree contend on locks. |
| **Commits** | Orchestrator commits both halves. Keeps history and attribution coherent. |

Then dispatch with `--sandbox workspace-write` and do your half concurrently.

## Phase 4 — land it

1. **Wait for the writer to exit, then compare content hashes — not `git status` text.**
   Two `git status` runs do not prove stability: a file already listed ` M src/auth.ts`
   that a late writer changes again is still ` M src/auth.ts`. Identical output, different
   bytes. Verified 2026-08-03 — same porcelain across a modification, different `shasum`.
   An exited process cannot flush more writes, so waiting is the real guarantee; hash the
   owned files as the backstop for anything it orphaned.
2. **Report deviations — before the build, before the commit.** Departed from the
   contract? Say so now and let it object, one bounded round. Discovering a broken
   contract *after* committing leaves no gate and no rollback rule. In the origin run
   the migration path was made to log rather than throw, against contract, because
   the caller's startup coroutine was unguarded — the second reviewer agreed, on
   sharper reasoning than the original.
3. **One build for both halves.**
4. **Mutation-check its tests too**, not just your own. Break what the test claims to
   guard, confirm red, restore — **and hash the file after the restore.** Per step 1,
   `git status` cannot see an unrestored constant; only the hash proves the mutation
   actually came back out.
5. **Name what no test covers** — at the line, with what breaks if it is deleted. A
   gate that cannot fail must not read as coverage.
6. **One commit per file set**, so attribution is honest. Credit the second reviewer
   in the body for what it found.

## Honest limits

- **Small sample.** Encoded from runs where it found real things. A bet that differing
  priors beat redundant ones, not a proven method.
- **It produces disagreement you must adjudicate.** Feature and cost. Not prepared to
  verify claims and be overruled? Run `/code-review`.
- **The second model isn't smarter. It's differently blind.** Everything here rests on
  that and nothing else.
