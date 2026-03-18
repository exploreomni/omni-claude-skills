# Contributing Guidelines

This repository is a Claude Code plugin marketplace. Keep plugin metadata, skill docs, and
repository documentation consistent so changes remain reviewable and safe to publish.

## Recommended Plugin

Anthropic's `plugin-dev@claude-plugins-official` plugin is useful when editing manifests, skills,
hooks, and other plugin structure files.

## Skill Requirements

- Store each skill at `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`.
- Add YAML frontmatter at the top of every `SKILL.md` with non-empty `name` and `description`
  fields.
- Keep skill directory names and frontmatter `name` values in kebab-case and in sync.
- Write descriptions that explain when the skill should be used, not just what it is called.

## Metadata Requirements

- Keep `.claude-plugin/marketplace.json` and each plugin's `.claude-plugin/plugin.json` aligned on
  plugin `name` and `version`.
- Ensure each marketplace `source` points to the plugin directory that contains the manifest.
- Do not commit hardcoded local filesystem paths such as `/Users/...` or `/home/...`.

## Quality Checks

Run the repository checks before opening a pull request:

```bash
bin/validate-skill-frontmatter
bin/validate-plugin-metadata
bin/check-hardcoded-paths
prek run --all-files
# or
pre-commit run --all-files
```

Install either runner locally:

- `prek`: `brew install prek && prek install`
- `pre-commit`: `python3 -m pip install pre-commit && pre-commit install`

## Pull Requests

- Keep changes scoped to the plugin, skill, or documentation update you are making.
- Include documentation updates when behavior, installation, or contributor workflow changes.
- Use the pull request template and note any follow-up work or remaining limitations.
