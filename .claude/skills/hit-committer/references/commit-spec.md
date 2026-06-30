# Hit commit-message spec

The authoritative commit-message rules for `hit-committer` (from the Hit manual §2.3). `github-flow` defers to this skill for the commit step, so this file is the single source of truth for the format.

## Format

```
[type] Imperative intent — WHY (first letter capitalized)

- Module-level WHAT changed (no function-by-function detail)
- 3–6 lines ideal, ≤10 max
```

- English only. `type` lowercase.
- Title ≤ 72 chars incl. `[type]`. Blank line before the body.
- **Title = intent / why, body = what.** Never a bare `Update X` / `Change Y` / `Modify Z`.

## Type semantics

| Type | Meaning |
|---|---|
| `[feat]` | Outward-visible new feature; changes behavior a user or caller can observe. |
| `[fix]` | Bug fix; non-urgent correction of behavior that didn't match expectations (normal feature/bug flow, not a production hotfix). |
| `[hotfix]` | Urgent fix to an already-shipped `release/v<X.Y.Z>`, and its cross-version `port/*` propagation (cherry-pick keeps the original message, so its type stays `[hotfix]`). |
| `[refactor]` | Internal restructuring with **no** outward behavior change. |
| `[perf]` | Performance optimization with no behavior change (if it also adds a feature, split that into a separate `[feat]`). |
| `[docs]` | Docs / comments / README only; no product code. |
| `[test]` | Test code only; no product code. |
| `[style]` | Formatting only (whitespace, indent, formatter) — no logic change. |
| `[chore]` | None of the above: version bumps, dependency upgrades, CI/build config, file moves. |

## Picking the type

Choose from the change itself, independent of the branch it sits on — a `feature/` branch may legitimately contain a `[test]` or `[chore]` commit. When a single unit genuinely spans two types (e.g. a feature plus an unrelated formatting sweep), that's a sign it should be **two** commits.

## Atomicity

One logical, self-contained change per commit. Group with explicit `git add <paths>`; split intra-file concerns with `git add -p`. Unrelated changes never share a commit, even when they're in the working tree at the same time.

## Examples

```
[feat] Support cross-subdomain SSO login

- Replace session-based auth with OAuth2 authorization-code flow.
- Add /oauth/callback endpoint and update User model schema.
- Bridge existing sessions on first login.
```

```
[fix] Keep the last item on each page in paginate

- Make the slice end exclusive so the final element is no longer dropped.
```

```
[hotfix] Prevent v1.1.0 boot crash on missing device_profile

- Read device_profile defensively so a missing key boots with a default
  instead of raising KeyError.
```
