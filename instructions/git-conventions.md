# Git Conventions

## Safety Rules

- `git push` 前必须告知用户并等待确认，不可直接执行
- `gh pr create` 前必须告知用户并等待确认，不可直接执行
- `git push --force` / `git push -f`：**默认拒绝**，除非用户明确说"我知道风险，强制推送"
- **Merge conflicts**: when a conflict is detected, analyze the resolution first:
  - If the resolution **combines both sides** (additive merge, no functionality lost) → resolve and proceed, briefly note what was merged
  - If the resolution **discards or overwrites functionality from either branch** → present the conflict and proposed resolution, then **wait for user confirmation before proceeding**. Never silently drop changes from either side.

## Commit Convention

Follow [Conventional Commits v1.0.0-beta.4](https://www.conventionalcommits.org/en/v1.0.0-beta.4/).

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer]
```

### Types

| Type | When to use |
|------|------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `chore` | Build process, dependency updates, tooling |

### Rules

- Description is lowercase, imperative mood, no period at end
- Scope is optional: `feat(parser): add ability to parse arrays`
- Breaking changes: append `!` after type — `feat!: remove deprecated API`
- Body and footer are separated from description by a blank line

### Examples

```
feat: add context engineering with /compact command

fix(tools): catch TimeoutExpired in execute_command

refactor: replace file tools with bash guidance

feat!: drop support for Python 3.10

chore: update dependencies
```
