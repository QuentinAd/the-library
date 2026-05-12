# Add a New Entry to the Library

## Context
Register a new skill, agent, or prompt in the library catalog.

## Input
The user provides: name, description, source, and optionally type and dependencies.

## Steps

### 1. Sync the Library Repo
Pull the latest changes before modifying:
```bash
cd <LIBRARY_SKILL_DIR>
git pull
```

### 2. Determine the Type
Figure out the type from the user's prompt or the source path:
- If the source path contains `SKILL.md` or user says "skill" → type is `skill`
- If the source path contains `AGENT.md` or user says "agent" → type is `agent`
- If user says "prompt" → type is `prompt`
- If the source points to a **directory** containing `.claude-plugin/marketplace.json`, or user says "plugin" → type is `plugin`
- If ambiguous, ask the user

### 3. Validate the Source

**For skill / agent / prompt** (source is a specific file):
- **Local path**: Verify the file exists at the given path
- **GitHub URL**: Verify the URL is well-formed (matches browser or raw URL patterns)
- Confirm the source points to a specific file, not a directory

**For plugin** (source is a marketplace directory):
- **Local path**: Verify the directory contains `.claude-plugin/marketplace.json` (read the file and capture the `name` field — needed for the entry's `marketplace_name`)
- **GitHub URL**: Should point at a directory in a repo (the marketplace root). Examples:
  - `https://github.com/<org>/<repo>` (whole repo is the marketplace)
  - `https://github.com/<org>/<repo>/tree/<branch>/<sub/path>` (marketplace at a subpath)
  - For GitHub URLs, optionally clone the repo to a temp dir and read `<sub>/.claude-plugin/marketplace.json` to capture the `name` field
- Also capture the plugin's own `name` (from its `.claude-plugin/plugin.json` inside `plugins/<name>/`). This may differ from the catalog entry name — but they usually match.

### 4. Parse Dependencies
Detect dependencies by looking through the skill/agent/prompt files, format them as typed references:
- `skill:name`, `agent:name`, `prompt:name`
- Verify each dependency already exists in `library.yaml` if or warn the user
  - If they don't exist add them to `library.yaml` first. If those files have dependencies, add them recursively.
  - You can detect these sometimes by looking at the frontmatter, and then in the file content look for `/<prompt|agent|skill>:name` references. If you're not sure, ask the user the user if they have any dependencies.

### 5. Add the Entry to library.yaml
Read `library.yaml`, add the new entry under the correct section:

```yaml
# Under library.skills, library.agents, library.prompts, or library.plugins
- name: <name>
  description: <description>
  source: <source>
  marketplace_name: <name>     # PLUGIN ONLY — must match marketplace.json's name field
  requires: [<typed:refs>]     # omit if no dependencies (skills/agents/prompts only)
```

**YAML formatting rules:**
- 2-space indentation
- List items use `- ` prefix
- Properties are indented under the list item
- Keep entries alphabetically sorted by name within each section
- For skills reference the `.../<skill-name>/SKILL.md` file,
- For agents reference the `.../<agent name>.md` file,
- For prompts reference the `.../<prompt name>.md` file (installed to `.claude/commands/`),
- For plugins reference the **marketplace directory** (the one containing `.claude-plugin/marketplace.json`), NOT a specific file
- Remember we'll be adding an absolute path or a github url (https or ssh)
- Plugin entries do NOT use `requires:` — plugin dependencies are declared in the plugin's own manifest and resolved by Claude Code

### 6. Commit and Push
```bash
cd <LIBRARY_SKILL_DIR>
git add library.yaml
git commit -m "library: added <type> <name>"
git push
```

### 7. Confirm
Tell the user the entry has been added and is now available for others to use via `/library use <name>`.
