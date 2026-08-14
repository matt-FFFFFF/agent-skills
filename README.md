# claude-skills

Personal Claude Code plugin marketplace hosting two skills:

- **[pr-flow](pr-flow)** — end-to-end pull-request workflow: create a PR with a
  Conventional-Commits title, wait for CI, fix failures in a loop, drive a
  GitHub Copilot review to a clean state.
- **[terraform-test](terraform-test)** — write and run Terraform tests using
  the built-in `.tftest.hcl` framework (unit + integration).

## Install

```
/plugin marketplace add matt-FFFFFF/claude-skills
/plugin install pr-flow@matt-ffffff-skills
/plugin install terraform-test@matt-ffffff-skills
```

Update later with:

```
/plugin marketplace update matt-ffffff-skills
```

## Layout

Each top-level directory is a self-contained plugin:

```
<plugin>/
  .claude-plugin/
    plugin.json      # plugin manifest
  skills/
    <plugin>/
      SKILL.md        # skill definition, auto-discovered
```

The marketplace catalog lives at `.claude-plugin/marketplace.json`.
