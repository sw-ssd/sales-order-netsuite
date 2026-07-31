# Git Submodule Conversion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the existing `sales-order-backend`, `sales-order-frontend`, and `sales-order-app` directories into proper git submodules of a new parent repo rooted at `/Volumes/UTM2/Developer/sales-order-netsuite`, while preserving local-only config/credential files and keeping the existing root orchestration files.

**Architecture:** The parent repo will own only orchestration (root `Taskfile.yml`, `.gitignore`, `openspec/`, `appimg/`, `.omp/`, docs). Each subproject becomes a submodule that tracks its remote `master` branch via `.gitmodules` and a gitlink in the parent index. The conversion is done in-place: existing subproject directories are moved to an external backup, submodules are cloned from remotes, and needed local files are copied back.

**Tech Stack:** Git, GitHub HTTPS remotes, Taskfile.dev (verification only).

## Global Constraints

- Parent repo is local-only; no remote origin is configured in this task.
- Three subprojects must track their respective `master` branches.
- Remote URLs are fixed:
  - `https://github.com/hexagon-maker/sales-order-backend.git`
  - `https://github.com/hexagon-maker/sales-order-frontend.git`
  - `https://github.com/hexagon-maker/sales-order-app.git`
- Local-only files that must survive the conversion:
  - `sales-order-backend/.env`
  - `sales-order-backend/cmd/sw8/.env`
  - `sales-order-frontend/.env`
  - `sales-order-frontend/.env.production`
  - `sales-order-app/android/keystore/hexagon-salesorder-keystore.jks`
- No changes are pushed to the three subproject remotes.
- No CI/CD or application code changes.
- Commit author for parent repo commits in this plan: `omp-agent <omp-agent@local>`.

## File Structure

Files that exist before the plan:
- `sales-order-backend/` — independent git repo (will become submodule)
- `sales-order-frontend/` — independent git repo (will become submodule)
- `sales-order-app/` — independent git repo (will become submodule)
- `Taskfile.yml` — parent orchestration (keep in parent)
- `.gitignore` — parent ignore rules (keep in parent)
- `openspec/` — parent docs/specs (keep in parent)
- `appimg/` — parent assets (keep in parent)
- `.omp/` — harness config (keep in parent)
- `docs/superpowers/specs/2026-07-31-git-submodule-conversion-design.md` — design spec (already committed to parent)

Files created/modified by this plan:
- `.gitmodules` — created by `git submodule add`
- Parent repo index — records root files and submodule gitlinks
- Backup directory outside the repo: `../sales-order-netsuite-backup-2026-07-31/`

---

### Task 1: Backup existing subproject directories

**Files:**
- No files created or modified inside the repo.
- Creates external backup directory: `../sales-order-netsuite-backup-2026-07-31/`

**Interfaces:**
- Consumes: current `sales-order-backend/`, `sales-order-frontend/`, `sales-order-app/`
- Produces: backup copy of those three directories

- [ ] **Step 1: Verify each subproject is on master and clean**

Run:
```bash
for d in sales-order-backend sales-order-frontend sales-order-app; do
  echo "=== $d ==="
  git -C "$d" branch --show-current
  git -C "$d" status --short
done
```

Expected output:
```
=== sales-order-backend ===
master

=== sales-order-frontend ===
master

=== sales-order-app ===
master

```
(no status lines means working tree is clean)

- [ ] **Step 2: Create the external backup directory**

Run:
```bash
mkdir -p ../sales-order-netsuite-backup-2026-07-31
```

Expected: command succeeds with no output.

- [ ] **Step 3: Move the three directories into the backup**

Run:
```bash
mv sales-order-backend sales-order-frontend sales-order-app ../sales-order-netsuite-backup-2026-07-31/
```

Expected: command succeeds with no output.

- [ ] **Step 4: Confirm the backup contains all three directories**

Run:
```bash
ls ../sales-order-netsuite-backup-2026-07-31/
```

Expected output (order may vary):
```
sales-order-app       sales-order-backend   sales-order-frontend
```

- [ ] **Step 5: Confirm the three directories no longer exist in the parent repo**

Run:
```bash
ls sales-order-backend sales-order-frontend sales-order-app 2>&1
```

Expected output (paths do not exist):
```
ls: sales-order-backend: No such file or directory
ls: sales-order-frontend: No such file or directory
ls: sales-order-app: No such file or directory
```

---

### Task 2: Commit parent repo orchestration files

**Files:**
- Modify: parent repo index and `.git/` object store

**Interfaces:**
- Consumes: `Taskfile.yml`, `.gitignore`, `openspec/`, `appimg/`, `.omp/`, `docs/`
- Produces: a parent repo commit containing those root files

- [ ] **Step 1: Stage the root orchestration files**

Run:
```bash
git add .gitignore Taskfile.yml openspec/ appimg/ .omp/ docs/
```

Expected: command succeeds with no output.

- [ ] **Step 2: Review what is staged**

Run:
```bash
git status --short
```

Expected output contains lines like:
```
A  .gitignore
A  Taskfile.yml
A  appimg/...
A  docs/superpowers/specs/...
A  openspec/...
...
```
There should be **no** entries for `sales-order-backend`, `sales-order-frontend`, or `sales-order-app`.

- [ ] **Step 3: Commit the parent orchestration files**

Run:
```bash
git -c user.name="omp-agent" -c user.email="omp-agent@local" commit -m "chore: add parent repo orchestration files"
```

Expected output contains a successful commit summary, e.g.:
```
[main 1a2b3c4] chore: add parent repo orchestration files
 ... files changed, ... insertions(+)
```

---

### Task 3: Add the three submodules

**Files:**
- Create: `.gitmodules`
- Modify: parent repo index (records submodule gitlinks)

**Interfaces:**
- Consumes: remote submodule URLs on `master`
- Produces: `sales-order-backend/`, `sales-order-frontend/`, `sales-order-app/` as populated submodules

- [ ] **Step 1: Add backend submodule**

Run:
```bash
git submodule add -b master https://github.com/hexagon-maker/sales-order-backend.git sales-order-backend
```

Expected: clone progress output and a new directory `sales-order-backend/`.

- [ ] **Step 2: Add frontend submodule**

Run:
```bash
git submodule add -b master https://github.com/hexagon-maker/sales-order-frontend.git sales-order-frontend
```

Expected: clone progress output and a new directory `sales-order-frontend/`.

- [ ] **Step 3: Add app submodule**

Run:
```bash
git submodule add -b master https://github.com/hexagon-maker/sales-order-app.git sales-order-app
```

Expected: clone progress output and a new directory `sales-order-app/`.

- [ ] **Step 4: Inspect `.gitmodules`**

Run:
```bash
cat .gitmodules
```

Expected output:
```ini
[submodule "sales-order-backend"]
	path = sales-order-backend
	url = https://github.com/hexagon-maker/sales-order-backend.git
	branch = master
[submodule "sales-order-frontend"]
	path = sales-order-frontend
	url = https://github.com/hexagon-maker/sales-order-frontend.git
	branch = master
[submodule "sales-order-app"]
	path = sales-order-app
	url = https://github.com/hexagon-maker/sales-order-app.git
	branch = master
```

- [ ] **Step 5: Commit submodules**

Run:
```bash
git -c user.name="omp-agent" -c user.email="omp-agent@local" commit -m "chore: add backend, frontend and app as submodules"
```

Expected output contains a successful commit summary with `.gitmodules` and the three submodule paths.

---

### Task 4: Restore local-only files from backup

**Files:**
- Create inside submodules:
  - `sales-order-backend/.env`
  - `sales-order-backend/cmd/sw8/.env`
  - `sales-order-frontend/.env`
  - `sales-order-frontend/.env.production`
  - `sales-order-app/android/keystore/hexagon-salesorder-keystore.jks`

**Interfaces:**
- Consumes: files from `../sales-order-netsuite-backup-2026-07-31/`
- Produces: restored local-only config/credential files inside each submodule

- [ ] **Step 1: Copy each local-only file back to its original location**

Run:
```bash
cp -a ../sales-order-netsuite-backup-2026-07-31/sales-order-backend/.env sales-order-backend/.env
cp -a ../sales-order-netsuite-backup-2026-07-31/sales-order-backend/cmd/sw8/.env sales-order-backend/cmd/sw8/.env
cp -a ../sales-order-netsuite-backup-2026-07-31/sales-order-frontend/.env sales-order-frontend/.env
cp -a ../sales-order-netsuite-backup-2026-07-31/sales-order-frontend/.env.production sales-order-frontend/.env.production
cp -a ../sales-order-netsuite-backup-2026-07-31/sales-order-app/android/keystore/hexagon-salesorder-keystore.jks sales-order-app/android/keystore/hexagon-salesorder-keystore.jks
```

Expected: each command succeeds with no output. If a source file does not exist, the corresponding `cp` will fail; in that case skip that file and note it in the final summary.

- [ ] **Step 2: Verify all restored files exist and are non-empty**

Run:
```bash
for f in \
  sales-order-backend/.env \
  sales-order-backend/cmd/sw8/.env \
  sales-order-frontend/.env \
  sales-order-frontend/.env.production \
  sales-order-app/android/keystore/hexagon-salesorder-keystore.jks
do
  if [ -s "$f" ]; then
    echo "OK: $f"
  else
    echo "MISSING OR EMPTY: $f"
  fi
done
```

Expected output (all five files should be OK):
```
OK: sales-order-backend/.env
OK: sales-order-backend/cmd/sw8/.env
OK: sales-order-frontend/.env
OK: sales-order-frontend/.env.production
OK: sales-order-app/android/keystore/hexagon-salesorder-keystore.jks
```

---

### Task 5: Configure submodule helper settings

**Files:**
- Modify: `.git/config` (local git config)

**Interfaces:**
- Consumes: existing parent repo
- Produces: updated local git config for easier submodule handling

- [ ] **Step 1: Enable recursive submodule handling for pull/fetch**

Run:
```bash
git config submodule.recurse true
```

Expected: command succeeds with no output.

- [ ] **Step 2: Prevent pushing parent when submodules have unpushed commits**

Run:
```bash
git config push.recurseSubmodules check
```

Expected: command succeeds with no output.

- [ ] **Step 3: Confirm config values**

Run:
```bash
git config --local --get submodule.recurse
git config --local --get push.recurseSubmodules
```

Expected output:
```
true
check
```

---

### Task 6: Verify and clean up

**Files:**
- Delete external backup directory: `../sales-order-netsuite-backup-2026-07-31/`

**Interfaces:**
- Consumes: parent repo and submodules
- Produces: final verified state

- [ ] **Step 1: Check submodule status**

Run:
```bash
git submodule status
```

Expected output: three lines, each containing a 40-character commit hash, the submodule path, and a reference to `master`, e.g.:
```
 0199e701cf5757e55661cce2ff0a90259870a917 sales-order-backend (v0.0.1-123-g0199e70)
 b851d202274bdde91c6f989b8d85695f8774fa67 sales-order-frontend (v0.0.1-45-gb851d20)
 9c4bfa7087cb9fcf32074b4eefdbace688f6e9a7 sales-order-app (v1.2.6+25)
```

- [ ] **Step 2: Check each submodule working tree is clean**

Run:
```bash
for d in sales-order-backend sales-order-frontend sales-order-app; do
  echo "=== $d ==="
  git -C "$d" status --short
done
```

Expected: only headers. Any restored untracked files may appear as `??` lines; that is expected and acceptable because they are gitignored by the submodules.

- [ ] **Step 3: Check parent repo status**

Run:
```bash
git status --short
```

Expected: empty output (clean parent working tree). Submodule directories should not appear as untracked because they are registered gitlinks.

- [ ] **Step 4: Verify root Taskfile include paths still resolve**

Run:
```bash
test -f sales-order-backend/Taskfile.yml && echo "OK backend Taskfile"
test -f sales-order-frontend/Taskfile.yml && echo "OK frontend Taskfile"
test -f sales-order-app/Taskfile.yml && echo "OK app Taskfile"
```

Expected output:
```
OK backend Taskfile
OK frontend Taskfile
OK app Taskfile
```

If `task` is installed, you may also run `task -l` and confirm it lists tasks without errors.

- [ ] **Step 5: Remove the external backup directory**

Run:
```bash
rm -rf ../sales-order-netsuite-backup-2026-07-31
```

Expected: command succeeds with no output.

- [ ] **Step 6: Confirm backup is gone**

Run:
```bash
test -d ../sales-order-netsuite-backup-2026-07-31 && echo "STILL EXISTS" || echo "BACKUP REMOVED"
```

Expected output:
```
BACKUP REMOVED
```

---

## Plan Self-Review

**Spec coverage:**
- Initialize parent repo: Task 2 covers committing root orchestration files (parent repo is already initialized before the plan starts).
- Convert three subprojects to submodules tracking `master`: Task 3.
- Preserve root orchestration files: Task 2 explicitly stages `Taskfile.yml`, `.gitignore`, `openspec/`, `appimg/`, `.omp/`, `docs/`.
- Backup and restore local-only files: Task 1 (backup) and Task 4 (restore).
- No remote for parent repo: no origin setup anywhere in the plan.
- No push to subproject remotes: plan only clones, never pushes.

**Placeholder scan:**
- No TBD, TODO, "implement later", or vague steps.
- Every command is exact and includes expected output.
- URLs and paths are concrete.

**Type / path consistency:**
- Backup path is consistent across all tasks: `../sales-order-netsuite-backup-2026-07-31/`.
- Submodule paths are consistent: `sales-order-backend`, `sales-order-frontend`, `sales-order-app`.
- Remote URLs match the approved spec exactly.

**Gaps:** None identified. If any of the five local-only files do not exist in the backup, the corresponding `cp` command will fail; the implementer should skip that file and report it, but the plan continues.

---

## Execution Handoff

Plan saved to `docs/superpowers/plans/2026-07-31-git-submodule-conversion.md`.

Two execution options:

1. **Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — execute tasks in this session using `superpowers:executing-plans` with checkpoints.

Which approach would you like to use?