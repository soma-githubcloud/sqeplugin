# /sqe-kit:scan-project — Scan Existing Project for Brownfield Mode

**Brownfield entry point.** Scans an existing Playwright TypeScript project to detect its
directory structure, naming conventions, and code patterns — then writes `.sqe/project-fingerprint.json`
to enable **brownfield mode** for all SQE Kit generation commands.

Run this once per project. After scanning, all `/sqe-kit:generate-automation-tests`, `/sqe-kit:e2e`,
and other commands automatically operate in brownfield mode for this project.

## Arguments

| Argument | Description |
|----------|-------------|
| `--project <path>` | Path to project root (default: current working directory) |
| `--dry-run` | Preview what would be detected without writing fingerprint |

## Examples

```bash
# Scan the current project
/sqe-kit:scan-project

# Scan a specific project directory
/sqe-kit:scan-project --project /path/to/my-playwright-project

# Preview detection without writing
/sqe-kit:scan-project --dry-run
```

---

## Execution

### Step 0: Setup

- `projectRoot` = `--project` argument OR current working directory

### Step 1: Run Project Scanner Agent

Read and execute `agents/project-scanner-agent.md` (resolve via PLUGIN_DIR). Pass:
```
projectRoot : resolved project root path
dryRun      : --dry-run flag
```

The project scanner agent:
1. Reads `skills/project-fingerprinter/SKILL.md` (resolve via PLUGIN_DIR) to scan the project
2. Detects: testStyle, POM dirs, spec dirs, feature dirs, import alias, constructor pattern, locator style
3. Lists all existing POMs, test specs, and feature files
4. Writes `.sqe/project-fingerprint.json`

### Step 2: Confirm Activation

If fingerprint was written successfully:
```
✅ Project Scan Complete

SQE Kit is now in brownfield mode for this project.
Fingerprint: .sqe/project-fingerprint.json

Run /sqe-kit:generate-automation-tests to generate artifacts
that integrate with your existing codebase.
```

---

## Important Rules

- This command ONLY reads the project — it never modifies existing code
- Safe to re-run: updates the fingerprint with the latest project state
- After scanning, all other sqe-kit commands detect brownfield mode automatically
- Use `/sqe-kit:scan-project` before `/sqe-kit:generate-automation-tests` for best results on existing projects
