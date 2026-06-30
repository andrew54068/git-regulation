# Hit branch strategy — distilled rules

Self-contained digest of the *Hit 分支策略與 GitHub 操作手冊* (1A v3.0, "Release-Branch per Version"). Source of truth for `github-flow`. Commit-message rules are intentionally NOT duplicated here — those belong to the `hit-committer` skill (see "Commits" below).

## Core model

- **`main`** — the single long-lived trunk (= develop). All `feature/ bug/ refactor/ test/ chore` work integrates here. **Protected.**
- **`release/v<X.Y.Z>`** — cut from `main` *at release time* and kept as that version's maintenance line. One per version; multiple can be in use at once (e.g. `v1.1.0` and `v1.2.0`). **Protected.** Because dev and release environments are identical, "release" = "branch from main + tag" (no staging sync branch).
- **Release flows back into `main` by PR (merge commit).** `main` is both the *source* of all future releases and the *confluence* where each line's fixes land. Two backflows: (1) **finalize sync** — a release line's release-prep commits (version bump, changelog) merge back; (2) **newest-line hotfix** — see below. Always **Create a merge commit** (never squash/rebase a protected line). A retired version's branch may be deleted.
- **Hotfix** — fix on the **earliest still-in-use `release/v*` line where the bug was introduced** (so forward-only propagation covers every affected line); branch `hotfix/v<X.Y.Z>-<name>` from it, PR back into that release, tag the patch. Then: **newest line** (no newer in-use release) → merge `release/v<X.Y.Z> → main`; **superseded line** → **cherry-pick** (via `port/*-to-<target>` + PR) to `main` and every *newer* in-use release.
- History: read `main`'s mainline with `git log --first-parent`; each `release/v<X.Y.Z>` is a snapshot of `main` at a point in time plus that version's own hotfixes.

## Iron rule

Integration into `main` or **any** `release/v<X.Y.Z>` is **only** through a GitHub Pull Request — **no direct push of commits**. `feature/* bug/* refactor/* test/* chore/* → main`; `hotfix/v*-* → release/v<X.Y.Z>`; `release/v<X.Y.Z> → main` backflow (finalize / newest-line hotfix, **merge commit**); superseded-line cherry-pick propagation via `port/*-to-<target>`. **Exception:** the initial `git push -u` that creates a new `release/v<X.Y.Z>` on the remote is allowed — this is branch creation, not a commit push. Whether the server enforces this via branch-protection is discovered in Step 0. Locally you never push commits straight to a long-lived / release branch — even the backflow uses `release/*` as a PR **head**, never a direct push.

## Branch table

Every branch except `main` is named `<category>/<description>`; version and short description join with `-` (e.g. `hotfix/v1.1.0-crash-on-boot`).

| Branch | Naming | Source | Endpoint | Commit type | Life |
|---|---|---|---|---|---|
| trunk (develop) | `main` | — | — | — | long-lived · protected |
| version release | `release/v<X.Y.Z>` | `main` | kept as version line; PR-merges back to `main` (finalize / newest-line hotfix) | — | version line · protected |
| feature | `feature/<desc>` | `main` | PR-merge → `main` | `[feat]` | short |
| bug fix | `bug/<desc>` | `main` | PR-merge → `main` | `[fix]` | short |
| refactor | `refactor/<desc>` | `main` | PR-merge → `main` | `[refactor]` | short |
| test | `test/<desc>` | `main` | PR-merge → `main` | `[test]` | short |
| chore | `chore/<desc>` | `main` | PR-merge → `main` | `[chore]` | short |
| hotfix | `hotfix/v<X.Y.Z>-<desc>` | `release/v<X.Y.Z>` | PR-merge → that release | `[hotfix]` | short |
| hotfix propagation | `port/v<X.Y.Z>-<desc>-to-<target>` | target (`main` / newer `release/v*`) | cherry-pick → PR-merge into target | `[hotfix]` | short |

- `refactor/*`: internal structure only, **no outward behavior change**.
- `test/*`: test code only, **no product code**.
- `chore/*`: maintenance — deps, build/CI, file moves, version bumps. Also the home for `docs`/`style`/`perf`-only changes (no dedicated branch exists for those); the commit *type* stays accurate via hit-committer.
- `port/*`: carries only a cherry-picked hotfix to one target; its name **ends in `-to-<target>`** (one branch per target, e.g. `-to-main`, `-to-v1.2.0`) so a multi-target propagation never collides. Cherry-pick keeps the original message, so its type stays `[hotfix]`.
- Short-lived branches merge via **Rebase and merge** or **Squash and merge**. The one exception: a `release/* → main` backflow PR **must** use **Create a merge commit** — squash/rebase would collapse or rewrite the protected maintenance line.

## Tags

| Tag | Purpose | Placed on | GitHub Release |
|---|---|---|---|
| `v<X.Y.Z>` | human-readable version anchor (PM / support) | `release/v<X.Y.Z>` — the cut point for a first release, or the hotfix PR-merge commit for a patch | ✅ yes |

One `v<X.Y.Z>` tag per version. Cross-version fixes are traced by **cherry-pick + PR**, not semantic tags. **Tagging and Releases are Release-Owner actions** — `github-flow` stops at "PR opened".

## Commits

Commit crafting is the **`hit-committer`** skill's job — `github-flow` delegates to it once the correct branch is checked out. In short: atomic, single-concern commits with English `[type] Description` titles (`[feat] [fix] [hotfix] [refactor] [perf] [docs] [test] [style] [chore]`). Full spec lives in `hit-committer/references/commit-spec.md`. Do not re-implement it here.

## The 5 disciplines

1. **`main` & `release/*` — no direct commit push, no history rewrite.** No `commit` / `rebase` / `commit --amend` / `push --force` / `tag -f` on them. All changes via a PR — a `release/*` line may be a PR **head** into `main` (backflow), still a PR, never a direct commit push. **Exception:** the initial `git push -u` that creates a new `release/v<X.Y.Z>` on the remote is allowed (branch creation, not a commit push). Whether the server enforces this via branch-protection is discovered in Step 0.
2. **Hotfixes propagate immediately, never accumulate.** Fix on the earliest affected in-use line; after it merges and is patch-tagged, *immediately* (before the next hotfix opens) carry it to `main` — **newest line** → merge `release/v<X.Y.Z> → main`; **superseded line** → cherry-pick (via `port/*-to-<target>` + PR) to `main` and every *newer* in-use release. Skipping `main` makes future releases regress the bug.
3. **The hotfix is the authority on conflicts.** When a `port/*` cherry-pick **or** a `release/* → main` backflow merge conflicts, the release's hotfix content wins — the target's pre-existing implementation is unverified and must be overwritten.
4. **Short-lived branches integrate clean + tested.** `rebase -i` tidy + the matching integration tests before PR-merge; CI must test the *merged* state.
5. **Every PR carries a test record.** Any PR into `main` / `release/*` must attach test evidence (build output, unit/integration results, on-device screenshots — as fits the change) in its description. Missing it ⇒ Reviewer `Request changes`, Release Owner won't merge. Until CI exists, this is the only quality gate.

## Roles

| Role | GitHub perm | Does |
|---|---|---|
| Repo Owner | Admin | repo, branch protection on `main` + `release/*`, CODEOWNERS |
| Release Owner | Maintain | **only** role that merges into `main` / `release/*`; creates `release/v<X.Y.Z>`; tags `v<X.Y.Z>`; publishes Releases; drives hotfix cherry-pick propagation |
| Developer | Write | creates `feature/ bug/ hotfix/ port/` branches, opens PRs, responds to review |
| Reviewer | Write | reviews, Approve / Request changes |

`github-flow` operates in the **Developer** lane: branch → (delegate commits) → PR. It does not merge, tag, or release.

## PR base routing (what the flow must get right)

- `feature/* bug/* refactor/* test/* chore/*` → base **`main`**
- `hotfix/v<X.Y.Z>-*` → base **`release/v<X.Y.Z>`** (matching version)
- `port/v<X.Y.Z>-*-to-<target>` → base **the target** (`main`, or a *newer* in-use `release/v*`)
- `release/v<X.Y.Z>` itself (finalize sync / newest-line hotfix backflow) → base **`main`**, merged with **Create a merge commit**

## Pre-PR checklist (developer)

- [ ] Clean atomic commits (crafted by hit-committer)
- [ ] Aligned with latest base (`git pull --rebase <remote> <base>`)
- [ ] Commit/PR references the Issue (`Closes #N`) when one exists
- [ ] **Test record present in the PR description** (Discipline 5)
- [ ] Base branch correct per routing table; target is protected & reached via PR
