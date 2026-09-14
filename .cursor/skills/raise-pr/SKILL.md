---
name: raise-pr
description: >-
  Run the full raise-PR workflow: inspect git state, create a feature branch if
  needed, commit all changes, push, merge develop, resolve conflicts, and open a
  GitHub PR with gh. Use when the user says "raise PR", "create PR", "open PR",
  "push and PR", "push and raise PR", "ship this as a PR", or "create branch".
---

# Raise PR Workflow (Mandatory)

When the user asks to **"raise PR"** (or "create PR", "open PR", "push and raise PR", "ship this as a PR"), run the full sequence below. Do **not** stop after committing, and do **not** ask "should I commit first?" — the phrase "raise PR" is explicit consent to `git add` + `git commit` all current changes.

> This overrides the general "only commit when I explicitly ask" rule **for this phrase only**. Any other prompt still requires explicit commit permission.

**Integration branch for this repo is `develop`** (not `main`). PRs target `develop` unless the user says otherwise.

---

## Sequence

### 1. Inspect state
```bash
git status --porcelain
git rev-parse --abbrev-ref HEAD
git diff --stat
```
Report the current branch and the file count you are about to commit before proceeding.

### 2. Branch first if on a base branch
| Current branch | Action |
|---|---|
| Feature branch (`feature/*`, `fix/*`, `docs/*`) | Stay on it |
| `develop`, `main`, `master` | `git checkout -b feature/<kebab-slug>` before committing — derive the slug from the actual change, never an unrelated name |

Uncommitted changes carry over to the new branch. Never leave a commit sitting on local `develop`/`main`.

### 3. Stage + commit everything
```bash
git add -A
git commit -m "$(cat <<'EOF'
<type>: <concise why-focused summary>
EOF
)"
```
- Commit **all** changes shown in Changes, including untracked files (new `supabase/migrations/*.sql`, new components).
- **Never** stage `.env`, `.env.*`, `config/live.public-env`, or `config/dev.public-env`. If `git status` shows them, warn the user and exclude them explicitly.
- Nothing to commit → skip to step 4, do not create an empty commit.

### 4. Push the branch
```bash
git push -u origin HEAD
```

### 5. Refresh `develop`
```bash
git checkout develop
git pull --ff-only origin develop
```

### 6. Merge `develop` into the feature branch
```bash
git checkout -
git merge develop
```

### 7. Resolve conflicts
- If the merge is clean, continue to step 8.
- If there are conflicts: list every conflicted file for the user, then resolve them by **keeping both sides' intent** — never blindly take `--ours` or `--theirs`.
- Special care:
  - `supabase/migrations/` — never edit an already-pushed migration; keep both files, rename yours to a later timestamp if ordering collides.
  - `src/integrations/supabase/types.ts` — regenerate rather than hand-merge if the conflict is large.
  - `package-lock.json` — take `develop`'s version, then re-run `npm install`.
- After resolving: `npx tsc --noEmit` must pass before committing the merge.
```bash
git add -A
git commit --no-edit
```

### 8. Re-push
```bash
git push
```

### 9. Open the PR
```bash
gh pr create --base develop --title "<title>" --body "$(cat <<'EOF'
## Summary
- <what changed and why>

## Test plan
- [ ] npx tsc --noEmit
- [ ] npm run lint
- [ ] <manual flow to verify>
EOF
)"
```
Return the PR URL. If `gh` is unauthenticated or the network is blocked, say so plainly and report that the branch is pushed and PR-ready, with the exact `gh` command to run manually.

---

## "Create branch" (standalone request)

Only steps 1 and 2: report state, create `feature/<kebab-slug>`, confirm. **Do not commit** — leave the working tree dirty. Commits still require explicit permission for this phrase.

---

## Hard rules

- Never `git push` to `develop`, `main`, or `master`.
- Never `--force` push unless the user explicitly asks.
- Never `--no-verify` or `--amend` a pushed commit.
- Never commit env files or secrets.
- Never abandon the sequence midway silently — if a step fails, report which step, the error, and the recovery option.
