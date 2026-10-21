# Split branch rulesets by bypass intent

`bypass_actors` is a ruleset-level field. GitHub has no way to say "these rules bind everyone, those rules have an escape hatch" inside a single ruleset. So branch protection on the default branch is expressed as two rulesets:

- an **unbypassable ruleset** holding only rules that guard against unrecoverable damage (`deletion`, `non_fast_forward`), with `bypass_actors: []`;
- a **bypassable ruleset** holding the workflow rules (`pull_request`, and optionally `required_status_checks`), carrying a repository-admin bypass.

The split is driven by reversibility, not by rule category: a deleted branch or a rewritten history cannot be undone by anyone, so no one may bypass those rules — including administrators. A blocked or malformed pull request is recoverable, so it gets an escape hatch.

## Consequences

The escape hatch is load-bearing, not a convenience. Pull requests opened by a workflow using the default `GITHUB_TOKEN` do not trigger other workflows (GitHub's anti-recursion rule), so no jobs run and no check runs are created. Any required check context on that head SHA then resolves to `Expected` forever, and an administrator bypassing the ruleset is the only way to merge. Any repository with bot-authored pull requests will hit this.

Because bypass is per-ruleset, adding a bypass to the unbypassable ruleset to "simplify things" silently destroys the whole model.
