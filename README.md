# essencium-frontend-migration

A Claude Code plugin that automates migrating downstream projects built on [essencium-frontend](https://github.com/Frachtwerk/essencium-frontend). It analyzes upstream changes between versions, detects customizations in your downstream project, and applies version-by-version migrations interactively -- so you stay up to date without losing your local modifications.

## Installation

```bash
# Add the plugin source
claude plugin marketplace add Frachtwerk/essencium-frontend-migration-plugin

# Install the plugin
claude plugin install essencium-frontend-migration@essencium-frontend-migration
```

## Updating

```bash
claude plugin marketplace update essencium-frontend-migration
claude plugin update essencium-frontend-migration@essencium-frontend-migration
```

Restart Claude Code after updating to apply changes.

## Usage

In your downstream project directory, invoke the migration skill:

```
/migrate-essencium-frontend
```

Optionally specify a target version:

```
/migrate-essencium-frontend 9.5.0
```

The plugin detects your current essencium version, calculates the migration path, and walks you through each version step interactively.

## What it automates

The plugin handles seven categories of upstream changes:

1. **Dependency ecosystem migrations** -- library upgrades (e.g., Zod v3 to v4) that affect your entire project, not just essencium-origin files
2. **Infrastructure/config changes** -- build tools, framework upgrades, PostCSS/ESLint/TypeScript config
3. **File tracking** -- modifications to files you copied from the essencium app boilerplate, merged with your customizations
4. **New files** -- files added upstream that your project may need
5. **File removals** -- files deleted upstream (e.g., CSS modules after Tailwind migration)
6. **Translation key changes** -- added, changed, or removed i18n keys in locale JSON files
7. **Environment variable changes** -- new or changed `.env` variables

For each category the plugin compares the upstream diff against your local files, detects customizations, and applies changes interactively -- preserving your modifications.

## Links

- Upstream framework: https://github.com/Frachtwerk/essencium-frontend
- Plugin repository: https://github.com/Frachtwerk/essencium-frontend-migration-plugin

## License

MIT
