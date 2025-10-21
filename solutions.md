# Pull Request (PR) Guide

This document covers:

1. **Steps to create a PR** (GitHub UI & CLI)
2. **Best practices for writing PR descriptions** (with templates)
3. **How to handle review comments** (workflow, commands, and examples)
4. **Differences between **`** and **` (when to use each)

---

## 1) Steps to Create a PR

### A. Prerequisites

* Ensure your local `main` (or `master`) is up to date:

  ```bash
  git checkout main
  git pull origin main
  ```
* Create a feature branch from the updated default branch:

  ```bash
  git checkout -b feature/<short-topic>
  ```
* Make changes, commit with a meaningful message, and push:

  ```bash
  git add .
  git commit -m "feat: add X to achieve Y"
  git push -u origin feature/<short-topic>
  ```

### B. Create a PR via GitHub Web UI

1. Navigate to your repository on GitHub.
2. You’ll see a prompt: **“Compare & pull request”** for your newly pushed branch — click it. (Or click **Pull requests > New pull request**.)
3. Set **base** = `main` (or the target branch) and **compare** = `feature/<short-topic>`.
4. Fill in the **Title** and **Description** (use the template below).
5. Link issues using **“Closes #****”** in the description.
6. Assign reviewers, labels, and project/milestone if applicable.
7. (Optional) Mark as **Draft** if the work isn’t ready for review.
8. Click **Create pull request**.

### C. Create a PR via CLI (GitHub CLI)

> Install GitHub CLI: [https://cli.github.com/](https://cli.github.com/)

```bash
# authenticate once
gh auth login

# from your repo root on the feature branch
gh pr create \
  --base main \
  --head feature/<short-topic> \
  --title "feat: concise, outcome-focused title" \
  --body-file .github/PULL_REQUEST_TEMPLATE.md \
  --draft
```

Useful subcommands:

```bash
gh pr view --web               # open PR in browser
gh pr status                   # see open PRs & CI state
gh pr ready --undo|--ready     # toggle draft/ready
```

### D. Keep the PR up to date

* Rebase onto latest `main` to keep history clean and avoid merge commits:

  ```bash
  git fetch origin
  git rebase origin/main
  # resolve conflicts if any, then
  git push --force-with-lease
  ```
* Alternatively, if your team prefers merges:

  ```bash
  git merge origin/main
  git push
  ```

---

## 2) Best Practices for Writing PR Descriptions

### A. Principles

* **One PR = One logical change** (small, focused, reviewable in < ~400 lines net).
* **Outcome-first**: Explain *why* and *what* changed before *how*.
* **Traceability**: Link to issues, design docs, incidents, tickets.
* **Testability**: Include how reviewers can verify (steps, test data, screenshots).
* **Risk awareness**: Call out impact, rollbacks, and migration notes.
* **Security & Compliance**: Mention secrets handling, PII, access control changes.
* **Deployment**: Note feature flags, config toggles, and rollout plan.

### B. PR Title Guidelines

* Be concise; use imperative mood (e.g., “add”, “fix”, “refactor”).
* Follow your team’s convention (e.g., Conventional Commits):

  * `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`
* Include scope when helpful: `feat(api): add rate limiting`

### C. PR Description Template (copy/paste)

```
## Summary
Briefly explain the change and the problem it solves. State the desired outcome.

## Context / Links
- Issue: Closes #123
- Design / ADR: <link>
- Related PRs: <links>

## Changes
- [x] What did you change at a high level (bullets)
- [x] Any schema / contract changes

## How to Test
1) Steps to reproduce current behavior (pre-change)
2) Steps to verify new behavior (post-change)
3) Include sample requests, creds scope (if mock), data setup, screenshots/logs

## Risk & Rollout
- Risk level: Low/Medium/High (why)
- Affected components/services: <list>
- Backward compatibility: <yes/no + details>
- Rollout plan: <flag name>, staged % rollout, or cron window
- Rollback plan: <how to revert / toggle off / data restore>

## Security / Privacy
- Secrets/keys: <none | managed via Vault/SM>
- Permissions/roles changed: <details>
- PII/PHI touched: <none | masked | retention impacts>

## Performance
- Expected impact: <latency/memory/cost>
- Load test result summary: <numbers/graphs if applicable>

## Checklist
- [ ] Unit tests added/updated
- [ ] Integration/e2e tests passing
- [ ] CI green; lint & type checks pass
- [ ] Docs updated (README/Runbook/Changelog)
- [ ] Observability (metrics/logs/traces) added or updated
```

---

## 3) Handling Review Comments

### A. Mindset & Etiquette

* Assume positive intent. Reviews are for quality, safety, and shared learning.
* Respond to **every** comment: resolve, clarify, or capture TODOs.
* Prefer discussion to debate; propose options with trade-offs.

### B. Workflow to Address Feedback

1. **Triage**: Label comments as *must-fix*, *nice-to-have*, or *question*.
2. **Group changes**: Batch related fixes into coherent commits.
3. **Make changes** locally and push updates:

   ```bash
   # amend last commit (small fix)
   git add <files>
   git commit --fixup=<SHA-to-fix>   # or: git commit --amend --no-edit
   git rebase -i --autosquash origin/main
   git push --force-with-lease
   ```
4. **Reply** in the PR:

   * “Applied suggestion in 1c9d2f7; added test case.”
   * “Chose option B due to latency; documented in README.”
5. **Mark as resolved** only after pushing the corresponding change or reaching agreement.

### C. Common Scenarios & How to Respond

* **Style/Lint issues**: “Fixed via `make fmt` and `eslint --fix`.”
* **Missing tests**: “Added unit tests for edge cases: empty input, 429, and timeout.”
* **Naming/architecture**: Provide rationale, accept if team standard differs.
* **Breaking change risk**: Add migration note, use feature flag, or add backward-compat layer.
* **Large PR**: Offer to split into multiple smaller PRs or follow-up task list.

### D. Using Suggested Changes (GitHub)

* For small edits, use **“Add suggestion to batch”** → **“Commit suggestions”** (creates a co-authored commit). For larger changes, implement locally.

### E. When You Disagree

* Offer data: benchmarks, references, ADRs.
* If unresolved, request a quick synchronous discussion; document decision in the PR.

### F. Keep History Clean (optional but recommended)

* Use `--fixup/--squash` then autosquash to keep a tidy history tied to review comments:

  ```bash
  git commit --fixup <SHA>
  git rebase -i --autosquash origin/main
  git push --force-with-lease
  ```
* Squash at merge if your team prefers a single commit per PR.

### G. Final Steps to Merge

* Ensure **CI is green** (tests, lint, SAST/DAST, license checks).
* Confirm **approvals** and required **status checks** met.
* **Squash & merge** or **Rebase & merge** per team policy.
* After merge: delete branch, update changelog/release notes.

---

## 4) Difference Between `git reset` and `git revert`

### A. `git reset`

* **Purpose:** Moves the current branch pointer to a previous commit.
* **Effect:** Rewrites history; can remove commits permanently from the current branch.
* **Use case:** Undo local commits *before* they’re pushed.
* **Example:**

  ```bash
  git reset --hard HEAD~1   # completely remove last commit
  git reset --soft HEAD~1   # keep changes staged
  git reset --mixed HEAD~1  # keep changes unstaged
  ```
* **Warning:** Never use `reset --hard` on shared branches like `main` — it rewrites public history.

### B. `git revert`

* **Purpose:** Creates a *new commit* that undoes changes from a previous commit.
* **Effect:** Safe for shared branches — it doesn’t rewrite history.
* **Use case:** Undo commits that are already pushed or merged.
* **Example:**

  ```bash
  git revert <commit-hash>
  git revert HEAD~2..HEAD   # revert multiple commits
  ```

### C. Comparison Table

| Aspect                 | `git reset`                    | `git revert`            |
| ---------------------- | ------------------------------ | ----------------------- |
| History                | Rewrites                       | Preserves               |
| Commit removal         | Deletes commits                | Adds new inverse commit |
| Use case               | Local undo                     | Public undo             |
| Safe on shared branch? | ❌ No                           | ✅ Yes                   |
| Typical scenario       | Fix local mistakes before push | Rollback merged feature |

### D. Summary

* Use `` to undo local work before pushing — keeps history clean.
* Use `` to safely undo pushed commits in shared branches.

---

When to Use git stash

Use git stash when you want to temporarily save your uncommitted changes (both staged and unstaged) without committing them — for example:

You need to switch branches but aren’t ready to commit your current work.

You want to pull or rebase your branch but have local modifications.

You’re experimenting or debugging and want to save your progress before testing something else.

Example:

git stash save "WIP: working on login feature"


This stores your current changes and cleans your working directory.


Difference Between git stash pop and git stash apply
Command	What It Does	Keeps Stash Entry?	When to Use
git stash apply	Applies stashed changes to your working directory	✅ Yes (stash remains in the list)	When you might need to reuse the same stash later
git stash pop	Applies stashed changes and deletes that stash entry	❌ No	When you’re sure you no longer need that stash

Example:

git stash apply stash@{0}   # applies stash and keeps it
git stash pop               # applies stash and removes it


Git Cherry-pick in Bug Fixes
A. How Cherry-picking is Used in Bug Fixes

Purpose: To apply a specific bug fix commit from one branch to another without merging unrelated changes.

Typical use case:

A critical bug is fixed in the dev branch, but the same fix is needed in main or a release branch.

Instead of merging all development commits, you cherry-pick just the fix.

Example:

# Switch to the production branch
git checkout main


# Apply only the bug fix commit from dev
git cherry-pick <commit-hash>

This applies the exact changes from that commit and creates a new commit on main.

Advantages:

Isolates and quickly transfers critical fixes.

Avoids unnecessary merges or extra commits.

Useful in multi-version or long-term support (LTS) projects.

B. Risks of Cherry-picking
Risk	Explanation	Mitigation
Duplicate commits	Each cherry-picked commit gets a new hash, leading to confusion if merged later.	Use tags or notes to track cherry-picked commits.
Merge conflicts	If branches diverge, applying the same change may cause conflicts.	Resolve carefully and test after cherry-pick.
Code inconsistency	The fix might rely on other uncherry-picked commits, causing missing dependencies.	Verify dependencies before cherry-picking.
History divergence	Frequent cherry-picks make branch histories harder to follow.	Use rebase or merges for regular syncs.
Accidental overwrite	If cherry-picking large or old commits, changes can unintentionally overwrite recent code.	Always review diffs (git diff) before applying.

Difference Between Merge and Rebase

git merge

Purpose: Combines changes from one branch into another without altering existing history.

How it works: Creates a new merge commit that joins the histories of both branches.

Example:

git checkout main
git merge feature-branch


Result:

Keeps full history of both branches (shows branching structure).

Easier for collaboration but can make the log more complex.

git rebase

Purpose: Moves or replays commits from one branch on top of another branch to create a linear history.

How it works: Rewrites commit history by applying your commits one by one on the new base branch.

Example:

git checkout feature-branch
git rebase main


Result:

Cleaner, linear history.

Commits get new hashes (history is rewritten).

Not safe for branches that are already pushed/shared.

Comparison Table
Aspect	git merge	git rebase
History style	Preserves full branch structure	Linear (no merge commits)
Commit hashes	Remain the same	Rewritten
New commit created?	Yes (merge commit)	No (reapplies commits)
Safe for shared branches?	✅ Yes	⚠️ No
Preferred for	Collaborative branches	Local history cleanup
⚙️ Best Practices for Rebasing

Rebase only private or local branches.
Don’t rebase public branches (like main) that others are using.

Use git pull --rebase to avoid unnecessary merge commits when syncing:

git pull --rebase origin main


Keep commits atomic and logical.
Each commit should represent a single meaningful change.

Use interactive rebase to clean history:

git rebase -i HEAD~5


Squash, reorder, or edit commits before pushing.

Handle conflicts carefully:

git add <resolved-file>
git rebase --continue


After rebasing, push with caution:

git push --force-with-lease


This avoids overwriting others’ work unintentionally.

Always test after rebasing.
Since history changes, verify everything builds and runs correctly.


Which Strategy Is Best for DevOps and CI/CD

In DevOps and CI/CD environments, both merge and rebase have their place — the choice depends on team workflow, branching model, and automation needs.

✅ Recommended Strategy: Rebase for local work, Merge for shared integration

Use rebase for keeping your feature branches clean and up to date before merging.
It ensures a linear history and makes CI pipelines (build/test/deploy) faster and easier to debug.

Use merge for integrating completed feature branches into main branches (main, develop, etc.)
It preserves context and avoids rewriting history, which is safer for collaborative environments.

Typical workflow:

# Keep your local branch updated
git fetch origin
git rebase origin/main

# Once ready, merge into main via PR
git checkout main
git merge feature-branch


This hybrid approach is the most CI/CD-friendly because:

Rebasing ensures clean, conflict-free PRs for automation tools (like Jenkins, GitHub Actions, GitLab CI).

Merging ensures traceable history for audits, releases, and rollbacks.

 Pros and Cons of Different Workflows
Workflow	Description	Pros	Cons	Best Use Case
Merge-based	Use git merge to combine branches (creates merge commits).	✅ Keeps full history
✅ Safe for teams
✅ Easy to revert	❌ History can get messy
❌ Merge commits add clutter	Shared branches in CI/CD pipelines, production releases
Rebase-based	Use git rebase to linearize commits (no merge commits).	✅ Clean, linear history
✅ Easier to debug CI failures
✅ Ideal for automation scripts	❌ Rewrites history
❌ Risky on shared branches	Local development, preparing clean PRs
Squash merge	Combine all feature commits into a single commit on merge.	✅ Clean history per feature
✅ Easy rollback
✅ Ideal for CI/CD triggers	❌ Loses detailed commit history	Merging PRs in GitHub/GitLab CI/CD
Rebase + Merge (Hybrid)	Rebase locally, then merge into main via PR.	✅ Combines both benefits
✅ Maintains clean history
✅ Safer for CI/CD	⚠️ Requires disciplined workflow	Most modern DevOps teams (GitHub Flow, GitLab Flow)

 Summary:

For CI/CD pipelines: prefer a rebase-then-merge or squash-merge strategy.

For collaboration safety: stick to merge-based workflows for shared branches.

For developer productivity: use rebase locally to keep commits tidy and conflict-free before creating a PR.

*End of document.*

