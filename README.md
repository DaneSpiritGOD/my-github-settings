# GitHub Repository Security Baseline

Copy below prompt into your coding agent:

```md
Make an execution plan that follows https://raw.githubusercontent.com/DaneSpiritGOD/my-github-settings/refs/heads/main/README.md. Note that:

1. Do not follow all blindly. The instructions in that file may not suit here. The plan needs to compare pros and cons and give good explanation of our decisions.
2. Do not execute the plan until I say 'go'.
```

## 1. Secret Scanning

| Setting | State | Notes |
|---|---|---|
| Secret scanning | ✅ enabled | Scans full repo history + PRs; opens an alert on known secret patterns |
| Push protection | ✅ enabled | Scans on `git push`; blocks the push when a secret pattern is hit (see below) |

- **UI entry point**: Settings → Code security (direct link: `…/settings/security_analysis`)

---

## 2. GitHub Actions Configuration

| Setting | State | Notes |
|---|---|---|
| Actions SHA pinning | ✅ required | Actions referenced in workflows must be pinned to a commit SHA, not a tag |
| Workflow default permissions | ✅ `read` (read-only) | GITHUB_TOKEN is read-only by default; jobs that write declare permissions explicitly |
| Workflows can approve PRs | ❌ disabled | Prevents a workflow token from approving its own PR |
| Allowed actions | all | No third-party Action allowlist is configured |

Explicit permissions inside the workflow:

```yaml
permissions:
  contents: write   # commit the feed back to main
  issues: write     # open an alarm issue on failure
```

Action reference example (SHA pin + version comment, kept current by Dependabot):

```yaml
- uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4.4.0
```

**Known pitfall**: when merging a PR that modifies files under `.github/workflows/`, the `gh` CLI fails with
`refusing to allow an OAuth App to create or update workflow without 'workflow' scope` (403).
Workaround: merge in the web UI (recommended), or run `gh auth refresh -s workflow` (not recommended).
This is an OAuth token scope issue, not a ruleset block.

---

## 3. Rulesets (Branch Governance)

`bypass_actors` is set per ruleset, never per rule, so branch protection is split into two rulesets by **whether a rule's failure is recoverable**. Reference rulesets live in [`ruleset-examples/`](./ruleset-examples); they are shapes that have been used, to adopt from — not a set every repository must apply.

### 3.1 Unbypassable ruleset

| Dimension | Value |
|---|---|
| Target | branch |
| Applied to | `~DEFAULT_BRANCH` |
| Rule 1 | `deletion` — the default branch cannot be deleted |
| Rule 2 | `non_fast_forward` — force-pushes cannot rewrite its history |
| Bypass | none — `bypass_actors: []`, so the rules bind administrators too |

- **UI entry point**: Settings → Rules → Rulesets
- **Design intent**: this rule set is reserved for **irreversible** damage. A deleted branch or a rewritten history cannot be undone by anyone, so no escape hatch is offered.
- **Reference**: [`ruleset-examples/main-unbypassable.json`](./ruleset-examples/main-unbypassable.json)

### 3.2 Bypassable ruleset

| Dimension | Value |
|---|---|
| Target | branch |
| Applied to | `~DEFAULT_BRANCH` |
| Rule 1 | `pull_request` — changes reach the branch through a pull request |
| Rule 2 (optional) | `required_status_checks` — pull requests must pass the named checks |
| Bypass | repository admin role (`actor_id: 5`, `bypass_mode: always`) |

- **UI entry point**: Settings → Rules → Rulesets
- **Design intent**: this rule set covers **recoverable** mistakes — a blocked or malformed pull request can always be fixed — so an administrator gets an escape hatch. That hatch is load-bearing: pull requests opened with the default `GITHUB_TOKEN` trigger no workflows, so no check runs exist on their head commit, and a required check would sit at `Expected` forever.
- **References**: [`main-with-pr.json`](./ruleset-examples/main-with-pr.json) · [`main-with-pr-and-checks.json`](./ruleset-examples/main-with-pr-and-checks.json) — at most one of these; they both describe the same ruleset.

**`required_status_checks` is optional, and its check names are placeholders.** In `main-with-pr-and-checks.json` the contexts are `lint`, `type-check` and `test`; they were captured from one repository's CI jobs and will not exist in another. Replace them with the job names of the repository adopting the reference.

**Both rules only evaluate on pull request merges.** On a repository where maintainers push to the default branch directly, 3.2 protects against nothing that occurs — which is why 3.1 carries the rules that matter there.

### 3.3 How required checks are matched

A required check is a bare string, matched by **exact equality against a check run's `name`** on the head commit. A check run is produced by a **job**, not by a workflow or a run, and its name defaults to the **job id**. Consequences: renaming a job silently disables the requirement forever, with no error — the Merge button simply never enables again; duplicate job names are legal and make the requirement ambiguous; and a job skipped via `if:` reports `conclusion: skipped`, which is **not** success and wedges every pull request.

### 3.4 Versus branch protection

Rulesets are GitHub's recommended replacement for branch protection rules, because they support bypass actors and richer rule combinations.

---

## 4. Dependabot

### 4.1 In-repo declaration (`.github/dependabot.yml`)

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    commit-message:
      prefix: "chore"
    groups:            # bundle all Action updates into one PR
      actions:
        patterns:
          - "*"
```

Traits: checked weekly; grouped commits (multiple Action updates within the group land as one PR instead of many).

### 4.2 Repository-level switches

| Setting | State |
|---|---|
| Dependabot alerts (vulnerability alerts) | ✅ enabled |
| Dependabot security updates (auto-opens fix PRs) | ✅ enabled |

### 4.3 PR specifics (from experience)

- Dependabot's PRs hit the same OAuth workflow scope restriction (see Section 2)
- The repo's workflows pin Actions by SHA (Section 2); Dependabot reads the `# vX.Y.Z` comment next to each pin to know which version to bump to — **the two mechanisms work together**: pinning blocks tampering, the comment lets Dependabot update
- A dependency PR that touches workflow files must be merged via the web UI or after `gh auth refresh -s workflow`

---

## 5. Recommended Additional Configuration

### 5.1 Do now (low cost)

1. **`delete_branch_on_merge: true`** — removes leftover branches
2. **Squash merge only** — linear history, easy to revert
3. **CODEOWNERS** (`.github/CODEOWNERS`) — even solo, explicitly declaring `* @<owner>` auto-requests owner review once collaborators are added later
4. **`.github/pull_request_template.md` / ISSUE_TEMPLATE** — normalize input
5. `.gitignore` hygiene (the first line of defense against leaks): ignore `.env`, `.env.local`, etc.

### 5.2 Worth it when needed (has cost)

1. **Terraform as-code** — worth introducing once there is more than one ruleset or the config must be replicated across repos:
   - `github_repository_ruleset` (Section 3) + the `security_and_analysis` block of `github_repository` (Section 1)
   - Requires a state backend (local state file or Terraform Cloud) + an admin PAT (`security_events` scope)
   - Core value: the three-way diff (desired vs. state vs. remote) **detects manual drift made through the web UI**
2. **Fine-grained PAT instead of a classic PAT** — grant the daily `gh` CLI least privilege instead of the full `repo` + `workflow` scopes
