# Changelog Commit Command

You are tasked with creating a git commit with an automatically generated, detailed changelog.

## Parse Arguments

First, determine the mode based on user input:
- If the user provided `--auto`, set mode to AUTO
- If the user provided `--review`, set mode to REVIEW
- Otherwise, set mode to SHOW_ONLY (default)

## Step 1: Analyze Changes

Execute the following git commands IN PARALLEL:
1. `git status` - to see all modified and untracked files
2. `git diff` - to see unstaged changes
3. `git diff --staged` - to see staged changes
4. `git log -5 --oneline` - to understand recent commit message style

## Step 2: Analyze Commit Type

Based on the changes detected, determine the appropriate commit type:

- **feat:** New features or capabilities added to the codebase
- **fix:** Bug fixes or error corrections
- **docs:** Documentation-only changes (README, markdown files, code comments)
- **refactor:** Code restructuring without changing functionality
- **chore:** Maintenance tasks (config files, dependencies, build tools)
- **test:** Adding or modifying tests
- **style:** Code formatting, whitespace, missing semicolons (no logic change)
- **perf:** Performance improvements

**Analysis Guidelines:**
- If multiple types apply, choose the most significant one
- New files typically indicate `feat` or `docs`
- Changes to `.md` files alone = `docs`
- Changes to config files (`.json`, `.yaml`, `.env.example`) = `chore`
- Changes to test files = `test`
- Mixed changes should use the type covering the majority of changes

## Step 3: Generate Changelog

Create a structured changelog that includes:

1. **Short description** (50 chars max): One-line summary of changes
2. **Changes section**: Bullet points describing what was modified
   - Group related changes together
   - Be specific but concise
   - Mention file paths when relevant
3. **Impact section**: Describe the effect of these changes
   - How does this improve the codebase?
   - What problems does it solve?
   - What new capabilities does it enable?

**Format:**
```
<type>: <short description>

## Changes
- Change description 1
- Change description 2
- Change description 3

## Impact
- Impact description
```

**Important:**
- DO NOT include emojis
- DO NOT include "Generated with Claude Code" footer
- DO NOT include "Co-Authored-By" line
- Keep it professional and concise
- Use Italian language if the project documentation is in Italian

## Step 4: Execute Based on Mode

### SHOW_ONLY Mode (default)
1. Display the suggested commit type
2. Display the complete changelog message
3. Inform the user: "This is the proposed changelog. No commit has been created yet."
4. Suggest: "Run `/changelog-commit --review` to commit with confirmation, or `/changelog-commit --auto` to commit immediately"
5. STOP - do not commit

### REVIEW Mode
1. Display the suggested commit type
2. Display the complete changelog message
3. Ask the user: "Do you want to proceed with this commit? (yes/no)"
4. If yes: execute `git add .` and `git commit` with the generated message
5. If no: STOP and inform "Commit cancelled"
6. After successful commit: show `git log -1 --stat` to confirm

### AUTO Mode
1. Display: "Analyzing changes and committing automatically..."
2. Execute `git add .`
3. Execute `git commit` with the generated changelog message
4. Show `git log -1 --stat` to confirm the commit
5. Display: "Commit created successfully. Run `git push` to publish."

## Important Notes

- Always use `git commit -m "$(cat <<'EOF' ... EOF)"` pattern for multi-line messages
- Ensure proper escaping of special characters in commit messages
- If there are no changes to commit, inform the user: "No changes detected. Working tree is clean."
- If `git add .` would include files that look like secrets (.env, credentials.json, etc.), warn the user before committing
- Never commit without analyzing what's being committed first

## Error Handling

- If git commands fail, display the error and explain what went wrong
- If pre-commit hooks fail, show the output and ask user how to proceed
- If commit message generation fails, ask user to provide a manual commit message

---

Now execute this workflow based on the user's input and the detected mode.
