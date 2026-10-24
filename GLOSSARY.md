# GitHub Repository Settings

A set of GitHub repository governance preferences, expressed as branch ruleset shapes and shared across repositories. It is a reference, not a house standard: each repository adopts the parts that fit its own situation.

## Language

**Reference ruleset**:
A ruleset shape that has actually been used, published so another repository can adopt the parts that fit. It states a target, not an obligation, and no repository is expected to apply every one.
_Avoid_: Preset, default, spec, policy, template

**Unbypassable ruleset**:
A ruleset whose rules bind every actor, including repository administrators, because its `bypass_actors` list is empty. It is reserved for rules that guard against damage nobody can undo.
_Avoid_: Strict ruleset, hard ruleset, locked ruleset

**Bypassable ruleset**:
A ruleset that grants an escape hatch to the repository-admin role. It carries the workflow rules, whose failure is recoverable.
_Avoid_: Soft ruleset, permissive ruleset

**Escape hatch**:
The ability of a repository administrator to merge despite a bypassable ruleset. It exists to unblock pull requests that can never satisfy their required checks — in practice, those opened by a workflow using the default token, which trigger no jobs at all.
_Avoid_: Override, admin bypass shortcut

**Bypass actor**:
An entry in `bypass_actors`, identified by actor type and, for most types, actor id. The whole list is per-ruleset, never per-rule, which is why rules are split across rulesets by whether they may be bypassed.
_Avoid_: Exemption, exception

**Bypass mode**:
The value on a bypass actor that decides how far the escape hatch reaches — `always` (push or merge despite the rules) or `pull_request` (merge a pull request despite the rules, but no direct push to the default branch).
_Avoid_: Bypass level, override mode

**Required check**:
A check-run name a bypassable ruleset demands on a pull request's head commit. It is matched by exact equality against the `name` of a check run, which defaults to the id of the job that produced it — never the workflow name, and never its provenance.
_Avoid_: Status check name, CI requirement

**PR-first landing**:
The practice of landing every change through a pull request, with no direct pushes to the default branch; the owner holds the same rules as everyone else. Its ruleset shape pairs `bypass_mode: pull_request` with squash-only merges.
_Avoid_: PR-only workflow, PR-first policy

**Squash-only**:
A merge policy that allows only squash merges, expressed as `allowed_merge_methods: ["squash"]` on the `pull_request` rule.
_Avoid_: Squash merges only, linear-history merges
