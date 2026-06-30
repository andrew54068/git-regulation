---
name: github-flow
description: >-
  Open the correct PR under the Hit branch strategy: pick the right branch and base, enforce protected `main` / `release/*`, and route the PR — incl. `release/* → main` backflow (finalize / newest-line hotfix) and older-line hotfix `port/*` cherry-pick propagation. Use for Hit-repo branch/PR work: open a PR, start a feature/bug/hotfix, backport/port a fix, finalize/merge a release, or anything touching `release/v*`, hotfixes, or backflow. Delegates commits to hit-committer; opens PRs but never merges, tags, or releases.
---

# github-flow

Own the path from a change to an open Pull Request under the **Hit branch strategy**: single protected `main`, per-version `release/v<X.Y.Z>` lines, PR-only integration, `release/* → main` backflow (finalize sync + newest-line hotfix merge), and cherry-pick propagation of older-line hotfixes. Pick the right branch, enforce the protections, route the PR to the right base, and drive hotfix backflow/backports.

It has **one responsibility — branch + integration — and delegates commits.** When it's time to record changes, it calls the **`hit-committer`** skill to craft atomic Hit-format commits, rather than re-implementing message rules. That separation is the point: `hit-committer` can be used alone for a quick commit; this skill handles everything around it.

The rules are bundled and self-contained, so this works in any team repo (`ipc-firmware`, `ipc-sdk`, …) even without the manual checked in:
- `references/hit-rules.md` — branch/tag tables, the 5 disciplines, roles, PR-base routing. Your source of truth.
- `references/hotfix-and-port.md` — the hotfix (§7.1), older-line cherry-pick propagation (§7.2), newest-line `release → main` merge (§7.3), and finalize-sync / release-cut (§6) command playbooks.

## The one rule that overrides everything

`main` and every `release/v<X.Y.Z>` are treated as long-lived integration branches: this skill will never commit while standing on them, never `rebase`/`amend`/`--force`/`tag -f` them, and never push commits to them directly. Every change reaches them via a PR. Whether the server actually enforces this is discovered in **Step 0** — if branch-protection is missing or incomplete, warn the user so they can fix it. If `HEAD` is on `main` or `release/*` **with changes to commit**, the first move is always to create the proper branch (uncommitted changes follow `git checkout -b`, so the long-lived branch stays clean). **Two exceptions:** (1) **Branch creation** — the initial `git push -u` that creates a new `release/v<X.Y.Z>` on the remote is allowed (this is branch creation, not a commit push). (2) **Backflow** — a *clean* `release/*` line that is itself the PR **head** for a `release/* → main` backflow (finalize sync / newest-line hotfix) is **not** moved off — you open `release/* → main` directly. The no-commit rule still holds: you never commit *onto* a long-lived branch, but one may *be* a PR head.

## Operating mode: Plan → approve → PR

Do the read-only investigation, then present **one plan** and wait for the user's go-ahead before changing anything. The plan shows: the branch (name + base), the atomic commits hit-committer will make (file groups + messages), and the PR (base + title + body). After one approval, execute through to the open PR. Don't merge — that's the Release Owner.

## Step 0 — Orient (read-only, in parallel)

```bash
git status
git branch --show-current          # on a protected branch?
git log --oneline -8
git remote                         # remote is NOT always "origin" (e.g. "hantop")
git branch -a --list 'release/*'   # which release lines exist / are in use
```

Derive the **remote name** (sole remote; else prefer `origin`, else the one matching the GitHub repo, else ask — never hardcode `origin`), the **trunk** (`main`), whether you're on a **protected branch**, and whether a valid short-lived branch already exists to reuse.

### Read branch-protection rules (also in Step 0)

Different repos have different protection configs. Before planning, probe the actual rules for `main` and every in-use `release/*` line so you know what the repo enforces — don't hardcode assumptions.

```bash
# main
gh api repos/{owner}/{repo}/branches/main/protection --jq '{
  require_pr: (.required_pull_request_reviews != null),
  required_approvals: .required_pull_request_reviews.required_approving_review_count,
  dismiss_stale: .required_pull_request_reviews.dismiss_stale_reviews,
  require_last_push_approval: .required_pull_request_reviews.require_last_push_approval,
  enforce_admins: .enforce_admins.enabled,
  force_push: .allow_force_pushes.enabled,
  linear_history: .required_linear_history.enabled,
  conversation_resolution: .required_conversation_resolution.enabled
}' 2>/dev/null || echo '{"error": "no protection or no access"}'

# each release line (URL-encode the slash)
gh api repos/{owner}/{repo}/branches/release%2Fv<X.Y.Z>/protection --jq '...' 2>/dev/null
```

Also check for the newer **rulesets** API (repos that use it won't have legacy protection):
```bash
gh api repos/{owner}/{repo}/rulesets --jq '.[].name' 2>/dev/null
```

**What to do with the data:**
- **`require_pr: true`** — confirms the PR-only integration rule is enforced server-side. If `false` on `main` or a `release/*`, warn the user ("branch protection isn't enforcing PR reviews — direct pushes could bypass the Hit strategy").
- **`required_approvals`** — note in the plan so the user knows how many approvals are needed before the Release Owner can merge.
- **`dismiss_stale_reviews: true`** — after force-pushing or rebasing, existing approvals reset. Warn: "approvals will be dismissed if you rebase."
- **`require_last_push_approval: true`** — the person who pushed last cannot be the sole approver. Note this if relevant (solo contributor).
- **`enforce_admins: true`** — even admins can't bypass. Useful context when the user *is* admin.
- **`force_push: true`** — unusual for protected branches; warn the user that force-push is allowed (the Hit strategy forbids it regardless, but the repo isn't enforcing that).
- **`linear_history: true`** — merge commits are banned, which conflicts with the backflow rule (backflow PRs *require* merge commits). Flag this as a blocker: "linear history is on for `main`, but `release/* → main` backflow needs a merge commit — ask the repo owner to allow merge commits or disable linear-history for backflow."
- **error / no protection** — the branch exists but has no protection rules. Warn: "no branch protection on `<branch>` — Hit strategy relies on it; recommend setting up protection."

Don't block on missing protection — the skill still works, it just can't rely on the server to prevent mistakes. Surface findings in the plan so the user can fix any gaps.

## Step 1 — Classify the change → branch category

The change's nature picks the branch. (Per-commit *type* is hit-committer's call and may differ within a branch.)

| The change is… | Branch | Base | Typical commit type |
|---|---|---|---|
| Externally visible new feature/behavior | `feature/<desc>` | `main` | `[feat]` |
| Non-urgent bug fix | `bug/<desc>` | `main` | `[fix]` |
| Internal restructuring, no outward change | `refactor/<desc>` | `main` | `[refactor]` |
| Test code only | `test/<desc>` | `main` | `[test]` |
| Deps / build / CI / version bump / **docs** / formatting / perf-only | `chore/<desc>` | `main` | `[chore]`/`[docs]`/`[style]`/`[perf]` |
| Urgent fix to a shipped version | `hotfix/v<X.Y.Z>-<desc>` | `release/v<X.Y.Z>` | `[hotfix]` |
| Propagating a merged hotfix onward | `port/v<X.Y.Z>-<desc>` | target branch | `[hotfix]` (cherry-picked) |

Trip-ups: the model has **no `docs/` `style/` or `perf/` branch** — those ride on `chore/` (say so in the plan). `refactor/*` must be behavior-preserving. `hotfix/` is only for fixing a shipped release; fix on the **earliest still-in-use `release/v*` line where the bug was introduced** (ask if unclear) — that line is in the branch name and decides the base. If the tree mixes unrelated concerns, prefer **separate branches/PRs** and surface the split in the plan. Note: merging a `release/*` line back to `main` (finalize sync / newest-line hotfix) is **not** a new branch — see "Hotfix propagation & backflow" below.

## Step 2 — Decide & create/select the branch

- **Name** `<category>/<concise-kebab-desc>`; versioned categories embed the version, and **every `port/*` ends in `-to-<target>`** (`hotfix/v1.1.0-crash-on-boot`, `port/v1.1.1-crash-on-boot-to-main`, `port/v1.1.1-crash-on-boot-to-v1.2.0`) so one hotfix propagating to several targets never collides.
- **Base / freshness**: `git fetch <remote>` first. feature/bug/refactor/test/chore branch from up-to-date `main`; hotfix from the matching `release/v<X.Y.Z>`; port from its target. From a clean checkout: `git checkout <base> && git pull --ff-only <remote> <base>` then `git checkout -b …`. With changes already dirty on a protected branch, `git checkout -b <branch>` carries them off safely.
- **Reuse** an existing valid short-lived branch if you're already on one; check `gh pr view <branch>` to avoid duplicate PRs (push updates instead).
- **hotfix / port / `release→main` backflow** are higher-stakes — follow `references/hotfix-and-port.md` exactly (earliest-affected line, SHA capture, "hotfix wins on conflict", newest→merge vs superseded→cherry-pick, immediate propagation).

## Step 3 — Commit (delegate to hit-committer)

Once you're on the correct branch, hand the commit step to the **`hit-committer`** skill: it groups the changes into atomic units and writes Hit `[type] Description` messages. Don't write commit messages here yourself — that's its job, and keeping it there means one source of truth for the format. You stay responsible for *which branch* those commits land on.

## Step 4 — Present the plan, then stop

Show: **branch** (name + base + why that category), **commits** (the atomic units + messages hit-committer will create), **PR** (base + title + body incl. the test-record template), and ask for the **Issue number** for `Closes #N` (proceed without if none). Wait for explicit approval before writing anything.

## Step 5 — Push + open the PR (after approval)

```bash
git fetch <remote>
git pull --rebase <remote> <base>     # align with latest base before pushing
git push -u <remote> <branch>
```

**PR base routing** — the part a generic tool gets wrong:

| Branch (PR head) | PR base |
|---|---|
| `feature/* bug/* refactor/* test/* chore/*` | `main` |
| `hotfix/v<X.Y.Z>-*` | `release/v<X.Y.Z>` (the matching version) |
| `port/v<X.Y.Z>-*-to-<target>` | the **target** (`main`, or a newer in-use `release/v*`) |
| `release/v<X.Y.Z>` itself (finalize sync / newest-line hotfix backflow) | `main` — **Create a merge commit** |

```bash
gh pr create --repo <owner/repo> --base <base> --head <branch> \
  --title "[type] Description (#issue)" --body-file /tmp/hit-pr-body.md
```

- **Title** mirrors the primary commit: `[type] Description (#issue)` (also becomes the squash-merge commit). The manual's Step-7 example uses a looser `feat: …`; standardize on the documented `[type]` form.
- **Body** must include `Closes #N` (when there's an issue) and the **Test record** section — Discipline 5 makes it mandatory before merge:

```markdown
## Summary
- <1–3 lines: what this PR does and why>

Closes #<N>

## Test record (Discipline 5 — required before merge)
Fill in real evidence, or mark N/A with a reason. Reviewers reject PRs lacking this.
- [ ] Build: <command output / N/A — reason>
- [ ] Unit tests: <output / N/A — reason>
- [ ] Integration tests: <output / N/A — reason>
- [ ] Manual / on-device verification: <screenshots / notes / N/A — reason>
```

This skill inserts the **template**; it doesn't run the tests. Tell the user the PR won't merge until the evidence is filled in. Recommended merge method (for the Release Owner): **Rebase and merge** or **Squash and merge** for short-lived branches; **Create a merge commit** is **mandatory** for a `release/* → main` backflow PR (squash/rebase would collapse or rewrite the protected maintenance line).

Report the PR URL when done. If this was a **hotfix**, remind the user of Discipline 2 — once merged and tagged, propagate **immediately** (before the next hotfix), following the decision below.

## Hotfix propagation & `release/* → main` backflow

Two flows carry a `release/*` line's work back to `main`. Both open a PR with **head = the release line itself** (no new short-lived branch) and **base = `main`**, merged with **Create a merge commit**. The skill **opens** this PR and stops; the Release Owner merges.

- **Finalize sync** — after a release is cut and its release-prep commits (version bump, changelog) land on `release/v<X.Y.Z>`, open `release/v<X.Y.Z> → main` so `main` gains them.
- **Newest-line hotfix** — see step 3 below.

**Decision** (run after the hotfix PR merges into its release line and is patch-tagged):
1. The hotfix was made on the **earliest still-in-use `release/v*` line where the bug was introduced** (ask if unclear). Everything affected is then ≥ that line, so forward-only propagation is complete coverage.
2. Is that line the **newest** in-use line — i.e. no in-use `release/v*` has a higher semver?
3. **Yes (newest)** → open **one** PR `release/v<X.Y.Z> → main` (merge commit). **No `port/*`-to-main**; `main` gets the fix from the merge.
4. **No (superseded)** → cherry-pick to **`main` and each *newer* in-use line**, one `port/v<patch>-<desc>-to-<target>` + PR per target. Do **not** merge the superseded line into `main`. Older in-use lines need nothing (you fixed at the earliest affected line).
5. Conflict on any `port/*` cherry-pick **or** the `release/* → main` merge → the **hotfix content is authoritative** (Discipline 3).

See `references/hotfix-and-port.md` for the exact command playbooks (§7.2 superseded, §7.3 newest, §6 finalize).

## Guardrails — never do these

- Never `commit`/`rebase`/`amend`/`--force`/`tag -f` on `main` or any `release/*`. Never push commits to them directly — all integration via PR. **Exception:** the initial `git push -u` that creates a new `release/v<X.Y.Z>` on the remote is allowed (branch creation, not a commit push). A `release/*` line MAY be a PR **head** into `main` (backflow) — still a PR, still never a direct commit push.
- Never invent a base. Hotfix → its release line; `port/*` → its target; `release/*` backflow → `main` (merge commit); everything else → `main`.
- Never merge the PR, create version tags, or publish GitHub Releases — Release-Owner actions. Stop at "PR opened" (including the `release/* → main` PR).
- Don't write commit messages here — delegate to `hit-committer`.
- Don't let a hotfix sit un-propagated (Discipline 2): newest line → merge to `main`; superseded → cherry-pick to `main` + each newer in-use line. On a cherry-pick **or** backflow-merge conflict, the hotfix is authoritative (Discipline 3).

## When to read the references

- `references/hit-rules.md` — branch/tag tables, the 5 disciplines, roles, PR-base routing. Source of truth for any rule question.
- `references/hotfix-and-port.md` — exact command playbooks for hotfix (§7.1), superseded-line cherry-pick propagation (§7.2), newest-line `release → main` merge (§7.3), and finalize sync / release cut (§6, Release-Owner context).
