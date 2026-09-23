---
name: reviewloop
description: Verify a just-implemented feature or bugfix against the real git diff, fix every finding, then review again from scratch — repeating until one full pass reports zero findings. Use when the user says "reviewloop", asks to double-check or harden work that was just written, wants "review until it's clean", or finishes a feature/bugfix and wants it verified before shipping. Unlike review-chain, which caps at 2 rounds, this loops to convergence.
argument-hint: "[what was asked for — feature description, bug report, or branch/PR]"
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Agent, Skill, TodoWrite
user-invocable: true
---

# Reviewloop

**review → fix → review again**, until a pass finds nothing. The loop exits on a clean
pass only — never on "close enough" or "the remaining ones are minor".

## Quick start

```
/reviewloop add a toggle for the new UI, keep the old one working
```

1. Pin the acceptance list (step 0)
2. Scope the diff (step 1)
3. Review pass → findings? (step 2)
4. Findings → fix them all → **back to step 3 with a fresh pass**
5. Zero findings → report iterations and stop

## Step 0 — Pin the contract

Write the acceptance list before looking at any code: one line per thing the user
asked for, phrased so it can be checked as true or false. Source it from the skill
arguments, the conversation, or the issue/PR body. If the request is too vague to
turn into checkable lines, ask the user now — not after three passes.

Every item is re-checked on **every** pass. Bugs are found by comparing the diff to
this list, not by re-reading your own summary of what you did.

## Step 1 — Scope the diff

```bash
git status --short          # what moved, including untracked
git diff                    # unstaged
git diff --staged           # staged
```

Tree clean? Review the branch instead:

```bash
base=$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main)
git diff "$base"...HEAD
```

**Untracked files never show up in `git diff`.** List them with
`git status --porcelain | grep '^??'` and read each one in full. Skipping new files is
the most common way this loop declares success on broken code.

## Step 2 — Review pass (the gate)

Prefer fresh eyes: spawn one subagent per pass with the acceptance list and the diff
scope, and none of your reasoning about why the code is right. One subagent per
iteration — no fan-out. If `/code-review` is installed, it is a fine engine for the
correctness sweep; the acceptance-list and leftovers checks are still yours.

A pass returns a findings list, possibly empty. Check, at minimum:

- **Implemented**: every acceptance item is actually in the diff, not just claimed
- **Correct**: edge cases, error paths, empty/null, off-by-one, async ordering
- **Whole**: callers, types, tests, docs and config that the change made stale
- **Clean**: no debug prints, commented-out code, scratch files, stray TODOs
- **Verified**: build / typecheck / tests / lint were *run*, with output, and are green

See [REFERENCE.md](REFERENCE.md) for the full checklist, the subagent prompt, and the
report template.

## Step 3 — Decide

- **Zero findings** → done. Report iteration count, what was fixed along the way, and
  the verification output from the final pass.
- **One or more findings** → fix all of them, then return to step 2 with a **new** pass
  over the updated diff. Do not spot-check only the lines you touched; a fix routinely
  breaks something adjacent.

## Guards

- **Cap at 5 iterations.** At the cap, stop and hand the user the open findings with
  what was tried. Looping forever is worse than reporting honestly.
- **Escalate on oscillation.** A finding that reappears after being fixed, or two
  findings that trade places, means the spec is ambiguous — ask instead of iterating.
- **Never launder a finding.** Deleting an assertion, widening a type to `any`,
  catching-and-ignoring, or marking a test skipped to make a finding go away is itself
  a finding.
- **Unrunnable verification is a finding.** If the build or test command can't run, say
  so in the findings; do not pass the gate on "it looks right".
- **No scope creep.** Changes outside the acceptance list are a finding, not a bonus.
