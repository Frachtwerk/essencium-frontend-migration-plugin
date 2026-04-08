# essencium-frontend-migration

A Claude Code plugin that automates migrating downstream projects built on [essencium-frontend](https://github.com/Frachtwerk/essencium-frontend). It analyzes upstream changes between versions, detects customizations in your downstream project, and applies version-by-version migrations interactively -- so you stay up to date without losing your local modifications.

## Installation

```
claude install essencium-frontend-migration
```

## Usage

In your downstream project directory, invoke the migration skill:

```
/migrate-essencium
```

The plugin will walk you through each pending version upgrade, presenting changes and letting you accept, adapt, or skip each one.

## What it automates

The plugin handles seven categories of upstream changes:

1. **Dependency updates** -- new, removed, or changed packages in `package.json`
2. **Configuration changes** -- updates to bundler, linter, TypeScript, and other config files
3. **API client changes** -- modifications to generated or hand-written API client code
4. **Component changes** -- updated, added, or removed React components
5. **Type/schema changes** -- Zod schemas, TypeScript types, and shared type definitions
6. **Routing changes** -- new or restructured routes and navigation
7. **Internationalization changes** -- added or modified translation keys and locale files

For each category the plugin compares the upstream diff against your local files, identifies conflicts, and guides you through resolution.

## Links

- Upstream framework: https://github.com/Frachtwerk/essencium-frontend
- Plugin repository: https://github.com/Frachtwerk/essencium-frontend-migration-plugin

## License

MIT
