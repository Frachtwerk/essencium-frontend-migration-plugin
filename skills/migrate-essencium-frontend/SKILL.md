---
name: migrate-essencium-frontend
description: "Migrate an essencium-frontend based project to a newer version. Detects current version, analyzes customizations, and applies upstream changes interactively. Use when upgrading @frachtwerk/essencium-lib or @frachtwerk/essencium-types."
argument-hint: "[target-version]"
---

# Essencium Frontend Migration Skill

You are performing an interactive, version-by-version migration of a downstream project that is based on essencium-frontend. This is a collaborative process -- be patient, explain what you are doing, and always wait for developer confirmation before making destructive or ambiguous changes.

## Important references

Before you begin, be aware of two reference documents bundled with this skill:

- **`./references/categories.md`** -- Detailed handling instructions for each of the 7 change categories. You MUST read this file during the APPLY phase (step 2c).
- **`./references/manifest-format.md`** -- The YAML schema for migration manifests. Read this file if you need to understand the structure of a manifest entry.

The upstream essencium-frontend repository is at: https://github.com/Frachtwerk/essencium-frontend
Git tags follow the pattern: `essencium-app-v<version>` (e.g., `essencium-app-v9.4.5`).

Migration manifests are fetched directly from the essencium-frontend GitHub repository at runtime. They are **not** bundled with this plugin — this means new manifests are available immediately when essencium-frontend publishes a release, without requiring a plugin update.

- **List available manifests:** `https://api.github.com/repos/Frachtwerk/essencium-frontend/contents/packages/app/manifests`
- **Fetch a specific manifest:** `https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/main/packages/app/manifests/<version>.yaml`

## GitHub API usage

This skill fetches manifests and upstream files from GitHub. Use the correct method for each type of request:

- **Raw file content** (manifest YAMLs, source files at a specific tag): Always use `raw.githubusercontent.com` URLs via WebFetch. Do NOT use `gh api repos/.../contents/...` for this — that endpoint returns base64-encoded JSON, not raw content, and requires extra decoding.
  - Manifest: `https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/main/packages/app/manifests/<version>.yaml`
  - Source file at tag: `https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/essencium-app-v<version>/packages/app/<path>`
- **Directory listings** (to find available manifests): Use the GitHub Contents API with an explicit Accept header:
  ```
  GET https://api.github.com/repos/Frachtwerk/essencium-frontend/contents/packages/app/manifests
  Accept: application/vnd.github.v3+json
  ```

Unauthenticated requests are limited to 60/hour. For multi-version migrations, if the user has a `GITHUB_TOKEN` environment variable set, use it in WebFetch headers (`Authorization: token <value>`) to increase the rate limit to 5000/hour. If rate limiting is encountered, suggest the developer set a `GITHUB_TOKEN`.

---

## Step 0: PRE-FLIGHT

Before starting the migration, establish baselines and verify the environment is ready.

1. **Clean git state.** Run `git status`. If there are uncommitted changes, ask the developer to commit or stash them before proceeding. The migration must start from a clean working tree so that git history accurately reflects migration work.

2. **Baseline type check.** Run the project's type-check command (typically `npx tsc --noEmit` or `pnpm typecheck`). Record whether it passes or fails, and how many errors exist. This baseline is critical: during verification, only errors that are NEW compared to this baseline are attributed to the migration. Pre-existing errors are ignored.

3. **Baseline test run.** Run the project's test command. Record pass/fail counts. Same rationale: pre-existing failures are not migration failures.

4. **Manifest pre-checks.** After loading each manifest (step 2a), if it contains a `pre_checks` array, execute each check before applying any changes for that version. If any check with `severity: blocker` fails, stop and help the developer resolve it before proceeding.

---

## Step 1: DETECT

Determine the current state of the downstream project and calculate the migration path.

1. **Read the downstream project's `package.json`** to find the currently installed versions of `@frachtwerk/essencium-lib` and `@frachtwerk/essencium-types`. These tell you the current essencium version the project is based on.

2. **Determine the target version.** If the developer passed a target version as `$ARGUMENTS`, use that. Otherwise, default to the latest (highest) version available from the manifest listing.

3. **List all available manifest files** by fetching the GitHub API:
   ```
   https://api.github.com/repos/Frachtwerk/essencium-frontend/contents/packages/app/manifests
   ```
   Parse the version number from each filename in the response (e.g., `9.4.5.yaml` means version `9.4.5`). Sort them in ascending semver order.

4. **Calculate the migration path.** Starting from the current version, collect every manifest version that is greater than the current version and less than or equal to the target version. These are the steps to execute, in ascending order. Each manifest's `from` field tells you which version it migrates from -- use this to verify the chain is valid.

   **Not all versions have manifests.** Manifests are only created for releases that contain app-level changes relevant to downstream projects. Patch releases that only touch library internals or docs will not have a manifest. This is by design — the migration path only includes versions with actual migration steps.

   - Only include versions that have a corresponding manifest YAML file.
   - If the downstream project's current version has no manifest (e.g., downstream is at 9.4.2 but the nearest manifests are 9.4.0 and 9.4.4), find the **next available manifest** whose version is greater than the current version. Apply it as the first step. The `from` field in that manifest indicates the intended baseline, not a hard requirement — patch versions between manifests typically don't introduce breaking changes that would invalidate the manifest.
   - If the downstream project's current version is **older** than the oldest available manifest, warn the developer that migration manifests only cover versions from 6.0.0 onward.

5. **Present the migration path** to the developer and ask for confirmation before proceeding:

   ```
   Current: @frachtwerk/essencium-lib@9.4.2
   Target: 9.5.0
   Migration path: 9.4.4 -> 9.4.5 -> 9.5.0
   (Only versions with migration manifests are shown. Your version 9.4.2 has no manifest — starting from the next available: 9.4.4)

   Proceed? (y/n)
   ```

6. If the developer declines, stop. If they confirm, proceed to step 2.

---

## Step 2: Execute migration steps (version by version)

For each version in the migration path, execute these sub-steps in order. Do NOT skip ahead to the next version until the current one passes verification.

### Step 2a: LOAD manifest

1. Fetch the manifest YAML file for this version step from GitHub:
   ```
   https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/main/packages/app/manifests/<version>.yaml
   ```
2. Parse the manifest and group the changes by their `type` field. The 7 types are:
   - `dependency_migration`
   - `infrastructure`
   - `file_removal`
   - `new_file`
   - `file_tracking`
   - `translation`
   - `env_variable`
3. Read the `./references/categories.md` file to understand the handling instructions for each category.

### Step 2b: PLAN

For each change entry in the manifest, assess its impact on the downstream project:

1. **For `dependency_migration` entries:**
   - Check if the dependency exists in the downstream `package.json`.
   - If `scope: project_wide`: use Grep to scan the entire downstream project for imports and usage of the affected package. Note how many files use it and whether the usage matches patterns that would need updating.
   - If `scope: package`: note that this is a simple version bump.

2. **For `file_tracking` entries:**
   - Check if the file exists in the downstream project.
   - If the file exists, detect whether the downstream project customized it:
     - Fetch the file from the essencium-frontend GitHub repo at the `from` version tag using WebFetch:
       ```
       https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/essencium-app-v<from-version>/packages/app/<path>
       ```
     - Compare the fetched upstream baseline with the downstream file content.
     - If they are identical, the file is **untouched** (can be auto-applied).
     - If they differ, the file is **customized** (requires interactive merge).
   - If the file does not exist, note that the downstream project deleted it.

3. **For `new_file` entries:**
   - Check if the downstream project already has a file at that path.
   - Search for files with similar names or functionality elsewhere in the downstream project.

4. **For `file_removal` entries:**
   - Check if the file exists in the downstream project.
   - If it exists, use Grep to search for imports and references to this file across the project.

5. **For `translation` entries:**
   - Check if the locale files exist in the downstream project.
   - Note which keys are added, changed, or removed.

6. **For `env_variable` entries:**
   - Check all `.env*` files in the downstream project root for existing variables.

7. **For `infrastructure` entries:**
   - Check if the config files exist in the downstream project.
   - Check for unexpected customizations.

**Present a summary** of the plan to the developer:

```
Version 9.4.5 migration plan:
- [dependency] axios 1.8.2 -> 1.12.0: package bump only
- [dependency] vite 6.2.7 -> 6.3.6: package bump only
- [file_tracking] src/app/.../RightsView.tsx: customized (2 extra functions) -- INTERACTIVE
- [translation] 1 key to add to de/en locale files -- AUTO
```

Ask the developer to confirm the plan before applying changes.

### Step 2c: APPLY

**Before applying, read `./references/categories.md` for the detailed handling procedures.** Follow those instructions precisely.

Process changes in this exact order (dependencies first, then structural, then content):

#### 0. Bump essencium packages

Before processing other changes, update `@frachtwerk/essencium-lib` and `@frachtwerk/essencium-types` in the downstream project's `package.json` to match this version step's target version. Run the package manager install command to fetch the updated packages. This ensures the downstream project has the correct lib/types for this version before applying other changes.

#### 1. `dependency_migration`

- **`scope: package`:** Auto-apply. Update the version in the downstream `package.json`.
- **`scope: project_wide`:** Interactive.
  1. Bump the version in `package.json`.
  2. Fetch the reference URL from the manifest entry using WebFetch (if available). Otherwise, rely on your knowledge of that library's migration guide.
  3. Use Grep to scan the entire downstream project for usage patterns of the old API.
  4. **Process ALL matches in a single pass.** Do NOT split work across parallel sub-agents — sequential processing of a single grep result set ensures no files are overlooked. If there are more than 50 files to transform, generate a codemod script instead of editing files individually (see "Batch operations" below).
  5. For each file with old usage, read the file, transform the code to the new API, and present the change to the developer for review before applying.
  6. After all transformations, run the package manager install command.
  7. **Test infrastructure.** After transforming application code, scan the test infrastructure for references to the old dependency:
     - Check test setup files (`setupTests.*`, `vitest.setup.*`, `jest.setup.*`, or equivalent) for global mocks or configuration of the migrated library.
     - Scan test files (`**/*.test.*`, `**/*.spec.*`) for local mocks of the migrated package (e.g., `vi.mock('<old-package>')`, `jest.mock('<old-package>')`).
     - Transform these files using the same migration guide.
     - If the manifest entry includes a `test_infrastructure` field, use its `files` and `patterns` as the starting point for this scan.
     - **Test infrastructure is NOT optional.** Stale test mocks after a dependency migration cause cascading test failures that are difficult to diagnose.
  8. **Verification checkpoint.** Run the project's type-check command. Compare against the Step 0 baseline — only new errors are relevant. If new type errors appear, they indicate missed files; fix them before proceeding to the next change category. This checkpoint is mandatory for all `scope: project_wide` migrations.

#### 2. `infrastructure`

- Read the downstream config files.
- Fetch the upstream diff via the PR link using WebFetch if a `pr` field is provided.
- Apply changes directly with a summary. If unexpected customizations are detected, flag for developer review instead.

#### 3. `file_removal`

- For each file to be removed:
  - If the file does not exist: skip.
  - If it exists and has no references (confirmed via Grep): confirm deletion with the developer.
  - If it exists and has references: list all referencing files and flag for review. Help the developer resolve references before removing.
- Always confirm with the developer before deleting any file.

#### 4. `new_file`

- For each new file:
  - If no file exists at that path: fetch the file content from the essencium-frontend repo at the target version tag:
    ```
    https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/essencium-app-v<target-version>/packages/app/<path>
    ```
    Create it in the downstream project. Auto-apply.
  - If a file or equivalent already exists: flag for review and show a diff.

#### 5. `file_tracking`

- For each tracked file change:
  - **If the file does not exist in the downstream project:** Flag it. Ask the developer whether the upstream change is relevant. Do not silently skip.
  - **If the file exists and is untouched** (matches the upstream baseline at the `from` version): auto-apply the upstream diff.
  - **If the file exists and is customized** (differs from upstream baseline): interactive merge. Show the developer what the upstream changed and how it interacts with their customizations. Let them decide how to merge.
  - **If the change was already applied** (e.g., by a dependency migration in step 1): skip and note it.
  - **For file renames** (entries with both an `added` and a `removed` path):
    - If the old file exists: rename it to the new path, then apply any content changes.
    - If the old file does not exist: create the new file as if it were a `new_file` entry.

#### 6. `translation`

- For each locale:
  - Read the downstream locale JSON file (typically `public/locales/<locale>/common.json`).
  - **`keys_added`:** Add each key if it does not already exist. Fetch values from the essencium-frontend repo at the target version tag. Preserve all existing custom keys. Auto-apply.
  - **`keys_changed`:** Check if the downstream value matches the old upstream value. If so, update automatically. If the downstream has a custom value, flag for review.
  - **`keys_removed`:** Always flag for developer review. The downstream project may still use these keys.
  - Write the updated JSON, preserving key ordering and formatting.

#### 7. `env_variable`

- For each variable:
  - Check all `.env*` files in the downstream project root.
  - If the variable already exists in any `.env*` file: skip.
  - If missing: add it to the primary `.env` file (or `.env.local` if that matches the project's pattern). Use the default value if specified in the manifest, otherwise add with a `# TODO: set value` comment.
  - Auto-apply with a summary.

### Step 2d: VERIFY

After applying all changes for this version step:

1. **Run the build command.** Detect from the downstream `package.json` scripts -- typically `npm run build` or `pnpm build`. If using a monorepo, identify the correct workspace command.

2. **Run the linter** if available (`npm run lint` or `pnpm lint`).

3. **Run tests** if available (`npm test` or `pnpm test`).

4. **If any command fails:**
   - Stop and help the developer diagnose and fix the issue.
   - Re-run verification after fixes are applied.
   - Do NOT proceed to the next version step until all checks pass.

5. **If all pass:**
   - Ask the developer for confirmation before committing.
   - Commit the migration for this version step:
     - Run `git status` to review all changes
     - Stage only the files that were modified during this migration step. Do NOT use `git add -A` as it may stage sensitive files (.env.local, credentials) or unrelated work-in-progress
     - Confirm the staged files with the developer before committing
   - Only commit after the developer confirms.

### Step 2e: Repeat

Move to the next version in the migration path and go back to step 2a. Continue until all versions have been migrated.

---

## Step 3: COMPLETE

After all version steps have been successfully migrated:

1. **Present a summary** of everything that was done:
   - List each version step that was applied.
   - Summarize key changes per version (dependencies bumped, files modified, new files added, files removed, etc.).
   - Note any interactive decisions the developer made.

2. **Remind the developer** to take the following final actions:
   - Run a final build and test to confirm everything works end-to-end.
   - Review any `# TODO` comments that were added during the migration.

---

## Edge cases and important notes

### PR link handling
When a manifest entry includes a `pr` field (a GitHub PR URL), use WebFetch to fetch the PR diff page to understand what changed upstream. If WebFetch is unavailable or fails, fall back to the manifest entry's `description` and `notes` fields to understand the change.

### Downstream deleted an upstream file
If a `file_tracking` entry references a file that does not exist in the downstream project, do NOT silently skip it. Flag it and ask the developer whether the upstream change is relevant to their project.

### Downstream already applied a fix
If the change described in a manifest entry is already present in the downstream file (the code already matches the target state), skip it and note that it was already applied. Do not re-apply or flag as a conflict.

### Dependency migration + file tracking overlap
When a dependency migration (step 2c.1) affects files that also appear in `file_tracking` entries (step 2c.5), the dependency migration scan may have already transformed those files. During `file_tracking` processing, check whether the relevant change was already applied before attempting to apply it again.

### File renames
Some `file_tracking` entries contain both an `added` and a `removed` file path. This represents a rename. Handle by moving/renaming the old file to the new path, then applying any content changes from the upstream diff.

### Batch operations (codemod-style)

When a dependency migration or file tracking change requires the same mechanical transformation across many files (more than 20), do NOT edit files one by one. Instead:

1. Identify the exact search pattern and its replacement.
2. Write a codemod script (Node.js or shell) that performs the transformation.
3. Present the script to the developer for review before running it.
4. Execute it once across all matching files.
5. Run the type-check command to verify.
6. Show a summary of files changed.

If the manifest entry includes a `batch_hint` field, use its `pattern`, `replacement`, and `file_glob` to generate the codemod script.

### Prohibited operations during migration

- **No `git stash` or `git reset --hard`.** These risk losing all migration changes accumulated so far. To determine whether a failure is pre-existing, consult the baseline recorded in Step 0 rather than stashing or resetting.
- **No `sleep` commands.** Run commands synchronously. If a command is asynchronous by nature, use a proper wait mechanism, not a timed sleep.
- **No `git checkout -- <file>` or `git restore` on migrated files.** If a change needs to be undone, edit the file back explicitly so the developer can see exactly what was reversed.

### Code generation guidelines

When generating or modifying code during the migration:

- **Respect the module system.** Before writing imports or exports, check what module system the target file uses — look at the file extension (`.mjs`, `.cjs`, `.ts`), the `"type"` field in the nearest `package.json`, and whether the project uses a bundler. Do not use `require()` in ESM files or `import` in CommonJS files.
- **Preserve downstream conventions.** Do not blindly copy upstream code into the downstream project. Check whether the downstream has local type overrides, its own utility functions, or different import paths for the same concepts, and preserve them.
- **Check the Node.js version.** Before using newer Node.js APIs (e.g., `import.meta.dirname`, available since Node 21.2.0), verify the Node.js version the downstream project targets. Use older equivalents (e.g., `path.dirname(fileURLToPath(import.meta.url))`) when compatibility requires it.

### This is a collaborative process
This migration skill is designed to be interactive and patient. Always explain what you are about to do. Always wait for developer confirmation before making ambiguous or destructive changes. Present clear summaries and diffs. If the developer wants to skip a change or handle it differently, respect their decision and note it for the summary.
