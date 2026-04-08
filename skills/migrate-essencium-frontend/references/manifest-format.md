# Manifest Format Reference

This document defines the YAML schema used by migration manifests. Each manifest describes the changes introduced in a single release version.

## Top-level fields

All top-level fields are required.

| Field | Type | Description |
|-------|------|-------------|
| `version` | string | The version being released, e.g., `"9.5.0"` |
| `from` | string | The previous version this manifest migrates from, e.g., `"9.4.5"` |
| `date` | string | Release date in `YYYY-MM-DD` format |
| `changes` | array | List of change entries (see below) |

## Change entry types

Each entry in the `changes` array must have a `type` field that determines its schema. The following types are supported:

### `dependency_migration`

Library version bumps. Used when an npm dependency is updated.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"dependency_migration"` |
| `package` | string | yes | npm package name, e.g., `"axios"` |
| `from` | string | yes | Old version |
| `to` | string | yes | New version |
| `scope` | string | yes | Either `project_wide` (downstream code may need changes) or `package` (just bump the version, no code changes needed) |
| `reference` | string | yes | URL to changelog or migration guide |
| `notes` | string | yes | Description of impact |

### `infrastructure`

Config, build tool, or project infrastructure changes.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"infrastructure"` |
| `description` | string | yes | What changed |
| `files` | array | yes | List of `{path, action}` objects. `path` is relative to the app root. `action` is `modified` or `added`. |
| `pr` | string | no | GitHub PR URL |
| `notes` | string | no | Additional context |

### `file_tracking`

Modified existing files. Used when source files are changed in ways that downstream projects should be aware of.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"file_tracking"` |
| `description` | string | yes | What changed and why |
| `files` | array | yes | List of `{path, action}` objects. `path` is relative to the app root. `action` is `modified` or `added`. |
| `pr` | string | no | GitHub PR URL |

### `new_file`

Newly added files that downstream projects should incorporate.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"new_file"` |
| `path` | string | yes | File path relative to the app root |
| `description` | string | yes | Purpose of the file |
| `pr` | string | no | GitHub PR URL |

### `file_removal`

Deleted files that downstream projects should also remove.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"file_removal"` |
| `path` | string | yes | File path relative to the app root |
| `description` | string | yes | Why it was removed and what replaces it |
| `pr` | string | no | GitHub PR URL |

### `translation`

Locale/i18n key changes.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"translation"` |
| `description` | string | yes | What translations changed |
| `locales` | array | yes | List of locale codes, e.g., `["de", "en"]` |
| `keys_added` | array | yes | Dotted key paths that were added |
| `keys_removed` | array | yes | Dotted key paths that were removed |
| `keys_changed` | array | yes | Dotted key paths whose values changed |

### `env_variable`

Environment variable changes.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | `"env_variable"` |
| `description` | string | yes | What variables changed |
| `variables` | array | yes | List of `{name, required}` objects. `name` is the variable name, `required` is a boolean. |

## Complete example

```yaml
version: "9.4.5"
from: "9.4.4"
date: "2025-09-22"

changes:
  - type: dependency_migration
    package: "axios"
    from: "1.8.2"
    to: "1.12.0"
    scope: package
    reference: "https://github.com/axios/axios/releases"
    notes: "Bump axios to resolve CVE. No API changes required."

  - type: file_tracking
    description: "Improve accessibility of rights view checkboxes"
    files:
      - path: "src/app/[locale]/(main)/admin/rights/RightsView.tsx"
        action: modified
    pr: "https://github.com/Frachtwerk/essencium-frontend/pull/867"

  - type: translation
    description: "Add aria-label translation key for rights view checkboxes"
    locales: ["de", "en"]
    keys_added:
      - "rightsView.table.checkbox"
    keys_removed: []
    keys_changed: []
```

## Notes

- All file paths are relative to the app package root (e.g., `src/...`, not `packages/app/src/...`).
- The `from` top-level field indicates which version this manifest migrates FROM.
- Manifests are stored in the `manifests/` directory, named by their version number (e.g., `9.4.5.yaml`).
