# Reviewloop reference

## Full review checklist

Run every group on every pass. A pass that skips a group is not a pass.

### 1. Acceptance
- [ ] Each acceptance item maps to concrete lines in the diff (name the file:line)
- [ ] Nothing in the list was silently dropped, narrowed, or deferred
- [ ] Nothing outside the list was changed (scope creep is a finding)
- [ ] Behaviour the user relies on today still works — the backward-compatible path

### 2. Correctness
- [ ] Empty / null / zero / single-element inputs
- [ ] Error and rejection paths, not just the happy path
- [ ] Off-by-one and boundary conditions on every index and slice
- [ ] Async: ordering, races, unawaited promises, effects firing twice
- [ ] State: stale closures, mutations of shared objects, cache invalidation
- [ ] Resource cleanup: listeners, timers, file handles, subscriptions

### 3. Integration
- [ ] Every caller of a changed signature was updated
- [ ] Types, schemas, migrations and generated artifacts regenerated
- [ ] Config, env vars and docs that reference changed names
- [ ] Tests that covered the old behaviour still assert something meaningful

### 4. Hygiene
- [ ] No `console.log` / `print` / `dbg!` debugging left behind
- [ ] No commented-out code blocks
- [ ] No scratch, temp or preview files committed to the repo
- [ ] No secrets, tokens or personal paths in the diff
- [ ] Style matches the surrounding code (naming, comment density, idiom)

### 5. Verification
- [ ] Build / typecheck ran, output captured, exit code 0
- [ ] Test suite ran; new behaviour has a test that fails without the change
- [ ] Lint / format ran if the repo has them
- [ ] For UI work: the screen was actually rendered or screenshotted, not assumed

## Subagent prompt template

One per iteration. Give it the contract and the scope, never your justification.

```
You are reviewing someone else's change with fresh eyes. Do not assume it is correct.

The change was supposed to deliver:
<acceptance list, one line per item>

Scope:
  git status --short
  git diff <and/or the base...HEAD range>
  Untracked files (read these in full): <paths>

Report a findings list. For each finding: file:line, one sentence on the defect, and
a concrete failure scenario (inputs -> wrong result). Rank most severe first.
Report an empty list only if you found nothing — do not pad, do not soften.
Do not fix anything; report only.
```

## Iteration bookkeeping

For loops longer than two iterations, or work that may survive a context compaction,
keep a scratch file (scratchpad dir, or `meta/reviewloop.md` if the repo has one):

```md
# Reviewloop: <feature>

## Acceptance
1. ...
2. ...

## Iteration 1 — 3 findings
- [x] api.ts:42 — token refresh not awaited → fixed in <commit/edit>
- [x] Panel.tsx:88 — empty list renders "undefined" → fixed
- [x] no test for the expired-token path → added

## Iteration 2 — 1 finding
- [x] Panel.test.tsx:12 — new test passes even with the fix reverted → strengthened

## Iteration 3 — 0 findings. Done.
Verification: `npm run build` ✓, `npm test` ✓ (48 passed)
```

## Final report template

```
Reviewloop converged after N iterations.

Fixed along the way:
- <finding> → <what changed>
- ...

Final pass: 0 findings.
Verification: <commands run, results>
Not covered: <anything the loop could not check, e.g. no integration env>
```

## Worked example

User: *"reviewloop — I added a `--dry-run` flag to the deploy script"*

1. **Contract**: (a) `--dry-run` prints planned actions, (b) performs no writes,
   (c) exits 0, (d) existing default behaviour unchanged.
2. **Scope**: `git diff` shows `deploy.sh`; `git status --porcelain` shows an untracked
   `deploy_dryrun_test.sh` — read it, it's a scratch file that should not be committed.
3. **Pass 1** → 3 findings: `rm -rf` still runs under dry-run (b violated); exit code is
   1 when the plan is empty (c); the scratch file is untracked clutter.
4. **Fix** all three, re-run `shellcheck` and the script both ways.
5. **Pass 2** → 1 finding: the dry-run guard is checked after `mkdir -p`, so it still
   creates the target directory.
6. **Fix**, **Pass 3** → 0 findings. Report: converged after 3 iterations.

## Relationship to other skills

- **review-chain** — same shape, capped at 2 rounds. Use it when a bounded check is
  enough; use reviewloop when the user wants convergence.
- **/code-review** — a review engine, not a loop. Reviewloop can call it as step 2.
- **tdd** — write-test-first for building. Reviewloop verifies what already exists.
