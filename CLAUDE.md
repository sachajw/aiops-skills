# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

The "skills" CLI is the package manager for the open agent skills ecosystem. It installs, manages, and updates SKILL.md files (reusable agent instruction sets) across 34+ AI coding agents including Claude Code, Cursor, GitHub Copilot, and others.

## Common Commands

| Command | Description |
|---------|-------------|
| `npm run build` | Build the project (uses obuild bundler) |
| `npm test` | Run tests (vitest) |
| `npm run type-check` | TypeScript type checking |
| `npm run format` | Format code with prettier |
| `npm run format:check` | Check code formatting |

## High-Level Architecture

### Core Entry Points

- **`src/cli.ts`**: Main CLI entry point with command routing, banner display, and telemetry initialization
  - Routes to subcommands: `add`, `list`, `find`, `remove`, `check`, `update`, `init`, `generate-lock`
  - Also aliased as `a`/`i`/`install` for add, and `ls` for list

- **`bin/cli.mjs`**: Thin wrapper that enables Node.js compile cache and imports the built dist

### Agent System

**`src/agents.ts`**: Central registry of 34+ supported AI coding agents

Each agent has:
- `skillsDir`: Project-local path (e.g., `.claude/skills`)
- `globalSkillsDir`: User-home path (e.g., `~/.claude/skills`)
- `detectInstalled()`: Async function checking if agent is present

Key pattern: When modifying agent support, also run `scripts/sync-agents.ts` to update README.md and package.json keywords.

### Installation System

**`src/installer.ts`**: Core installation logic

Two installation modes:
- **Symlink (default)**: Single canonical copy in `.agents/skills/`, symlinks to each agent
- **Copy**: Independent copies per agent (fallback when symlinks unsupported)

Security:
- `sanitizeName()`: Prevents path traversal, converts to kebab-case
- `isPathSafe()`: Validates paths stay within expected base directories

Installation flow:
1. Clone git repo or use local path
2. Discover all SKILL.md files (recursive search, max depth 5)
3. Parse frontmatter (name, description, metadata.internal)
4. Install to canonical location (`.agents/skills/`)
5. Symlink/copy to each selected agent's skills directory

### Skill Discovery

**`src/skills.ts`**: Finds and parses SKILL.md files

- Searches standard directories: `skills/`, `.agents/skills/`, agent-specific dirs
- Parses YAML frontmatter with `gray-matter`
- Skips `internal: true` skills unless `INSTALL_INTERNAL_SKILLS=1`
- Recursive directory traversal (max 5 deep, skips `node_modules`, `.git`, `dist`)

### Source Parsing

**`src/source-parser.ts`**: Parses various skill source formats

Supported inputs:
- GitHub shorthand: `owner/repo`
- Full GitHub URLs: `https://github.com/owner/repo`
- GitLab URLs, any git URL
- Direct skill URLs: Mintlify docs, HuggingFace Spaces
- Local paths: `./local-skills`

Extracts:
- Owner/repo for telemetry
- Private repo detection (GitHub API)
- Skill filter from `@skill-name` syntax

### Git Operations

**`src/git.ts`**: Git cloning with timeout and error handling

- Uses `simple-git` with 60s timeout
- Shallow clones (`--depth 1`) for performance
- Custom error types: `GitCloneError` with timeout/auth detection
- Automatic temp dir cleanup on failure

### Provider System

**`src/providers/`**: Extensible registry for remote skill hosts

Built-in providers:
- **Mintlify**: Fetches skills from Mintlify documentation sites
- **HuggingFace**: Fetches from HuggingFace Spaces
- **WellKnown**: Fetches from sites with `.well-known/skills` index

Provider interface:
- `match(url)`: Detects if URL belongs to provider
- `fetchSkill(url)`: Fetches and parses SKILL.md
- `toRawUrl(url)`: Converts to raw content URL
- `getSourceIdentifier(url)`: Returns telemetry identifier

### Lock File System

**`src/skill-lock.ts`**: Tracks installed skills for update detection

Lock file location: `~/.agents/.skill-lock.json`

Schema version 3:
- `skillFolderHash`: GitHub tree SHA for entire skill folder
- Tracks `source`, `sourceType`, `sourceUrl`, `installedAt`, `updatedAt`
- Stores dismissed prompts and last-selected agents

Update checking:
- POST to `https://add-skill.vercel.sh/check-updates`
- Always sends `forceRefresh: true` to bypass cache
- API fetches fresh content, computes hash, compares to lock file

### CLI Commands

- **`add`** (`src/add.ts`): Install skills from sources
- **list** (`src/list.ts`): List installed skills
- **find** (`src/find.ts`): Interactive search with fzf-style prompt
- **remove** (`src/remove.ts`): Remove installed skills
- **check/update**: Use lock file + API to detect and apply updates
- **init**: Create new SKILL.md template
- **generate-lock**: Match installed skills to sources via skills.sh API

### Testing

- Uses vitest framework
- Test files co-located: `src/*.test.ts` and `tests/*.test.ts`
- Key test suites:
  - Security: `sanitize-name.test.ts` (path traversal prevention)
  - Integration: `installer-symlink.test.ts`, `list-installed.test.ts`
  - Unit: `source-parser.test.ts`, `skill-matching.test.ts`

## Important Patterns

### Internal Skills

Skills can be marked as internal in frontmatter:
```yaml
---
name: my-internal-skill
description: Hidden by default
metadata:
  internal: true
---
```

Only shown/installed when `INSTALL_INTERNAL_SKILLS=1` is set.

### Multi-Word Skill Names

When installing specific skills with spaces in names, quotes are required:
```bash
npx skills add owner/repo --skill "My Skill Name"
```

### Agent Detection

The CLI automatically detects installed agents by checking for their config directories. If none detected, prompts user to select.

### Path Safety

All user-provided skill names are sanitized before filesystem operations:
- Converts to lowercase
- Replaces special chars with hyphens
- Removes path traversal attempts (`../`)
- Limits to 255 chars

### Telemetry

Anonymous usage tracking is enabled by default (disabled in CI). Can be disabled via `DISABLE_TELEMETRY` or `DO_NOT_TRACK` env vars.
