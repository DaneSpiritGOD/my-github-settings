# The default-branch reference uses `~DEFAULT_BRANCH` alone

Reference rulesets target `ref_name.include: ["~DEFAULT_BRANCH"]` and nothing else. Anything more specific would be a claim about a particular repository's branch topology, which a reference ruleset cannot make.

Combining `~DEFAULT_BRANCH` with a literal `refs/heads/main` adds no protection on a repository whose default branch is `main` (they resolve to the same ref) and quietly splits intent on one whose default branch is not, where it appears to protect a `main` that may not exist. Repositories that need a specific branch — or several — change this one line, which is the correct place for that decision to be made.
