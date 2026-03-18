# Omni Analytics Plugin for Claude Code

A Claude Code plugin that helps analytics engineers and data teams work with
[Omni Analytics](https://omni.co) programmatically through Omni's REST APIs.

## Installation

### From Marketplace (recommended)

```bash
/plugin marketplace add exploreomni/omni-claude-skills
/plugin install omni-analytics@omni-analytics
```

### From Git URL

```bash
/plugin marketplace add https://github.com/exploreomni/omni-claude-skills.git
/plugin install omni-analytics@omni-analytics
```

### Updating

Enable plugin auto-updates:

1. Run /plugin
2. Go to Marketplaces
3. Select the marketplace → Enable auto-update

Alternatively, update the plugin manually from the /plugin menu or reinstall it to ensure you’re
using the latest version.

## Recommended Plugins

Anthropic's `plugin-dev@claude-plugins-official` plugin is useful when working on plugin manifests,
skills, hooks, and marketplace structure in this repository.

## Local Checks Setup

You can run the repository checks locally with either `prek` or `pre-commit`.

### Option 1: prek

```bash
brew install prek
prek install
prek run --all-files
```

### Option 2: pre-commit

```bash
python3 -m pip install pre-commit
pre-commit install
pre-commit run --all-files
```

CI uses `prek`, but the repository hook configuration is compatible with both.

## Setup

After installation, set these environment variables:

```bash
export OMNI_BASE_URL="https://yourorg.omniapp.co"
export OMNI_API_KEY="your-api-key"
```

API keys are created in **Settings > API Keys** (Organization Admin) or
**User Profile > Manage Account > Generate Token** (Personal Access Token for Modeler/Connection
Admin roles).

> **Token security**: These tokens appear in terminal scrollback when used in shell commands. For
> team deployments, consider loading tokens from a secrets manager or wrapping API calls in an MCP
> server. See `omni-admin` for details.

## Skills

This plugin includes 8 skills that Claude activates automatically based on your request:

| Skill | Description |
|-------|-------------|
| **omni-model-explorer** | Discover and inspect models, topics, views, fields, and relationships. Includes field impact analysis. |
| **omni-model-builder** | Create and edit views, topics, dimensions, measures, and relationships in YAML |
| **omni-query** | Run queries against Omni's semantic layer and interpret results |
| **omni-content-explorer** | Find, browse, and organize dashboards, workbooks, and folders |
| **omni-content-builder** | Create, update, and manage documents and dashboards programmatically — lifecycle, tiles, filters, layouts |
| **omni-ai-optimizer** | Optimize your Omni model for Blobby (Omni's AI assistant) |
| **omni-admin** | Manage connections, users, groups, permissions, schedules, and token security |
| **omni-embed** | Embed dashboards in external applications — URL signing, custom themes, iframe events |

## Usage

Just ask Claude naturally — skills activate automatically:

```text
"What topics are available in our Omni model?"          → omni-model-explorer
"Run a query showing revenue by month"                  → omni-query
"Add a new dimension for customer tier to the users view" → omni-model-builder
"Find the dashboard about sales performance"            → omni-content-explorer
"Build a dashboard showing revenue by month"             → omni-content-builder
"Improve the AI context on our orders topic"            → omni-ai-optimizer
"Give the marketing team access to the sales dashboard" → omni-admin
```

## Team Deployment

To make this plugin available to your entire team automatically, add it to your project's
`.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "omni-analytics": {
      "source": {
        "source": "github",
        "repo": "exploreomni/omni-claude-skills"
      }
    }
  },
  "enabledPlugins": ["omni-analytics@omni-analytics"]
}
```

When team members trust the repository folder, Claude Code automatically installs the marketplace
and plugin.

## Documentation

- [Omni REST API Reference](https://docs.omni.co/api.md)
- [Omni Modeling Documentation](https://docs.omni.co/modeling.md)
- [Omni AI Optimization Guide](https://docs.omni.co/ai/optimize-models.md)
- [Omni MCP Server](https://docs.omni.co/ai/mcp.md) (complementary — this plugin uses the REST API
  directly)
- [Claude Code Plugin Docs](https://code.claude.com/docs/en/plugins)

## Repository Structure

```text
omni-claude-skills/
├── .github/
│   ├── workflows/
│   │   └── ci-lint.yml               ← GitHub Actions lint checks
│   ├── dependabot.yml                ← GitHub Actions update automation
│   └── PULL_REQUEST_TEMPLATE.md      ← Contributor PR checklist
├── .claude-plugin/
│   └── marketplace.json              ← Marketplace catalog
├── bin/
│   ├── validate-skill-frontmatter    ← Validates SKILL.md YAML frontmatter
│   ├── validate-plugin-metadata      ← Checks marketplace/plugin manifest consistency
│   └── check-hardcoded-paths         ← Scans plugin files for local absolute paths
├── plugins/
│   └── omni-analytics/
│       ├── .claude-plugin/
│       │   └── plugin.json           ← Plugin manifest
│       └── skills/
│           ├── omni-model-explorer/
│           │   └── SKILL.md
│           ├── omni-query/
│           │   └── SKILL.md
│           ├── omni-model-builder/
│           │   └── SKILL.md
│           ├── omni-content-explorer/
│           │   └── SKILL.md
│           ├── omni-content-builder/
│           │   └── SKILL.md
│           ├── omni-ai-optimizer/
│           │   └── SKILL.md
│           ├── omni-admin/
│           │   └── SKILL.md
│           └── omni-embed/
│               └── SKILL.md
├── .pre-commit-config.yaml           ← Local lint and validation hooks
├── .rumdl.toml                       ← Markdown lint configuration
├── CLAUDE.md                         ← Contribution standards
└── README.md
```

## Contributing

Contributions are welcome.

Before opening a pull request:

1. Review [CLAUDE.md](CLAUDE.md) for repository-specific contribution standards.
2. Run `bin/validate-skill-frontmatter`.
3. Run `bin/validate-plugin-metadata`.
4. Run `bin/check-hardcoded-paths`.
5. Run `prek run --all-files` or `pre-commit run --all-files`.

Use [.github/PULL_REQUEST_TEMPLATE.md](.github/PULL_REQUEST_TEMPLATE.md) when preparing your PR.

## License

Apache 2.0
