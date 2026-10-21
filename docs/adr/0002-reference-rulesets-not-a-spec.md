# The rulesets are references, not a specification to copy

The JSON files under `ruleset-examples/` are **reference rulesets**: shapes that have actually been used, offered so another repository can adopt the parts that fit. They are not a mandate, and no repository is expected to apply all of them.

This is why the README states desired end states and omits execution steps. Readers of this repository — humans and coding agents alike — already know how to create a ruleset; what they cannot reconstruct is the *target shape* and the reasoning behind it. Commands also rot: the REST API's re-apply semantics are undocumented, `gh` has no `ruleset` subcommand, and any `curl`/`gh api` recipe written here would be wrong within a release or two. Recording the outcome is stable; recording the invocation is not.

## Considered Options

- **Executable as-code** (Terraform, or an `apply.sh` that deletes and re-creates rulesets). Rejected for now: it duplicates the repository's purpose, carries its own maintenance burden, and the three-way drift diff it would buy is already tracked as a future option in the README.
- **A specification every repository must reach.** Rejected: these are one person's preferences across repositories with very different shapes — some solo with direct pushes, some with collaborators and CI.

## Consequences

Consumers own the gap between a reference ruleset and their own repository. Because no execution guidance is given, a reference ruleset's parameters must be self-describing: a field whose value only makes sense for the repository it was captured from has to be flagged where the rulesets are described.
