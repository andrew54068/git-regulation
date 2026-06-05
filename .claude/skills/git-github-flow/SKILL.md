---
name: git-github-flow
description: >-
  Run the Hit branch strategy end to end: classify a change, create or select the correct short-lived branch from the correct base, enforce protected `main` / `release/*`, push to the auto-detected remote, and open a Pull Request against the right base with the mandatory test-record — plus the hotfix → cherry-pick `port/*` cross-version propagation flow. Use this WHENEVER a task touches branches or integration in a Hit-strategy repo: "open a PR", "commit this and open a PR", "start a feature / bug / hotfix", "backport / port a fix", "create a branch", "follow the git flow", or anything involving `release/v*`, hotfixes, or propagation. SINGLE RESPONSIBILITY = BRANCH + PR + PROPAGATION; it DELEGATES the actual commit crafting to the hit-committer skill (Hit `[type] Description` messages). It never commits or pushes directly to protected branches, and never merges / tags / publishes releases (those are Release-Owner actions). For a pure "just commit this" with no branch or PR, use hit-committer directly instead.
---

# git-github-flow

Own the path from a change to an open Pull Request under the **Hit branch strategy**: single protected `main`, per-version `release/v<X.Y.Z>` lines, PR-only integration, and cherry-pick propagation of hotfixes. Pick the right branch, enforce the protections, route the PR to the right base, and drive hotfix backports.

It has **one responsibility — branch + integration — and delegates commits.** When it's time to record changes, it calls the **`hit-committer`** skill to craft atomic Hit-format commits, rather than re-implementing message rules. That separation is the point: `hit-committer` can be used alone for a quick commit; this skill handles everything around it.

The rules are bundled and self-contained, so this works in any team repo (`ipc-firmware`, `ipc-sdk`, …) even without the manual checked in:
- `references/hit-rules.md` — branch/tag tables, the 5 disciplines, roles, PR-base routing. Your source of truth.
- `references/hotfix-and-port.md` — the hotfix (§7.1), cherry-pick propagation (§7.2), and release-cut (§6) command playbooks.

## The one rule that overrides everything

`main` and every `release/v<X.Y.Z>` are **protected**: no direct push, no `rebase`/`amend`/`--force`/`tag -f`, no committing while standing on them. Every change reaches them via a short-lived branch and a PR. If `HEAD` is on `main` or `release/*`, the first move is always to create the proper branch (uncommitted changes follow `git checkout -b`, so the protected branch stays clean).

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

Trip-ups: the model has **no `docs/` `style/` or `perf/` branch** — those ride on `chore/` (say so in the plan). `refactor/*` must be behavior-preserving. `hotfix/` is only for fixing a shipped release; if the affected version is unclear, ask (it's in the branch name and decides the base). If the tree mixes unrelated concerns, prefer **separate branches/PRs** and surface the split in the plan.

## Step 2 — Decide & create/select the branch

- **Name** `<category>/<concise-kebab-desc>`; versioned categories embed the version (`hotfix/v1.1.0-crash-on-boot`, `port/v1.1.1-crash-on-boot`).
- **Base / freshness**: `git fetch <remote>` first. feature/bug/refactor/test/chore branch from up-to-date `main`; hotfix from the matching `release/v<X.Y.Z>`; port from its target. From a clean checkout: `git checkout <base> && git pull --ff-only <remote> <base>` then `git checkout -b …`. With changes already dirty on a protected branch, `git checkout -b <branch>` carries them off safely.
- **Reuse** an existing valid short-lived branch if you're already on one; check `gh pr view <branch>` to avoid duplicate PRs (push updates instead).
- **hotfix / port** are higher-stakes — follow `references/hotfix-and-port.md` exactly (SHA capture, "hotfix wins on conflict", immediate propagation).

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

| Branch | PR base |
|---|---|
| `feature/* bug/* refactor/* test/* chore/*` | `main` |
| `hotfix/v<X.Y.Z>-*` | `release/v<X.Y.Z>` (the matching version) |
| `port/v<X.Y.Z>-*` | the **target** (`main`, or another in-use `release/v*`) |

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

This skill inserts the **template**; it doesn't run the tests. Tell the user the PR won't merge until the evidence is filled in. Recommended merge method (for the Release Owner): **Rebase and merge** or **Squash and merge**.

Report the PR URL when done. If this was a **hotfix**, remind the user of Discipline 2: once merged and tagged, the fix must be **immediately** cherry-picked to `main` and every other in-use `release/v*` via `port/*` PRs (see `references/hotfix-and-port.md`).

## Guardrails — never do these

- Never `commit`/`push`/`rebase`/`amend`/`--force`/`tag -f` on `main` or any `release/*`. Everything via a branch + PR.
- Never invent a base. Hotfix → its release line; port → its target; everything else → `main`.
- Never merge the PR, create version tags, or publish GitHub Releases — Release-Owner actions. Stop at "PR opened".
- Don't write commit messages here — delegate to `hit-committer`.
- Don't let a hotfix sit un-propagated (Discipline 2); on cherry-pick conflicts the release's hotfix is authoritative (Discipline 3).

## When to read the references

- `references/hit-rules.md` — branch/tag tables, the 5 disciplines, roles, PR-base routing. Source of truth for any rule question.
- `references/hotfix-and-port.md` — exact command playbooks for hotfix (§7.1), cross-version cherry-pick propagation (§7.2), and the release cut (§6, Release-Owner context).
