# Pull Request

## Summary

Describe the plugin, skill, manifest, or documentation change.

## Validation

- [ ] `bin/validate-skill-frontmatter`
- [ ] `bin/validate-plugin-metadata`
- [ ] `bin/check-hardcoded-paths`
- [ ] `prek run --all-files` or `pre-commit run --all-files`

## Checklist

- [ ] Updated `SKILL.md` frontmatter when adding or renaming skills
- [ ] Kept `.claude-plugin/marketplace.json` and plugin `.claude-plugin/plugin.json` metadata in
      sync
- [ ] Avoided hardcoded local filesystem paths such as `/Users/...` or `/home/...`
- [ ] Added or updated docs for behavior changes
