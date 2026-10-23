---
name: commit-msg
description: "Generate a Conventional Commits message based on staged or recent changes. TRIGGER when: user invokes /commit-msg, asks to generate a commit message, wants to know what commit message to write for current changes."
argument-hint: [ticket-number]
allowed-tools: [Bash]
author: NV
---

# Conventional Commit Message Generator

Generate a commit message following https://www.conventionalcommits.org/en/v1.0.0/ based on the current git changes.

## Steps

### 1. Gather context

Run these commands to understand the changes:

```bash
git diff --staged
git diff
git status
git log --oneline -5
```

If `git diff --staged` is empty, use `git diff` (unstaged changes). If both are empty, use `git log -1 --format="%H"` and inspect the last commit with `git show --stat HEAD`.

### 2. Determine the type

Choose the most appropriate type:

| Type | When to use |
|------|-------------|
| `feat` | A new feature or capability |
| `fix` | A bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `style` | Formatting, missing semicolons, etc. — no logic change |
| `test` | Adding or fixing tests |
| `docs` | Documentation only changes |
| `chore` | Build process, dependencies, tooling |
| `perf` | Performance improvement |
| `ci` | CI/CD configuration changes |
| `revert` | Reverts a previous commit |

### 3. Determine the scope (optional)

The scope is the part of the codebase affected, e.g. `auth`, `api`, `cart`, `user`. Use it only when it adds clarity. Derive it from the files changed (folder name, module name, feature name).

### 4. Write the subject line

Rules:
- Imperative mood: "add", "fix", "update" — not "added", "fixing"
- Max 72 characters
- No period at the end
- In English

Format: `<type>(<scope>): <subject>`

Examples:
- `feat(auth): add OAuth2 login with Google`
- `fix(cart): prevent duplicate items on quick add`
- `refactor: extract payment service from order module`

### 5. Breaking change

If the change breaks backward compatibility, add `!` after the type/scope:
`feat(api)!: rename /users endpoint to /accounts`

### 6. Output

Output **only** the final commit message — nothing else, no explanation, no markdown fences.

If `$ARGUMENTS` contains a ticket number (e.g. `PROJ-123`), append it to the message body on a new line:

```
feat(auth): add OAuth2 login with Google

Refs: PROJ-123
```

Otherwise output just the subject line.

The message is printed so the user can copy it, optionally edit it, then run `git commit -m "..."` themselves.
