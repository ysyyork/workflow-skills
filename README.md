# workflow-skills

Claude Code plugin marketplace for workflow skills.

## Plugins

- **workflow**: `code_feature_builder`, `rebase-guide`, `babysit-pr`

## Install

Add this repo as a marketplace, then install the plugin:

```sh
claude plugin marketplace add ysyyork/workflow-skills
claude plugin install workflow@workflow-skills
```

Use `/workflow:code_feature_builder`, `/workflow:rebase-guide`, and `/workflow:babysit-pr`.

## Notes

These are generic versions of workflow skills. They assume a git repo with `gh` access. `babysit-pr` also uses the `CronCreate` tool. `code_feature_builder` refers to `/rebase-guide` and `/babysit-pr`, both included here.
