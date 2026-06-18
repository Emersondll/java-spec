# 23 - Git Branching Strategy & GitHub Workflow

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory branching and PR workflow for all microservices.
> No developer — regardless of seniority — works directly on `main`.

---

## 1. Core Rule — NEVER Push Directly to `main`

- `main` is **always** production-ready and deployable
- Direct push to `main` is **FORBIDDEN** — enforced via GitHub branch protection
- Every change goes through a **Pull Request** with at least 1 approval
- CI pipeline must be green before any merge

| What is forbidden | What to do instead |
|---|---|
| `git push origin main` | Open a PR from your feature branch |
| `git commit --amend` on a pushed branch | Open a new commit |
| Force-push (`--force`) to any shared branch | Never — fix forward |
| Working directly in the `main` working tree | Always create a branch first |

---

## 2. Branch Naming Convention

Format: `<type>/<ticket>-short-description`

| Type | Pattern | Example |
|---|---|---|
| Feature | `feat/<ticket>-short-description` | `feat/US-123-user-registration` |
| Bug fix | `fix/<ticket>-short-description` | `fix/BUG-456-email-validation` |
| Refactoring | `refactor/<ticket>-short-description` | `refactor/US-789-extract-user-service` |
| Test | `test/<ticket>-short-description` | `test/US-321-add-unit-tests` |
| Docs | `docs/<ticket>-short-description` | `docs/US-654-update-api-spec` |
| Hotfix | `hotfix/<ticket>-short-description` | `hotfix/BUG-999-payment-crash` |
| Release | `release/<version>` | `release/1.2.0` |

**Rules:**
- All lowercase, hyphens only (no underscores, no spaces, no uppercase)
- Always include the ticket or issue reference
- Short description: max 5 words after the ticket
- Branch life: max 1 week — split the work if it takes longer

---

## 3. Standard Workflow

```
main
  └── feat/US-123-user-registration
        ↓ (local: mvn verify passes)
        ↓ (git push origin feat/US-123-user-registration)
        ↓ (open PR → main)
        ↓ (CI runs: mvn verify + JaCoCo + SpotBugs)
        ↓ (1 approval required)
        ↓ (Squash and Merge → main)
        ↓ (delete branch)
```

### Step-by-step commands

```bash
# 0. Ensure local git identity is set correctly (run once per repo, do NOT change global config)
git config user.name "Emerson Lima"
git config user.email "emersondll@outlook.com"

# 1. Always start from updated main
git checkout main
git pull origin main

# 2. Create feature branch
git checkout -b feat/US-123-user-registration

# 3. Work and commit frequently (Conventional Commits)
git add src/main/java/com/company/...
git commit -m "feat(user): add POST /users registration endpoint"

git add src/test/java/com/company/...
git commit -m "test(user): add unit and integration tests for registration"

# 4. Keep branch up-to-date with main (rebase — never merge)
git fetch origin
git rebase origin/main

# 5. Run the full verification suite locally before pushing
mvn verify

# 6. Push to remote
git push origin feat/US-123-user-registration

# 7. Open a Pull Request on GitHub → target: main
#    Fill the PR template (see Section 5)
```

> **Why rebase and not merge?**
> `git rebase origin/main` rewrites your branch on top of the latest main, producing a linear history.
> `git merge origin/main` creates an extra merge commit that pollutes the log and makes `git bisect` harder.

---

## 4. Commit Messages — Conventional Commits

Format: `<type>(<scope>): <short description>`

| Type | When to use |
|---|---|
| `feat` | New feature or endpoint |
| `fix` | Bug fix |
| `refactor` | Code restructuring with no behavior change |
| `test` | Adding or updating tests |
| `docs` | Documentation changes (openapi.yaml, README, spec) |
| `chore` | Build, dependencies, CI config, version bumps |

**Rules:**
- All in **English**
- Lowercase type and scope
- Imperative mood: "add" not "added", "fix" not "fixed"
- Subject line: max 72 characters
- Body (optional): explain **WHY**, not what — the diff shows what
- **NEVER** include `Co-Authored-By:` with any AI tool name

### Good commit examples

```
feat(auth): add JWT validation in GatewayAuthFilter
fix(order): prevent duplicate creation when Idempotency-Key replayed
refactor(user): extract email validation to UserValidator class
test(payment): add integration test covering refund with TestContainers
chore(deps): upgrade spring-boot-starter-parent to 4.0.7
docs(openapi): add Idempotency-Key header to POST /payments
```

### Bad commit examples

```
fix                         ← no scope, no description
updated stuff               ← not imperative, too vague
WIP                         ← work-in-progress commits must be squashed before PR
fixed the bug               ← past tense
feat: Add User Registration  ← uppercase first letter
```

---

## 5. Pull Request Rules

### PR Title

Same format as a commit message:
```
feat(user): add registration endpoint with email uniqueness check
```

### PR Description Template

```markdown
## What
Short description of the change (1–3 sentences).

## Why
Business or technical motivation — why this change is needed now.

## How
Key implementation decisions, trade-offs made, patterns used.

## Checklist
- [ ] `mvn verify` passes locally
- [ ] JaCoCo coverage >= 80% overall (services >= 90%)
- [ ] No `@Autowired` field injection — constructor only
- [ ] All DTOs/requests/responses are `record` types
- [ ] `openapi.yaml` updated if any endpoint was added or changed
- [ ] No secrets, passwords, or tokens in code or comments
- [ ] All commits follow Conventional Commits format
- [ ] Branch rebased on latest `main`
- [ ] PR < 400 lines changed (or reason documented above)
```

### PR Size Guidelines

| Lines changed | Guideline |
|---|---|
| < 200 | Ideal — fast to review |
| 200–400 | Acceptable — add extra context in description |
| > 400 | Split into multiple PRs — reviewers lose focus |
| > 800 | Blocked — must be split before review |

### Merge Strategy

**Squash and Merge** (only allowed strategy):
- All commits from the branch squash into one commit on `main`
- The squash commit message = PR title (Conventional Commit format)
- Branch deleted immediately after merge

**Not allowed:**
- Merge Commit (`--no-ff`) — pollutes history
- Rebase Merge — rewrites contributor identity

---

## 6. Branch Protection Rules (GitHub Settings)

Configure on `main` under: **Settings → Branches → Add rule**

| Setting | Value |
|---|---|
| Branch name pattern | `main` |
| Require a pull request before merging | ✅ enabled |
| Required number of approvals | `1` minimum |
| Dismiss stale reviews when new commits pushed | ✅ enabled |
| Require review from code owners | ✅ enabled (if CODEOWNERS configured) |
| Require status checks to pass before merging | ✅ enabled |
| Required status checks | `build`, `test`, `coverage` |
| Require branches to be up to date before merging | ✅ enabled |
| Require conversation resolution before merging | ✅ enabled |
| Do not allow bypassing the above settings | ✅ enabled (applies even to admins) |
| Allow force pushes | ❌ disabled |
| Allow deletions | ❌ disabled |

### CODEOWNERS example

```
# .github/CODEOWNERS
# Every PR requires review from Emerson
* @Emersondll

# API spec changes require explicit approval
src/main/resources/static/openapi.yaml @Emersondll
spec/ @Emersondll
```

---

## 7. Hotfix Flow

For critical production bugs that require an immediate fix:

```bash
# 1. Branch from main — NOT from a feature or develop branch
git checkout main
git pull origin main
git checkout -b hotfix/BUG-999-payment-null-crash

# 2. Apply the minimal fix — no scope creep
# Fix, add a regression test, then:
mvn verify

git commit -m "fix(payment): prevent NPE when cart items list is null"

# 3. Push and open PR → main (fast-track review — still requires 1 approval)
git push origin hotfix/BUG-999-payment-null-crash
```

**Hotfix rules:**
- Still requires at least 1 PR approval — no "emergency bypass"
- Must include a regression test proving the fix
- PR description must include: what broke, when, impact, fix
- After merge, tag immediately (see Section 8)

---

## 8. Release Tagging

Every merge to `main` that produces a deployable release MUST be tagged:

```bash
# After the PR is merged
git checkout main
git pull origin main

# Annotated tag with message
git tag -a v1.2.0 -m "release: order-service v1.2.0 — adds SAGA compensation flow"
git push origin v1.2.0
```

**Tag naming:** `v<major>.<minor>.<patch>` — SemVer

| Change | Bump |
|---|---|
| Breaking API change | Major: `v2.0.0` |
| New feature, backwards-compatible | Minor: `v1.3.0` |
| Bug fix or patch | Patch: `v1.2.1` |
| Hotfix | Patch: `v1.2.1` |

---

## 9. Anti-Patterns

| Forbidden | Correct approach |
|---|---|
| `git push origin main` | Open a PR from a branch |
| `git commit -m "fix"` | `git commit -m "fix(user): resolve null email on registration"` |
| Branches alive > 1 week | Break into smaller PRs and merge often |
| `git merge origin/main` on feature branch | `git rebase origin/main` |
| PR with 2000+ lines changed | Split into multiple focused PRs |
| Secrets or credentials in any commit | Rotate immediately; use env vars / Vault |
| `Co-Authored-By: Claude` or AI tool in commit | Remove before push |
| Squashing history post-merge manually | Let GitHub Squash-and-Merge handle it |
| Committing `target/`, `.env`, `*.log` files | Add to `.gitignore` before first commit |
| `git push --force` on a shared branch | Never — open a new commit |

---

## 10. `.gitignore` Mandatory Entries

Every microservice MUST include at minimum:

```gitignore
# Build output
target/
*.class

# IDE
.idea/
*.iml
.vscode/
.eclipse/

# Environment and secrets — NEVER commit these
.env
.env.*
*.env
application-secret.yml
*.key
*.p12
*.jks

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/
```

---

## 11. Per-Branch Checklist

Before opening a PR:

- [ ] Branch created from updated `main` (`git pull origin main` immediately before branching)
- [ ] Branch name follows convention: `feat/US-123-short-description`
- [ ] All commits follow Conventional Commits (type(scope): description)
- [ ] `mvn verify` is green locally
- [ ] JaCoCo report checked: overall >= 80%, services >= 90% (`target/site/jacoco/index.html`)
- [ ] No `System.out.println`, `e.printStackTrace()`, or debug code left in
- [ ] No hardcoded secrets, passwords, or API keys
- [ ] `openapi.yaml` updated if any endpoint was added, changed, or removed
- [ ] PR description filled: What / Why / How / Checklist
- [ ] PR < 400 lines changed (or documented reason in description)
- [ ] Branch rebased on latest `main` (`git rebase origin/main`) before final push

---

## Related Specs

| Spec | Relationship |
|---|---|
| `24-CICD-SECRETS.md` | CI/CD pipeline triggered by PR merge — what runs after a branch merges to main |
| `20-MAVEN-POM.md` | What `mvn verify` runs: JaCoCo gate, SpotBugs, Failsafe, Checkstyle |
| `13-OPENAPI-REST-REFERENCE.md` | `openapi.yaml` must be updated before PR if endpoints changed |
| `14-SECURITY.md` | Secrets management rules that apply to commits and branches |

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
