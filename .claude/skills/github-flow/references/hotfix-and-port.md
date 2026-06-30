# Hotfix, cross-version propagation, and release cut — playbooks

Exact command sequences for the higher-stakes flows. `<remote>` is the detected remote (often `origin`, sometimes `hantop`). `github-flow` drives the Developer-lane steps (branch → delegate commits → PR) and stops before Release-Owner-only actions (merge, tag, Release). Commit crafting in every "commit" step below is delegated to the **`hit-committer`** skill — don't hand-write the messages here.

---

## §7.1 Hotfix → its `release/v<X.Y.Z>`

An urgent fix to an already-shipped version. It is fixed **on the release line**, not on `main`. First pick the **earliest still-in-use `release/v*` line where the bug was introduced** (ask if unclear): fixing at the earliest affected line means forward-only propagation (§7.2 / §7.3) covers every affected version with no older line left behind. The example uses `release/v1.1.0`.

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

Then publish the GitHub Release. **Immediately** carry the fix to `main` (Discipline 2): if `release/v1.1.0` is the **newest** in-use line → **§7.3** (merge it into `main`); if it is **superseded** by a newer in-use line → **§7.2** (cherry-pick to `main` + every newer line).

---

## §7.2 Propagate a *superseded-line* hotfix (cherry-pick via `port/*-to-<target>` + PR)

Use this when the hotfixed line is **not** the newest in-use line (a newer `release/v*` exists). If it **is** the newest, use §7.3 instead. Start the moment `v1.1.1` is tagged. Targets are **`main`** (always — else the next release cut from `main` regresses the bug) **and every *newer* in-use `release/v*`** (e.g. a concurrently-maintained `release/v1.2.0`). Older in-use lines need nothing — you fixed at the earliest affected line. Retired releases are skipped. **One `port/*` branch per target**, name ending in `-to-<target>`:

```bash
# 0. find the hotfix's commit SHA on the release line
git checkout release/v1.1.0
git pull --ff-only <remote> release/v1.1.0
git log --oneline -5            # note the hotfix SHA → <hotfix-sha>

# 1. cut a port branch FROM THE TARGET, named -to-<target> (main shown)
git checkout main
git pull --ff-only <remote> main
git checkout -b port/v1.1.1-crash-on-boot-to-main
#    for a newer release line instead, e.g. release/v1.2.0:
#    git checkout release/v1.2.0 && git pull --ff-only <remote> release/v1.2.0
#    git checkout -b port/v1.1.1-crash-on-boot-to-v1.2.0

# 2. cherry-pick the hotfix (this carries the original commit, so no hit-committer step here)
git cherry-pick <hotfix-sha>
#    On conflict: the hotfix is authoritative (Discipline 3). Resolve in its favor, then:
#    git add <file> && git cherry-pick --continue

# 3. push and open the PR
git push -u <remote> port/v1.1.1-crash-on-boot-to-main
```

PR settings:

| PR field | Value |
|---|---|
| base | the **target** (`main`, or a *newer* in-use `release/v*` like `release/v1.2.0`) |
| head | `port/v1.1.1-crash-on-boot-to-<target>` (e.g. `-to-main`, `-to-v1.2.0`) |
| title | `[hotfix] Backport v1.1.1 crash-on-boot fix to <target>` |
| body | link the `v1.1.1` Release / original hotfix PR; include the Discipline-5 test record (verify the fix still works on this target) |
| merge method | Rebase and merge (or Squash) |

Repeat per target (one `-to-<target>` branch each). Cost: the same fix gets a different SHA on each branch (original on the release + one per target). This double-SHA is one-shot and bounded — it does **not** reintroduce the old repeated-conflict problem (which came from repeatedly two-way-rebasing long-lived branches).

After all targets are done: confirm each port PR is open/merged, the Release is published, the board card is in Done, and the short-lived + `port/*` branches are deleted (the `release/*` lines stay).

---

## §7.3 Backflow a *newest-line* hotfix (merge `release/v<X.Y.Z>` → `main`)

Use this when the hotfixed line is the **newest** in-use line (no newer `release/v*`). The fix reaches `main` by **merging the release line itself** — there is **no `port/*`-to-main**. `github-flow` opens the PR; the Release Owner merges (both branches protected). Example: newest line `release/v1.2.0`, hotfix tagged `v1.2.1`.

```bash
# align both ends locally (never push protected branches directly)
git fetch <remote>
git checkout release/v1.2.0
git pull --ff-only <remote> release/v1.2.0   # already has the hotfix + tag v1.2.1
git checkout main
git pull --ff-only <remote> main
```

PR settings:

| PR field | Value |
|---|---|
| base | `main` |
| head | `release/v1.2.0` (the newest maintenance line itself — not a new branch) |
| title | `[hotfix] Merge v1.2.1 fix back to main` |
| body | link the `v1.2.1` Release / original hotfix PR; include the Discipline-5 test record |
| merge method | **Create a merge commit** (mandatory — squash/rebase would collapse or rewrite the protected line) |

- On a merge conflict the **hotfix is authoritative** (Discipline 3).
- After merge the maintenance line **stays** — don't let "Automatically delete head branches" remove it; re-push from local if it does.
- No `port/*` is needed — `main` now carries the fix via the merge. (If older in-use lines were affected too, you'd have fixed at the earliest one and used §7.2 instead.)

---

## §6 Release cut (Release-Owner context — informational)

`github-flow` does **not** perform this (it's not a PR flow, and only the Release Owner may). Documented so you can recognize and explain it.

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

Then GitHub `Releases → Draft a new release` → choose tag `v1.1.0` → `Generate release notes` → attach artifacts → `Publish`. The branch is kept as the version's maintenance line.

**Finalize sync (a PR flow `github-flow` *does* open):** if release-prep commits (version bump, changelog, release notes) were added on the line, open a `release/v1.1.0 → main` PR so `main` gains them:

| PR field | Value |
|---|---|
| base | `main` |
| head | `release/v1.1.0` (the maintenance line itself) |
| merge method | **Create a merge commit** (mandatory) |

`release/*` is protected — the skill opens this PR and stops; the Release Owner merges, and the line **stays**. Later fixes arrive by backflow merge (newest line, §7.3) or cherry-pick (superseded, §7.2).
