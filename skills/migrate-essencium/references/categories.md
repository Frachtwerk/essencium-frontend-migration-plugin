# Migration Category Handling Reference

This document describes how to handle each of the 7 change categories during the APPLY phase of an essencium-frontend migration. The main migration skill reads this reference to determine the correct procedure for every manifest entry.

## Processing Order

When applying changes for a version step, process categories in this order:

1. **`dependency_migration`** — project-wide scan and transform first
2. **`infrastructure`** — config files
3. **`file_removal`** — remove before adding to avoid conflicts
4. **`new_file`** — add new files
5. **`file_tracking`** — modify existing files (some may already be partially handled by dependency migration)
6. **`translation`** — merge translation keys
7. **`env_variable`** — add missing variables

---

## Category 1 — Dependency Ecosystem Migration (`dependency_migration`)

### When `scope: package`

Just bump the version in the downstream project's `package.json`. No code changes are needed.

### When `scope: project_wide`

1. Bump the dependency version in the downstream project's `package.json`.
2. Fetch the reference URL from the manifest entry. Use WebFetch if available; otherwise rely on training knowledge of that library's migration guide.
3. Scan the ENTIRE downstream project for usage patterns of the old API. Use Grep to find imports and usage of the affected library.
4. For each file with old usage:
   - Read the file and understand the surrounding context.
   - Transform the code to use the new API, following the migration guide.
   - Present each transformed file to the developer for review before applying the change.
5. After all transformations are complete, run the package manager install command.

### Auto/Interactive

- **`scope: package`** — Auto-apply. Just a version bump, no review needed.
- **`scope: project_wide`** — Interactive. The developer reviews every transformed file before it is written.

---

## Category 2 — Infrastructure/Config (`infrastructure`)

### Procedure

1. Read the manifest entry's file list and notes.
2. For each config file:
   - Read the downstream project's version of the file.
   - Fetch the upstream diff via the PR link (use WebFetch on the GitHub PR diff URL).
   - Apply the changes to the downstream file.
3. These files are rarely customized by downstream projects, so apply changes directly with a summary of what changed.
4. If the downstream file contains unexpected customizations (content that differs from the known upstream baseline), flag the file for developer review instead of auto-applying.

### Auto/Interactive

Auto-apply with a summary. Flag for review only if unexpected customizations are detected.

---

## Category 3 — File Tracking (`file_tracking`)

### Procedure

For each file in the manifest entry:

1. Check if the file exists in the downstream project.
2. **If it does not exist:** The downstream project deleted this file. Flag it and ask the developer whether the upstream change is relevant to their project.
3. **If it exists:**
   - Fetch the upstream diff via the PR link.
   - Read the downstream file.
   - Detect whether the downstream project customized this file:
     - Fetch the file from GitHub at the `from` version tag via WebFetch:
       ```
       https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/essencium-app-v<from-version>/packages/app/<path>
       ```
     - Compare the fetched upstream baseline with the downstream file.
     - If they are identical, the file is untouched.
     - If they differ, the downstream project customized it.
   - **If untouched:** Apply the upstream diff directly (auto).
   - **If customized:** Apply interactively. Show the developer what the upstream changed and how it interacts with the downstream customizations. Let them decide how to merge.

### Auto/Interactive

- **Untouched files** — Auto-apply the upstream diff.
- **Customized files** — Interactive. The developer reviews and decides how to merge upstream changes with their customizations.

---

## Category 4 — New Files (`new_file`)

### Procedure

For each file in the manifest entry:

1. Check if the downstream project already has a file at that path.
2. **If not:** Fetch the file content from the essencium-frontend repo at the target version tag:
   ```
   https://raw.githubusercontent.com/Frachtwerk/essencium-frontend/essencium-app-v<target-version>/packages/app/<path>
   ```
   Create it in the downstream project.
3. **If yes:** The downstream project already has something at that path. Flag for review and show the diff between the upstream file and the existing downstream file.
4. Also search for files with similar names or functionality elsewhere in the downstream project. The downstream team may have created their own version in a different location.

### Auto/Interactive

- **No equivalent exists** — Auto-apply. Create the file.
- **File or equivalent already exists** — Flag for review.

---

## Category 5 — File Removals (`file_removal`)

### Procedure

For each file in the manifest entry:

1. Check if the file exists in the downstream project.
2. **If it does not exist:** Skip. It was already removed or never existed.
3. **If it exists:**
   - Search for imports and references to this file across the entire downstream project (use Grep).
   - **If no references found:** Confirm deletion with the developer.
   - **If references exist:** List all referencing files and flag for review. The developer needs to decide how to handle the references before the file can be removed.

### Auto/Interactive

Interactive — always confirm with the developer before deleting any file.

---

## Category 6 — Translation Keys (`translation`)

### Procedure

For each locale in the manifest entry:

1. Read the downstream project's locale JSON file (typically `public/locales/<locale>/common.json`).
2. **`keys_added`:** Add each key if it does not already exist in the downstream file. Fetch the values from the essencium-frontend repo at the target version tag. Preserve all existing custom keys.
3. **`keys_changed`:** For each key, check whether the downstream project still has the same old value (unchanged from upstream). If so, update to the new value. If the downstream project has a custom value (different from the old upstream value), flag for review.
4. **`keys_removed`:** Flag every removal for review. The downstream project may still use these keys.
5. Write the updated JSON file, preserving key ordering and formatting.

### Auto/Interactive

- **Additions** — Auto-apply.
- **Changes** — Interactive if the downstream value was customized; auto if it matches the old upstream value.
- **Removals** — Interactive. Always flag for developer review.

---

## Category 7 — Environment Variables (`env_variable`)

### Procedure

For each variable in the manifest entry:

1. Check all `.env*` files in the downstream project root.
2. **If the variable already exists** in any `.env*` file: Skip it.
3. **If missing:**
   - Add it to the primary `.env` file (or `.env.local` if that matches the project's pattern).
   - If the manifest specifies a default value, use that value.
   - If no default is specified, add the variable with a `# TODO: set value` comment.

### Auto/Interactive

Auto-apply with a summary of which variables were added and to which files.

---

## Edge Cases

### Downstream deleted an upstream file

The manifest says to modify a file (`file_tracking`) but it does not exist in the downstream project. Flag this and ask the developer whether the upstream change is relevant to their project. Do not silently skip it.

### Downstream already applied a fix

If the change described in the manifest entry is already present in the downstream file (the code already matches the target state), skip it and note that it was already applied. Do not apply it again or flag it as a conflict.

### Dependency migration + file tracking overlap

When a dependency migration (e.g., Zod v4) affects both custom downstream code and essencium-origin files, the dependency migration scan in step 1 may already transform some files that also appear in `file_tracking` entries in step 5. During `file_tracking` processing, check whether the relevant change was already applied by the dependency migration step before attempting to apply it again.

### File renames

Some `file_tracking` entries may contain both an "added" and a "removed" file path — this represents a rename. Handle by:
- If the old file exists in the downstream project: move/rename it to the new path.
- If the old file does not exist: create the new file as if it were a `new_file` entry.
- After renaming, apply any content changes from the upstream diff to the renamed file.
