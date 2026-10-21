# `required_status_checks` is optional, and its check names are per-repository

Requiring status checks is treated as an optional addition rather than part of the baseline branch protection, for two independent reasons.

First, rules are only evaluated when a pull request is merged. On a repository where maintainers push to the default branch directly, `required_status_checks` and `pull_request` are inert — they protect against nothing that occurs there.

Second, required checks are matched by **exact equality against a check run's `name`**, which defaults to the job id of the job that produced it. Workflow names are irrelevant; provenance is never considered. The contexts therefore cannot be made generic: they are whatever the target repository's jobs are called. A ruleset that requires a job the repository does not run leaves every pull request permanently blocked, and a job that is skipped via `if:` reports `conclusion: skipped`, which is not success and wedges merges the same way.

## Consequences

The reference ruleset that includes `required_status_checks` carries check names captured from another repository. They are placeholders, and the README says so. Renaming a job later silently breaks the requirement — GitHub reports no error, the Merge button simply never enables again. Duplicate job names are legal and make the requirement ambiguous, since the first successful match satisfies it.
