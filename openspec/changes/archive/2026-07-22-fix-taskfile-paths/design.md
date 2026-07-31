## Context

The repo moved from `~/Documents/sales-order-netsuit/` to `/Volumes/UTM2/Developer/sales-order-netsuite/`. The app Taskfile (`sales-order-app/Taskfile.yml`) still has 5 hardcoded references to the old path, making screenshot and export tasks fail.

The root `Taskfile.yml` includes subproject Taskfiles with `dir:` settings that are relative to the root Taskfile path. The VSCode sessions.json uses absolute paths matching the current correct location.

### Hardcoded paths found

| Line | Task | Old Path |
|------|------|----------|
| 170 | `screenshots:android` | `~/Documents/sales-order-netsuit/sales-order-backend` |
| 179 | `screenshots:ios` | `~/Documents/sales-order-netsuit/sales-order-backend` |
| 196 | `screenshots:export:ios` | `file:///Users/ssd/Documents/sales-order-netsuit/appimg/screenshots.butterkit/` |
| 197 | `screenshots:export:ios` | `~/Documents/sales-order-netsuit/appimg/export_app_store` |
| 199 | `screenshots:export:ios` | `~/Documents/sales-order-netsuit/appimg/export_app_store` |

## Goals / Non-Goals

**Goals:**
- All hardcoded old paths in app Taskfile replaced with `{{.TASKFILE_DIR}}`-relative equivalents
- Root Taskfile include `dir` configurations verified for correctness
- Screenshot and export tasks work from the current repo root

**Non-Goals:**
- No changes to VSCode sessions.json (paths are already correct)
- No changes to backend/frontend Taskfile logic (no broken paths there)
- No restructuring or renaming of Taskfile tasks

## Decisions

### Decision: Use `{{.TASKFILE_DIR}}/..` as project root reference

`{{.TASKFILE_DIR}}` resolves to the directory containing the app Taskfile (`sales-order-app/`). The project root is one level up (`..`). Define a `PROJECT_ROOT` var at the top of the file for reuse.

**Alternatives considered:**
- `{{.ROOT_DIR}}` — resolves to the Taskfile's own directory, same as `{{.TASKFILE_DIR}}`, not the git root. Not suitable.
- Hardcoded absolute paths — fragile, breaks on any location change. Current problem.
- Symlink — adds git-ignored artifact, confusing. Rejected.

### Decision: Keep error messages as relative path hints

The `screenshots:android/ios` tasks have error messages telling the user where to cd. These will use `{{.PROJECT_ROOT}}/sales-order-backend` so the message works regardless of where the repo is checked out.

### Decision: ButterKit file:// URL uses explicit construction

The `documentId` is a `file://` URL inside a JSON string. Construct it as:
`file://{{.PROJECT_ROOT | replace " " "%20"}}/appimg/screenshots.butterkit/`
to dynamically resolve. This avoids hardcoding the absolute path.

## Risks / Trade-offs

- **`{{.TASKFILE_DIR}}` depends on how Task is invoked**: If run via `task screenshots:android` from the repo root's `task app:*` include, `TASKFILE_DIR` still points to the app Taskfile's directory (the included `dir:` sets the working dir, but the taskfile variable is about the *file* location). Verified: Task v3 always resolves `TASKFILE_DIR` to the Taskfile's own directory, regardless of how it was invoked.
- **ButterKit JSON escaping**: The `file://` URL construction inside a multi-line shell command + JSON string needs careful quoting. If `PROJECT_ROOT` contains spaces, the `replace` filter handles it for the URL, but the shell context may need additional quoting.
