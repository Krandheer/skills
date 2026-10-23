---
name: review-pr
description: Review a GitHub pull request with every claim verified against the code before it is written down. Use when asked to review a PR by URL or number, in any language or repo. Produces severity-labelled, copy-pasteable comments anchored to file:line, each with an explanation and a concrete alternative. Separates defects this PR introduced from pre-existing ones, and reports the hypotheses it ruled out.
---

# Verified PR review

The goal is a review the author can act on without re-deriving anything, where
**nothing is asserted that wasn't checked**. A confident wrong comment costs more than a
missing one — it sends the author to rewrite working code.

## 1. Get the real PR, in isolation

```bash
gh pr view <N> --repo <owner>/<repo> --json title,body,author,state,baseRefName,headRefName,additions,deletions,changedFiles,commits
gh pr diff <N> --repo <owner>/<repo> --name-only
gh pr diff <N> --repo <owner>/<repo>
```

Then check out the PR head **by ref**, never by branch name:

```bash
git fetch origin refs/pull/<N>/head:pr-<N>-review
git worktree add <scratchpad>/pr<N> pr-<N>-review
git rev-parse pr-<N>-review    # MUST equal commits[-1].oid from gh pr view
```

- **A local branch with the PR's name is not the PR.** It is routinely stale or rebased.
  Verify the SHA, and re-verify if you leave and come back — someone may pull mid-review.
- **Never touch the user's working tree or branch.** Work only in the worktree. Symlink
  dependency dirs in rather than reinstalling (`node_modules`, `.dart_tool`, `vendor`, `.venv`).
- **`baseRefName` is often not the default branch** — stacked PRs are common. Diff against
  the real base: `git diff origin/<baseRefName>...HEAD`.
- **Note the size.** Past ~400 changed lines of non-mechanical logic, say so and suggest
  splitting. Judge by reviewable substance, not line count — a 600-line rename is fine.
- Check CI status before reading; a red build may answer the question for you.
- Clean up: remove symlinks, `git worktree remove --force`, `git branch -D pr-<N>-review`.
  Confirm with `git worktree list` and `git status --porcelain`.

## 2. Decide what to look for

Don't run a flat checklist — it produces noise and crowds out specific reasoning. Three
sources decide where to dig, in this order.

**The PR's own claims are the test suite.** Every assertion in the title, body, and
comments is something to check: "byte-identical", "no behaviour change", "dead code",
"eventually consistent", "same as before", "purely mechanical". Each is a finding if
false, and a positive worth stating if true. Highest yield, needs no domain knowledge.

**Change shape predicts the defect family.** Match the diff to its shape:

| Shape | Dig here |
|---|---|
| Retry / timeout / error handling added | The helper's real semantics (delay curve, attempt count, jitter). What happens at exhaustion. Is the retried work idempotent. Is the root cause fixable instead of masked. |
| Extraction / dedup refactor | Diff *every* constant and default, old vs new — silently added or dropped scaling/config is the classic miss. Does the abstraction flatten differences that mattered. |
| Dead-code removal | Prove unreachability from the language/framework's actual semantics (hit-test order, dispatch, overload resolution). Never accept the assertion. Then ask what dead code was left behind. |
| New shared abstraction | Parameter surface vs. call sites. Immutability / const-ness / allocation. Placement. Does one convention now contradict a sibling. |
| Async consumer (queue, event, webhook) | Producer ordering. Poison messages and the DLQ path. Replay and idempotency. Read-consistency of anything the producer just wrote. |
| Schema / data-shape change | Migration of existing rows. Readers that predate the change. Nullability at the boundary. |
| Perf / caching change | Invalidation. Staleness window. Behaviour when the cache is cold, full, or wrong. |
| Auth / input handling | Injection (SQL, command, template), authorization vs. authentication, IDOR, what reaches logs. |

Across all shapes, sweep these dimensions once: logic and edge cases, error handling,
concurrency, security, performance (N+1, complexity, leaks), test coverage, naming,
design and maintainability.

**Naming — always do this pass.** Read every identifier the PR introduces: variables, functions,
types, constants, files, response fields. Flag:
- **One name, two meanings.** The same name, or near-identical names, used for different domain
  objects in the same file. For example, `orderItems` holding customer order items in one function
  and warehouse order items in the next.
- **Names that differ by one character** for different things, such as `revertOrders` and
  `revertOrder`.
- **A name that hides behaviour.** `checkX` that returns data or throws, or `getX` that writes
  or validates.
- **Vague names where the domain has a precise word.** `data`, `result`, `reserved`, `list`.
- **Mismatches with the repo's existing vocabulary** for the same concept. Grep for how sibling
  code names it.

Report a naming issue only when it could mislead a reader. Give the rename as a diff.

**Design and maintainability — always do this pass.** Measure the dominant pattern before
judging conformance. Count the occurrences, don't guess. For example: how many sibling handlers
are class methods vs. free functions, and where do sibling schemas, types and constants live?
Then check:
- **Structure.** Does the new code follow that pattern (class vs. functions, file placement,
  registry wiring)? If it deviates, say whether a local precedent exists, and recommend the
  dominant shape with the counts as evidence.
- **Framework semantics before proposing a shape.** For example, how does the router invoke
  handlers? Is `this` bound? A suggested refactor that breaks at runtime is worse than none.
- **Duplicate paths.** Does it add a second way to do something the codebase already does, with
  different rules?
- **Overloaded fields and stringly-typed outcomes.** One field carrying several meanings that
  callers must parse.
- **Growth.** Does it grow an already oversized file or class? Does it put a new file where an
  existing one fits?

Design findings are usually 🟡 or 💡, not 🔴, unless they cause a defect. Still give the
restructure as concrete code.

**Check that the comments agree with each other before presenting or posting them.** Read the
full set as the author will receive it. If one suggestion removes the reason for another, keep
only one. For example, "fold this into the existing endpoint" removes the reason for "make this
handler a class", which only applies to registered handlers. Also make every code snippet assume
the same final shape: the same names, and calls written as `fn()` or `api.fn()` consistently.

**Reuse, both directions.** Does new code duplicate an existing helper — search adjacent
files and shared modules before accepting it. And when proposing a fix, cite the in-repo
precedent (an analogous call site, DAO, widget, prior workaround). Deviation from local
convention is a finding; matching it makes the suggestion conventional rather than taste.

**Don't hand-review what tooling owns.** Formatting, import order, lint violations,
typos: run the tool, don't eyeball it — and only report the *delta* the PR introduced
(see below). Never spend review attention on what a formatter would fix.

## 3. Verify before writing

For each candidate finding, do the work that could disprove it. The diff alone is never enough.

- **Read the whole file**, not the hunk. Removed context is where behaviour changes hide.
- **Read the implementation of everything the diff calls** — helpers, DAOs, base classes,
  framework widgets. Defaults and semantics decide whether the change is right. (A helper
  named `retryWithBackoff` turned out to use a *constant* delay.)
- **Trace the other side.** Reviewing a consumer? Find the producer and read the ordering.
  Reviewing a producer? Find who reads it.
- **Pre-existing or introduced?** Run the identical check on the base branch and compare.
  Linter, formatter, and test findings are worthless without this — most "failures" are
  inherited. Only what this PR introduced can block; label the rest pre-existing and
  out of scope.
- **Run the tools and quote exact output** — linter, formatter, type checker, affected
  tests. One quoted line of tool output outranks a paragraph of reasoning.
- **Try to kill your own findings.** State the hypothesis, then look for the code that
  refutes it. What survives is a finding; what doesn't goes in the "checked, not an issue"
  list so nobody redoes the work.

## 4. Output format

Lead with a verdict and the evidence, then the comments.

```
## PR <N> — <title>

**<X> files, +A/−B.** One sentence on what it does.

**Verdict: <Approve | Comment | Request changes> — <one line why.>**

What I verified: <commands run, files read, exact tool output, hypotheses ruled out.
Include what the PR gets right.>

---

## Comments to post

### 1. 🔴 [blocking] <Imperative headline naming the defect>
`path/to/file.ext` — **line N**

> Short and direct. One sentence on the defect and a failure trace of at most 2–3 steps
> (inputs or state → wrong output). Then always the fix, as a diff the author can apply.
> Describing the fix, or showing only a test, isn't enough.

### 2. 🟡 [important] ...

---

**🟢 Nits:** <one sentence each.>

**Non-issues I checked:** <hypotheses ruled out, with evidence.>
```

Severity label on every comment in the chat review — ordering alone doesn't communicate
weight. These labels are for chat only; they are stripped before posting (see section 6):

- 🔴 `[blocking]` — must fix before merge
- 🟡 `[important]` — should fix; discuss if you disagree
- 🟢 `[nit]` — optional
- 💡 `[suggestion]` — alternative approach, genuinely a trade-off
- 🎉 `[praise]` — verified-correct non-obvious work

Rules:

- **Severity first, not file order.** Correctness → behaviour change → maintainability → nits.
- **Anchor to `file` + `line`** in post-merge numbering. Get real numbers from the worktree
  (`grep -n`), never by counting diff lines.
- **Flag un-postable anchors.** The review API only accepts lines inside a diff hunk.
  Compute each hunk's new-side range from its `@@ -old,+new,span @@` header; if a finding
  falls outside, say so and give the nearest in-hunk anchor or suggest a file-level comment.
- **Every comment carries a concrete alternative** — real code in the repo's style, not
  "consider refactoring".
- **Say what the PR gets right**, especially where you verified a non-obvious claim.
  That's information, not politeness — and it tells the author which claims were checked.
- **Address the root cause.** If the change treats a symptom and the real fix is available,
  lead with that, then say whether the stopgap is shippable and what should mark it.

## 5. Tone

State verified findings as facts. You read the code and ran the tools, so write
declaratively: "this changes rendering on any device narrower than 412dp, here are the
numbers." Do **not** soften a verified defect into a rhetorical question ("what happens if
the list is empty?") or a hedge ("this might be a problem?") — you already know the answer,
and pretending otherwise makes the author redo your work and misreads your confidence.

Save collaborative phrasing for where it's honest: genuine trade-offs, taste, and
architecture calls get "recommend X because Y — your call." Attack the code, never the
author, and never editorialise about how the mistake happened.

## 6. Don't post without being asked

Publishing is outward-facing and visible to the author and team. Present the comments in
chat and ask whether to post them, all or a subset. Only then:

```bash
gh pr review <N> --repo <owner>/<repo> --comment --body-file <file>
gh api repos/<owner>/<repo>/pulls/<N>/comments -f body=... -f commit_id=... -f path=... -F line=... -f side=RIGHT
```

**Never post severity markers to GitHub.** No emoji (🔴 🟡 🟢 💡 🎉) and no labels
(`[blocking]`, `[important]`, `[nit]`, `[suggestion]`, `[praise]`) in any posted comment or
review body. Start each posted comment directly with its plain headline:
`**An async mapper sends {} with a 200**`, not `🟡 **[important] An async mapper…**`.
Before posting, grep every body file for these markers and remove them.

## 7. If a finding turns out to be wrong

Retract in one line, say what the code actually shows, and state whether the
recommendation survives. Don't defend it, don't over-apologise. A correction is part of
the deliverable — but the fix is to verify harder next time, not to hedge every claim.

---

*Section 2's dimension sweep, the severity labels, the size gate, and the
"don't hand-review what tooling owns" rule are adapted from
[awesome-skills/code-review-skill](https://github.com/awesome-skills/code-review-skill) (MIT).
That skill's per-language reference guides are worth installing alongside this one — it
covers breadth (what to look for in 24 ecosystems), this covers depth (how to be sure).*
