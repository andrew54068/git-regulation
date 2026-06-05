# Hotfix, cross-version propagation, and release cut — playbooks

Exact command sequences for the higher-stakes flows. `<remote>` is the detected remote (often `origin`, sometimes `hantop`). `git-github-flow` drives the Developer-lane steps (branch → delegate commits → PR) and stops before Release-Owner-only actions (merge, tag, Release). Commit crafting in every "commit" step below is delegated to the **`hit-committer`** skill — don't hand-write the messages here.

---

## §7.1 Hotfix → its `release/v<X.Y.Z>`

An urgent fix to an already-shipped version. It is fixed **on the release line**, not on `main`.

### Phase A — local prep

```bash
git fetch <remote>
git checkout release/v1.1.0
git pull --ff-only <remote> release/v1.1.0
git checkout -b hotfix/v1.1.0-crash-on-boot
```

Then make the fix and **commit via hit-committer** (one atomic `[hotfix]` commit). Tidy with `git rebase -i release/v1.1.0` only if needed, then align and push:

```bash
git pull --rebase <remote> release/v1.1.0
git push -u <remote> hotfix/v1.1.0-crash-on-boot
```

### Phase B — PR into the release (Developer opens; Release Owner merges)

| PR field | Value |
|---|---|
| base | `release/v1.1.0` |
| head | `hotfix/v1.1.0-crash-on-boot` |
| merge method | Rebase and merge (or Squash) |

`release/*` is protected — **never push the fix straight onto it.** The PR body still needs the Discipline-5 test record.

### Phase C — patch tag (Release Owner)

```bash
git checkout release/v1.1.0
git pull --ff-only <remote> release/v1.1.0
git tag -a v1.1.1 -m "Hotfix release v1.1.1"
git push <remote> v1.1.1
```

Then publish the GitHub Release. **Immediately** continue to §7.2 propagation (Discipline 2).

---

## §7.2 Propagate the hotfix (cherry-pick via `port/*` + PR)

Start the moment `v1.1.1` is tagged. Repeat for **every** branch that needs the fix: **always `main`** (else the next release cut from `main` regresses the bug), plus every other in-use `release/v*` (e.g. a concurrently-maintained `release/v1.2.0`). Retired releases can be skipped.

```bash
# 0. find the hotfix's commit SHA on the release line
git checkout release/v1.1.0
git pull --ff-only <remote> release/v1.1.0
git log --oneline -5            # note the hotfix SHA → <hotfix-sha>

# 1. cut a port branch FROM THE TARGET (main shown; other release/v* identical)
git checkout main
git pull --ff-only <remote> main
git checkout -b port/v1.1.1-crash-on-boot

# 2. cherry-pick the hotfix (this carries the original commit, so no hit-committer step here)
git cherry-pick <hotfix-sha>
#    On conflict: the hotfix is authoritative (Discipline 3). Resolve in its favor, then:
#    git add <file> && git cherry-pick --continue

# 3. push and open the PR
git push -u <remote> port/v1.1.1-crash-on-boot
```

PR settings:

| PR field | Value |
|---|---|
| base | the **target** (`main`, or another in-use `release/v*` like `release/v1.2.0`) |
| head | `port/v1.1.1-crash-on-boot` |
| title | `[hotfix] Backport v1.1.1 crash-on-boot fix to <target>` |
| body | link the `v1.1.1` Release / original hotfix PR; include the Discipline-5 test record (verify the fix still works on this target) |
| merge method | Rebase and merge (or Squash) |

Repeat per affected, in-use branch. Cost: the same fix gets a different SHA on each branch (original on the release + one per target). This double-SHA is one-shot and bounded — it does **not** reintroduce the old repeated-conflict problem (which came from repeatedly two-way-rebasing long-lived branches).

After all targets are done: confirm each port PR is open/merged, the Release is published, the board card is in Done, and the short-lived + `port/*` branches are deleted.

---

## §6 Release cut (Release-Owner context — informational)

`git-github-flow` does **not** perform this (it's not a PR flow, and only the Release Owner may). Documented so you can recognize and explain it.

```bash
git checkout main
git pull --ff-only <remote> main

# cut the version line from main and push it
git checkout -b release/v1.1.0
git push -u <remote> release/v1.1.0

# tag the cut point and push the tag
git tag -a v1.1.0 -m "Release v1.1.0"
git push <remote> v1.1.0
```

Then GitHub `Releases → Draft a new release` → choose tag `v1.1.0` → `Generate release notes` → attach artifacts → `Publish`. The branch is kept as the version's maintenance line; it never merges back to `main`. Future fixes arrive by cherry-pick (§7.2).
