## Why

App Taskfile (`sales-order-app/Taskfile.yml`) contains hardcoded absolute paths referencing `~/Documents/sales-order-netsuit/` — a stale location from when the repo lived under a different mount. The current project root is `/Volumes/UTM2/Developer/sales-order-netsuite/`. These hardcoded paths cause the `screenshots:ios`, `screenshots:android`, `screenshots:export:ios` tasks to fail with path-not-found errors.

Additionally, the root `Taskfile.yml` include `dir` directives and the VSCode sessions.json absolute paths should be reviewed for consistency with how Task resolves working directories per the included subproject Taskfiles.

## What Changes

- **App Taskfile**: Replace hardcoded `~/Documents/sales-order-netsuit/` absolute paths with `{{.ROOT_DIR}}` or `{{.TASKFILE_DIR}}` Taskfile variables in the `screenshots:android`, `screenshots:ios`, and `screenshots:export:ios` tasks
- **Root Taskfile**: Review and verify include `dir` settings work correctly when invoked from repo root
- **VSCode sessions.json**: Review absolute path consistency — paths currently match the correct location

## Capabilities

### New Capabilities
(無)

### Modified Capabilities
(無 — 純工具修正，不涉及產品規格變更)
