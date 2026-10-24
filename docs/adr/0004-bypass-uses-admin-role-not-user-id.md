# Bypass is granted to the repository-admin role, not to a user

Every bypassable ruleset grants its bypass to `actor_type: RepositoryRole` with `actor_id: 5` — the built-in "Repository admin" role — rather than to a specific `User`. The `bypass_mode` value differs by variant: `always` in the escape-hatch reference, `pull_request` in the PR-first reference (see ADR 0007).

A user id would pin the escape hatch to one account: it stops matching if the repository is transferred, if an administrator later needs to unblock a bot-authored pull request, and it cannot be copied verbatim into anyone else's repository, since the id is not theirs. `RepositoryRole` survives all three.

The trade-off is deliberate and accepted: a new administrator of a repository using these references silently inherits the escape hatch. That is the correct default for repositories maintained by their owner, where whoever holds admin is by definition the person who needs to unblock a stuck merge. Locking the hatch to a single user id is a legitimate hardening step for repositories with more than one administrator, but it is a per-repository decision, not part of the reference rulesets.

`OrganizationAdmin` is not an option: it does not apply to personal repositories.
