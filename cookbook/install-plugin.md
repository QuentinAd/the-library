# Install a Plugin from the Library

## Context
Install a plugin entry from `library.yaml`. Plugins are Claude Code plugins (with `.claude-plugin/plugin.json`) that live inside a marketplace (with `.claude-plugin/marketplace.json`). They are managed by Claude Code's own plugin system — no manual file-copy is performed. Instead, the library cookbook shells out to the `claude plugin` CLI.

## Why this is different from skills/agents/prompts
Skills, agents, and prompts are installed by copying files into a directory (`.claude/skills/<name>/`, `.claude/agents/<name>.md`, etc.). Plugins go through Claude Code's plugin loader, which caches them under `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/` and tracks state in `~/.claude/plugins/installed_plugins.json`. Don't manipulate those files directly — use the CLI.

## Input
The user provides a plugin name. The cookbook reads the entry from `library.yaml`.

## Steps

### 1. Sync the Library Repo
```bash
cd <LIBRARY_SKILL_DIR>
git pull
```

### 2. Find the Plugin Entry
- Read `library.yaml`
- Look up the name under `library.plugins`
- If not found, tell the user the entry doesn't exist; suggest `/library search` or `/library add`
- If found, capture: `name`, `source`, `marketplace_name`

### 3. Validate the Source
The `source` field of a plugin entry must point to a **marketplace directory** — i.e., a directory containing `.claude-plugin/marketplace.json`. Two formats are supported:

**Local path** (starts with `/` or `~`):
```bash
src="<expand ~>"
[ -f "$src/.claude-plugin/marketplace.json" ] || { echo "missing marketplace.json at $src"; exit 1; }
```

**GitHub URL** (browser-form):
- Pattern: `https://github.com/<org>/<repo>` (whole repo is the marketplace)
- Or: `https://github.com/<org>/<repo>/tree/<branch>/<sub/path>` (marketplace lives at a subpath)
- The `claude plugin marketplace add` CLI accepts both forms directly — let it do the validation.

### 4. Register the Marketplace
The `claude plugin marketplace add` command is idempotent — re-running it on an already-registered marketplace either no-ops or refreshes.

```bash
claude plugin marketplace add "<source>"
```

After the command, verify by listing:
```bash
claude plugin marketplace list
```

Confirm `<marketplace_name>` (from the entry) appears in the list. If it doesn't, the marketplace's `marketplace.json` `name` field doesn't match what we expected — surface this to the user.

### 5. Install the Plugin
```bash
claude plugin install "<plugin_name>@<marketplace_name>"
```

The `@<marketplace_name>` suffix disambiguates when multiple marketplaces could provide a plugin with the same name. Always include it.

If the plugin is already installed, `claude plugin install` will refresh it.

### 6. Verify Installation
```bash
claude plugin list
```

The output should include `<plugin_name>@<marketplace_name>`. Also check:
```bash
ls ~/.claude/plugins/cache/<marketplace_name>/<plugin_name>/ 2>/dev/null
```

A non-empty cache directory confirms the plugin's files are in place.

### 7. Confirm to User
Tell the user:
- The marketplace that was registered (if newly registered)
- The plugin installed (with `@<marketplace>` suffix)
- That the plugin's slash commands (typically `/<plugin>:<command>`) are now available
- A reminder to **restart Claude Code** if the new commands don't show up immediately — plugin discovery happens at session start

## Notes

**For refresh (re-install)**: `claude plugin install` is idempotent and pulls the latest. For a clean reinstall, run `claude plugin uninstall <plugin>@<marketplace>` first, then install again.

**For dependencies**: Plugins may declare dependencies on other plugins via their own manifest. Claude Code handles those automatically; the library cookbook does not need to recurse.

**Side effects**: `claude plugin marketplace add` clones the marketplace repo (if remote) into `~/.claude/plugins/marketplaces/<marketplace_name>/`. `claude plugin install` clones or symlinks the plugin into `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`. Both are managed by Claude Code; the library cookbook never touches these paths directly.

**Local marketplaces and git**: When the marketplace `source` is a local path, `claude plugin marketplace add` registers the directory by path. Updates to the source directory are picked up automatically (no re-add needed). However, if the marketplace.json itself changes, you may need `claude plugin marketplace update <marketplace_name>`.
