---
name: library
description: Private agentics distribution system. Use when the user wants to install, use, add, push, remove, sync, list, or search for skills, agents, prompts, or plugins from their private library catalog. Triggers on /library commands or mentions of library, skill distribution, or agentic management.
argument-hint: "[command or prompt] [name or details]"
---

# The Library

A meta-skill for private-first distribution of agentics (skills, agents, prompts, and plugins) across agents, devices, and teams.

## Variables

> Update these after forking and cloning the library repo.

- **LIBRARY_REPO_URL**: `git@github.com:QuentinAd/the-library.git`
- **LIBRARY_YAML_PATH**: `~/.agents/skills/library/library.yaml`
- **LIBRARY_SKILL_DIR**: `~/.agents/skills/library/`

## How It Works

The Library is a catalog of references to your agentics. The `library.yaml` file points to where skills, agents, and prompts live (local filesystem or GitHub repos). Nothing is fetched until you ask for it.

**The `library.yaml` is a catalog, not a manifest.** Entries define what's *available* — not what gets installed. You pull specific items on demand with `/library use <name>`.

## Commands

| Command                     | Purpose                                  |
| --------------------------- | ---------------------------------------- |
| `/library install`          | First-time setup: fork, clone, configure |
| `/library add <details>`    | Register a new entry in the catalog      |
| `/library use <name>`       | Pull from source (install or refresh)    |
| `/library push <name>`      | Push local changes back to source        |
| `/library remove <name>`    | Remove from catalog and optionally local |
| `/library list`             | Show full catalog with install status    |
| `/library sync`             | Re-pull all installed items from source   |
| `/library search <keyword>` | Find entries by keyword                  |

## Cookbook

Each command has a detailed step-by-step guide. **Read the relevant cookbook file before executing a command.**

| Command           | Cookbook                                          | Use When                                                     |
| ----------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| install           | [cookbook/install.md](cookbook/install.md)        | First-time setup on a new device                             |
| add               | [cookbook/add.md](cookbook/add.md)                | Register a new skill/agent/prompt/plugin in the catalog      |
| use               | [cookbook/use.md](cookbook/use.md)                | Pull or refresh an item from the catalog                     |
| use (plugin path) | [cookbook/install-plugin.md](cookbook/install-plugin.md) | Plugin-specific install flow via `claude plugin` CLI    |
| push              | [cookbook/push.md](cookbook/push.md)              | Push local skill changes back to source (skills/agents/prompts only — plugins update through Claude Code's plugin update) |
| remove            | [cookbook/remove.md](cookbook/remove.md)          | Remove an entry from the catalog                             |
| list              | [cookbook/list.md](cookbook/list.md)              | See what's available and what's installed                    |
| sync              | [cookbook/sync.md](cookbook/sync.md)              | Refresh all installed items at once                          |
| search            | [cookbook/search.md](cookbook/search.md)          | Find an entry by keyword                                     |

**When a user invokes a `/library` command, read the matching cookbook file first, then execute the steps.**

## Source Format

The `source` field in `library.yaml` supports these formats (auto-detected from the entry's type):

### Skills, agents, prompts (source points to a single file)
- `/absolute/path/to/SKILL.md` — local filesystem
- `https://github.com/org/repo/blob/main/path/to/SKILL.md` — GitHub browser URL
- `https://raw.githubusercontent.com/org/repo/main/path/to/SKILL.md` — GitHub raw URL

The source points to a specific file (`SKILL.md`, `AGENT.md`, or prompt file). The install copies the entire parent directory, not just the file.

### Plugins (source points to a marketplace directory)
- `/absolute/path/to/marketplace-dir/` — local filesystem, must contain `.claude-plugin/marketplace.json`
- `https://github.com/org/repo` — whole repo is the marketplace
- `https://github.com/org/repo/tree/branch/sub/path` — marketplace at a subpath in the repo

Plugin entries also require a `marketplace_name` field that matches the `name` field inside `marketplace.json`. The install shells out to `claude plugin marketplace add <source>` and `claude plugin install <plugin>@<marketplace_name>`. The library never copies plugin files directly — Claude Code's plugin loader owns the cache.

For private GitHub repos, use SSH or `GITHUB_TOKEN` for auth automatically (this applies to both file-based sources and plugin marketplace sources).

## Source Parsing Rules

**Local paths** start with `/` or `~`:
- Use the path directly. Copy the parent directory of the referenced file.

**GitHub browser URLs** match `https://github.com/<org>/<repo>/blob/<branch>/<path>`:
- Parse: `org`, `repo`, `branch`, `file_path`
- Clone URL: `https://github.com/<org>/<repo>.git`
- File location within repo: `<path>`

**GitHub raw URLs** match `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`:
- Parse: `org`, `repo`, `branch`, `file_path`
- Clone URL: `https://github.com/<org>/<repo>.git`
- File location within repo: `<path>`

## GitHub Workflow

When working with GitHub sources, prefer `gh api` for accessing single files (e.g., reading a SKILL.md to check metadata). For pulling entire skill directories, clone into a temp dir per the steps below.

**Fetching (use):**
1. Clone the repo with `git clone --depth 1 <clone_url>` into a temporary directory
2. Navigate to the parent directory of the referenced file
3. Copy that entire directory to the target local directory
4. The temporary directory is cleaned up automatically

**Pushing (push):**
1. Clone the repo with `git clone --depth 1 <clone_url>` into a temporary directory
2. Overwrite the skill directory in the clone with the local version
3. Stage only the relevant changes: `git add <skill_directory_path>`
4. Commit with message: `library: updated <skill name> <what changed>`
5. Push to remote
6. The temporary directory is cleaned up automatically

## Typed Dependencies

The `requires` field uses typed references to avoid ambiguity:
- `skill:name` — references a skill in the library catalog
- `agent:name` — references an agent in the library catalog
- `prompt:name` — references a prompt in the library catalog

When resolving dependencies: look up each reference in `library.yaml`, fetch all dependencies first (recursively), then fetch the requested item.

**Plugins do NOT use `requires`.** Plugin-to-plugin dependencies are declared in the plugin's own manifest (`plugin.json`) and resolved by Claude Code's plugin loader.

## Target Directories

By default, **skill / agent / prompt** items are installed to the **default** directory from `library.yaml`:

```yaml
default_dirs:
    skills:
        - default: .claude/skills/
        - global: ~/.agents/skills/
    agents:
        - default: .claude/agents/
        - global: ~/.agents/agents/
    prompts:
        - default: .claude/commands/
        - global: ~/.agents/prompts/
```

- If the user says "global" or "globally", use the `global` directory.
- If the user specifies a custom path, use that path.
- Otherwise, use the `default` directory.

**Plugins** bypass this entirely. Claude Code's plugin loader owns the install path (`~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`) and the library cookbook never touches it directly. See [cookbook/install-plugin.md](cookbook/install-plugin.md) for the install flow.

## Symlinks

The `symlinks` block in `library.yaml` defines directories that should mirror the global install location via symlinks. After every `use` (global install or refresh) and `sync`, apply symlinks for the installed item:

```yaml
symlinks:
  - from: ~/.agents/skills/
    to: ~/.claude/skills/
```

**Applying symlinks (after global install):**
- For each entry in `symlinks` where `from` matches the install directory:
  - Resolve `~` in both paths
  - Ensure the `to` directory exists: `mkdir -p <to_dir>`
  - Create a symlink for the item: `ln -sfn <from_dir>/<name> <to_dir>/<name>`
- This means one install propagates to all configured harnesses automatically.

**On remove (if local copy deleted):**
- For each matching `symlinks` entry, also remove the symlink:
  ```bash
  rm -f <to_dir>/<name>
  ```

**Symlinks are only applied for global installs**, not default (project-local) installs.

## Library Repo Sync

The library skill itself lives in `<LIBRARY_SKILL_DIR>` as a cloned git repo. When running `add` (which modifies `library.yaml`), always:
1. `git pull` in the library directory first to get latest
2. Make the changes
3. `git add library.yaml && git commit && git push`

This keeps the catalog in sync across devices.

## Example Filled Library File

```yaml
default_dirs:
  skills:
    - default: .claude/skills/
    - global: ~/.claude/skills/
  agents:
    - default: .claude/agents/
    - global: ~/.claude/agents/
  prompts:
    - default: .claude/prompts/
    - global: ~/.claude/prompts/

library:
  skills:
    - name: firecrawl
      description: Scrape, crawl, and search websites using Firecrawl CLI
      source: /Users/me/projects/tools/skills/firecrawl/SKILL.md

    - name: meta-skill
      description: Creates new Agent Skills following best practices
      source: /Users/me/projects/tools/skills/meta-skill/SKILL.md

    - name: diagram-kroki
      description: Generate diagrams via Kroki HTTP API supporting 28+ languages
      source: https://github.com/myorg/private-skills/blob/main/skills/diagram-kroki/SKILL.md
      requires: [skill:firecrawl]

    - name: green-screen-captions
      description: Generate and burn AI-powered captions onto green screen videos
      source: https://raw.githubusercontent.com/myorg/video-tools/main/skills/green-screen-captions/SKILL.md
      requires: [agent:video-processor, prompt:caption-style]

  agents:
    - name: video-processor
      description: Processes video files with ffmpeg and whisper transcription
      source: /Users/me/projects/tools/agents/video-processor/AGENT.md

    - name: code-reviewer
      description: Reviews code for quality, security, and performance
      source: https://github.com/myorg/agent-configs/blob/main/agents/code-reviewer/AGENT.md

  prompts:
    - name: caption-style
      description: Style guide for generating video captions
      source: /Users/me/projects/content/prompts/caption-style.md

    - name: commit-message
      description: Standardized commit message format for all projects
      source: https://github.com/myorg/team-prompts/blob/main/prompts/commit-message.md
```
