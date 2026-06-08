---
name: hit-committer
description: >-
  Turn the working tree into atomic Git commits that follow the Hit commit-message spec — English `[type] Description` titles (lowercase type, capitalized intent, ≤72 chars) with module-level body bullets explaining what changed. Use this WHENEVER you are committing in a Hit-strategy repo: the user says "commit", "commit this", "make atomic commits", "hit commit", or the git-github-flow skill hands off its commit step. SINGLE RESPONSIBILITY = COMMITS ONLY: it groups changes into atomic, single-concern commits and writes Hit-format messages. It does NOT create branches, push, or open PRs — that is git-github-flow's job; for "commit AND open a PR" or anything involving branches/hotfix/release, use git-github-flow (which calls this skill for the commit step). It refuses to commit onto protected `main` / `release/*` and defers to git-github-flow to place the work on a proper branch first. Prefer this over the generic git-committer in Hit repos, because the Hit message format (`[type] Description`) differs from conventional commits (`type(scope):`).
---

# hit-committer

One job: take the changes in the working tree and record them as **atomic commits** with **Hit-compliant messages**. Nothing else.

This is deliberately narrow. Choosing branches, pushing, opening PRs, hotfix backports — none of that lives here. That's `git-github-flow`, which calls this skill when it needs commits crafted. Keeping the two apart means you can commit cleanly without triggering a whole branch/PR workflow, and the flow skill never has to re-implement message rules.

Not the generic `git-committer` either: Hit messages are `[type] Description` (English, capitalized), not conventional `type(scope): description`.

## Boundary — what this skill will and won't do

- **Will:** group the diff into atomic single-concern commits, pick each commit's `[type]`, write the message, stage explicit paths, commit, leave the tree clean.
- **Won't:** create/rename/switch branches, choose a base, `git push`, open PRs, tag, or propagate. If the task needs any of those, that's `git-github-flow`.
- **Refuses** to commit while `HEAD` is on protected `main` or `release/*`. Committing there breaks the branch strategy — stop and tell the user to run `git-github-flow` first to move the work onto a proper short-lived branch (their uncommitted changes follow a `git checkout -b`, so nothing is lost). The one exception: you were invoked *by* `git-github-flow`, which has already put you on the correct branch.

## Step 0 — Orient (read-only, in parallel)

```bash
git status
git diff
git diff --staged
git branch --show-current     # protected? -> stop and defer to git-github-flow
git log --oneline -8          # match the existing message style
```

If the current branch is `main` or matches `release/*`, do not commit — defer to `git-github-flow` (see Boundary above) unless it invoked you.

## Step 1 — Group into atomic commits

Atomic = **one logical, self-contained change per commit**; never mix unrelated edits. Read the diff, cluster it into units, and order them sensibly (foundations before the things that build on them).

- Prefer explicit `git add <paths>` per unit — `git add -A` destroys atomicity when concerns are mixed.
- When one file holds two unrelated changes, split them with `git add -p`.
- If the tree mixes genuinely separate concerns (a feature *and* an unrelated fix), still give each its own commit. If they also belong on different branches/PRs, say so — but that branching decision is `git-github-flow`'s to make, not yours.

## Step 2 — Write the message (exact Hit format)

```
[type] Imperative intent — WHY this change (first letter capitalized)

- Module-level WHAT changed (not function-by-function detail)
- 3–6 body lines is ideal; never exceed 10
```

Rules:
- **English only.** `type` is lowercase, one of `[feat] [fix] [hotfix] [refactor] [perf] [docs] [test] [style] [chore]`.
- **Title ≤ 72 chars including the `[type]` prefix**; description starts with a **capital** letter.
- **Title = intent / why; body = what.** Never a bare `Update X` / `Change Y`.
- One blank line between title and body.
- Pick the type from the change's nature, not the branch. A `feature/` branch can still contain a `[chore]` or `[test]` commit. Full type semantics: `references/commit-spec.md`.

**Example:**

```
[feat] Support cross-subdomain SSO login

- Replace session-based auth with OAuth2 authorization-code flow.
- Add /oauth/callback endpoint and update User model schema.
- Bridge existing sessions on first login.
```

## Step 3 — Commit and verify

Show the user the plan first — the ordered units and their messages — then commit. (If `git-github-flow` invoked you, it already gated the plan with the user; don't ask for approval again.)

```bash
git add src/notifications.py src/app.py
git commit -m "$(cat <<'EOF'
[feat] Support cross-subdomain SSO login

- Replace session-based auth with OAuth2 authorization-code flow.
- Add /oauth/callback endpoint and update User model schema.
EOF
)"
# …repeat per atomic unit…
git status      # must end clean — every intended change committed
```

Use the heredoc form so multi-line messages and special characters survive intact. Never `--amend`/`--force`/`--no-verify` to force things through; if a hook fails, fix the cause.

When done, report what you committed (one line per commit). If branches, pushing, or a PR are needed next, hand back to `git-github-flow`.
