# agent-skills

Personal skills marketplace hosting two skills:

- **[pr-flow](pr-flow)** — end-to-end pull-request workflow: create a PR with a
  Conventional-Commits title, wait for CI, fix failures in a loop, drive a
  GitHub Copilot review to a clean state.
- **[terraform-test](terraform-test)** — write and run Terraform tests using
  the built-in `.tftest.hcl` framework (unit + integration).

## Install

### Any agent (recommended) — [`skills` CLI](https://github.com/vercel-labs/skills)

Works for Claude Code, opencode, Cursor, and 70+ other agents — the CLI reads
this repo's `.claude-plugin/marketplace.json` directly.

```bash
npx skills add matt-FFFFFF/agent-skills          # install to every detected agent
npx skills add matt-FFFFFF/agent-skills -a opencode      # opencode only
npx skills add matt-FFFFFF/agent-skills -a claude-code    # Claude Code only
npx skills add matt-FFFFFF/agent-skills -g                # global instead of project scope
```

Add `-s pr-flow` or `-s terraform-test` to install just one skill. `bunx` works
the same in place of `npx`.

Update later with:

```bash
npx skills update
```

### Claude Code native — plugin marketplace

```
/plugin marketplace add matt-FFFFFF/agent-skills
/plugin install pr-flow@matt-ffffff-skills
/plugin install terraform-test@matt-ffffff-skills
```

Update later with:

```
/plugin marketplace update matt-ffffff-skills
```

### opencode manual (no CLI)

opencode also scans `.claude/skills/` and `.agents/skills/` directly, so a
plain clone + symlink works without any installer:

```bash
git clone https://github.com/matt-FFFFFF/agent-skills ~/src/agent-skills
ln -s ~/src/agent-skills/pr-flow/skills/pr-flow ~/.config/opencode/skills/pr-flow
ln -s ~/src/agent-skills/terraform-test/skills/terraform-test ~/.config/opencode/skills/terraform-test
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
