# Manifest Format Reference

This document defines the YAML schema used by migration manifests. Each manifest describes the changes introduced in a single release version.

## Top-level fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | yes | The version being released, e.g., `"9.5.0"` |
| `from` | string | yes | The previous version this manifest migrates from, e.g., `"9.4.5"` |
| `date` | string | yes | Release date in `YYYY-MM-DD` format |
| `changes` | array | yes | List of change entries (see below) |
| `pre_checks` | array | no | Validations to run before applying any changes for this version. See "Pre-checks" section below. |

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
| `batch_hint` | object | no | Hints for efficient batch transformation. When present, the migration agent should generate a codemod script rather than editing files individually. Contains `pattern` (regex matching the old code), `replacement` (new code), and `file_glob` (glob pattern for files to scan, e.g. `"src/**/*.tsx"`). |
| `test_infrastructure` | object | no | Test setup files and patterns known to be affected by this dependency change. Contains `files` (array of paths to test setup files) and `patterns` (array of grep patterns to find affected test mocks, e.g. `"vi.mock('old-package')"`). Used as a starting point for the test infrastructure scan. |

**Special values:**

- `from` can be `null` when a dependency is newly added (not previously present in the downstream project).
- `to` can be `"removed"` when a dependency is being uninstalled.
- Both `reference` and `notes` are still required in these cases.

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
| `files` | array | yes | List of `{path, action}` objects. `path` is relative to the app root. `action` is `modified`, `added`, or `removed`. |
| `pr` | string | no | GitHub PR URL |

**File renames:** When a `file_tracking` entry contains files with both `action: added` and `action: removed`, this represents a file rename. The migration skill handles renames by moving the old file to the new path.

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
| `variables` | array | yes | List of `{name, required, default?}` objects. `name` is the variable name, `required` is a boolean, `default` is an optional string with the initial value. |

## Pre-checks

The optional `pre_checks` top-level array lists validations that must pass before any changes for this version are applied. Each entry has the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | yes | What is being checked and why it matters |
| `command` | string | yes | Shell command to execute. Exit code `0` = pass, non-zero = fail. |
| `severity` | string | yes | `blocker` (migration stops until resolved) or `warning` (reported but does not block) |

Pre-checks are designed to catch downstream-specific incompatibilities that the migration agent cannot anticipate without running the check. Common use cases:

- Validating that existing data files (e.g., locale JSONs, config files) are compatible with a new library before applying changes
- Verifying that the environment meets minimum version requirements
- Confirming that a prerequisite manual step has already been completed

**Example:**
```yaml
pre_checks:
  - description: "Verify the project's locale files use key formats compatible with the new i18n library"
    command: "node -e \"const f=require('fs');const check=(d)=>{for(const e of f.readdirSync(d,{withFileTypes:true})){if(e.isDirectory())check(d+'/'+e.name);else if(e.name.endsWith('.json')){const k=Object.keys(JSON.parse(f.readFileSync(d+'/'+e.name)));const bad=k.filter(x=>/[^\\w\\-]/.test(x.replace(/\\./,'_')));if(bad.length){console.error('Incompatible keys in '+d+'/'+e.name+':',bad);process.exit(1)}}}};check('public/locales')\""
    severity: blocker
  - description: "Check that Node.js version meets the minimum requirement for this release"
    command: "node -e \"const [major]=process.version.slice(1).split('.').map(Number);if(major<20){console.error('Node '+process.version+' detected, >= 20 required');process.exit(1)}\""
    severity: warning
```

---

## Complete example

```yaml
version: "9.4.5"
from: "9.4.4"
date: "2025-09-22"

pre_checks:
  - description: "Verify the project is on a supported Node.js version"
    command: "node -e \"const [major]=process.version.slice(1).split('.').map(Number);if(major<18){console.error('Node '+process.version+' detected, >= 18 required');process.exit(1)}\""
    severity: warning

changes:
  - type: dependency_migration
    package: "axios"
    from: "1.8.2"
    to: "1.12.0"
    scope: package
    reference: "https://github.com/axios/axios/releases"
    notes: "Bump axios to resolve CVE. No API changes required."

  - type: dependency_migration
    package: "some-query-lib"
    from: "3.x"
    to: "4.x"
    scope: project_wide
    reference: "https://github.com/example/some-query-lib/releases/v4"
    notes: "useQuery signature changed: options object is now the second argument."
    batch_hint:
      pattern: "useQuery\\(([^,]+),\\s*([^,]+)\\)"
      replacement: "useQuery($1, { queryFn: $2 })"
      file_glob: "src/**/*.{ts,tsx}"
    test_infrastructure:
      files:
        - "src/setupTests.ts"
      patterns:
        - "vi.mock('some-query-lib')"
        - "jest.mock('some-query-lib')"

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
