# `pull_request` bypass does not unlock merge methods excluded by `allowed_merge_methods`

A bypass actor's `bypass_mode` controls which rules may be bypassed, not how a merge is performed. `pull_request` lets an administrator merge a pull request despite missing or red required checks, but the merge itself is still bound by the `pull_request` rule's own `allowed_merge_methods`: a method excluded there is rejected even while bypassing. Only `bypass_mode: always` unlocks an excluded method.

Verified by a live experiment on `DaneSpiritGOD/md` (ruleset 25528972): with `allowed_merge_methods: ["squash"]` and a `pull_request` bypass, a merge-commit merge is rejected while bypassing; the squash merge succeeds.

## Consequences

- The PR-first preference (`bypass_mode: pull_request` + squash-only) is strict for the owner too: there is no way to land one special PR type (e.g. an upstream-sync PR) as a merge commit without holding `always`.
- To allow one special PR type to use a merge commit while everything else stays squash-only, a separate GitHub App with `always` bypass plus a land workflow is needed. Deferred on `md` (issue #34).
