# GitHub Repository Security Baseline

> Raw file (copy-friendly): https://raw.githubusercontent.com/DaneSpiritGOD/my-github-settings/refs/heads/main/README.md

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

### The only ruleset: `Only <repo-owner> can modify main` (ruleset id `<ruleset-id>`, active)

| Dimension | Value |
|---|---|
| Target | branch |
| Applied to | `refs/heads/main` |
| Rule 1 | `deletion` — main cannot be deleted |
| Rule 2 | `non_fast_forward` — force-pushes cannot rewrite main's history |
| Bypass | `<owner>` (user id `<owner-user-id>`) only, `bypass_mode: always` |

- **UI entry point**: Settings → Rules → Rulesets (direct link: `…/settings/rules`)
- **Design intent**: stop outsiders and CI from deleting the branch or rewriting history; the owner is an always-bypass actor, so for the owner it is only a reminder
- **Versus branch protection**: Rulesets are GitHub's officially recommended replacement — they support bypass actors and richer rule combinations.

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

1. **`delete_branch_on_merge: true`** — one API call, removes leftover branches
2. **Squash merge only** — linear history, easy to revert
3. **CODEOWNERS** (`.github/CODEOWNERS`) — even solo, explicitly declaring `* @<owner>` auto-requests owner review once collaborators are added later
4. **`.github/pull_request_template.md` / ISSUE_TEMPLATE** — normalize input
5. Keep `allow_force_push` off
6. `.gitignore` hygiene (the first line of defense against leaks): ignore `.env`, `.env.local`, etc.

### 5.2 Ruleset enhancements

1. **Add a `pull_request` rule to the ruleset**: required reviews / required CI check (e.g. Refresh feed must pass before merge) — optional for a solo repo
2. **Tags ruleset**: `target: push` to protect tags from deletion/retargeting, guarding against SHA pinning being bypassed (pinning's threat model is precisely that "tags can be re-pointed"; we pin SHAs, but tag governance is still good hygiene)

### 5.3 Worth it when needed (has cost)

1. **Terraform as-code** — worth introducing once there is more than one ruleset or the config must be replicated across repos:
   - `github_repository_ruleset` (Section 3) + the `security_and_analysis` block of `github_repository` (Section 1)
   - Requires a state backend (local state file or Terraform Cloud) + an admin PAT (`security_events` scope)
   - Core value: the three-way diff (desired vs. state vs. remote) **detects manual drift made through the web UI**
2. **Fine-grained PAT instead of a classic PAT** — grant the daily `gh` CLI least privilege instead of the full `repo` + `workflow` scopes
