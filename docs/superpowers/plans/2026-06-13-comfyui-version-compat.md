# ComfyUI Version Compatibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Merge the `legacy-comfyui` and `new-comfyui` branches into a single `main`, add a non-blocking frontend-version log line on plugin load, and update the README so users only need one download.

**Architecture:** `main` already contains the new code (commit `5da86e7`). The plan adds a small helper in `__init__.py` that prints `comfyui_frontend_package.__version__` to the console for diagnostic purposes (no version gate), and rewrites the README "Version Downloads" section to "System Requirements". Source files (`gaussian_viewer.py`, `render_gaussian.py`, JS files) need no change. Branch cleanup is handled at the end with explicit user confirmation.

**Tech Stack:** Python 3 (ComfyUI custom node entry point), Markdown.

**Spec:** `docs/superpowers/specs/2026-06-13-comfyui-version-compat-design.md`

**Working branch:** `pr-11-test` (already contains the spec commits). All implementation commits land here, then merge to `main`.

**Note on testing:** This plugin has no automated test framework (no pytest/jest config, no `tests/` folder). The plan does **not** introduce one — that would be out of scope. Verification is manual, via the user running ComfyUI with the plugin loaded. This is explicitly called out in the spec's "非目标" section.

---

## File Structure

| File | Responsibility | Change type |
|---|---|---|
| `__init__.py` | Plugin entry, node registration | Modify: add `_log_frontend_version()` and call it before existing imports |
| `README.md` | Chinese user-facing docs | Modify: remove "版本下载" section, add "系统要求" section |
| `README_EN.md` | English user-facing docs | Modify: remove "Version Downloads" section, add "System Requirements" section |

No new files. No deletions of source files. Branch cleanup at the end is git-only.

---

### Task 1: Add `_log_frontend_version()` to `__init__.py`

**Files:**
- Modify: `__init__.py` (insert helper + call between line 12 and line 14, i.e., after the `CAMERA_PARAMS_BY_KEY = {}` block and **before** the `from .gaussian_viewer import ...` line)

**Why before the imports:** the helper has no dependency on any plugin module. Placing it before the heavy imports means the version log appears first in the console, which makes it the first thing a user sees when they paste their startup log into an issue.

- [ ] **Step 1: Read the current `__init__.py` to confirm structure**

Run: `cat __init__.py`

Expected: matches the structure documented in the spec (header comment, `CAMERA_PARAMS_BY_KEY = {}`, then two `from .X import ...` lines, then `NODE_CLASS_MAPPINGS = {...}`, etc.)

- [ ] **Step 2: Edit `__init__.py` — insert the helper and call**

Use Edit tool. Find this exact block:

```python
# Shared camera params cache - must be at module level for persistence
CAMERA_PARAMS_BY_KEY = {}

from .gaussian_viewer import GaussianViewerNode, NODE_CLASS_MAPPINGS as VIEWER_MAPPINGS, NODE_DISPLAY_NAME_MAPPINGS as VIEWER_DISPLAY_MAPPINGS
```

Replace with:

```python
# Shared camera params cache - must be at module level for persistence
CAMERA_PARAMS_BY_KEY = {}


def _log_frontend_version():
    try:
        import comfyui_frontend_package
        version = getattr(comfyui_frontend_package, "__version__", None)
        if version:
            print(f"[GaussianViewer] ComfyUI frontend version: {version}")
    except ImportError:
        pass


_log_frontend_version()


from .gaussian_viewer import GaussianViewerNode, NODE_CLASS_MAPPINGS as VIEWER_MAPPINGS, NODE_DISPLAY_NAME_MAPPINGS as VIEWER_DISPLAY_MAPPINGS
```

- [ ] **Step 3: Manually verify the edit by reading the file back**

Run: `cat __init__.py`

Expected: file contains the new `_log_frontend_version` function and a `_log_frontend_version()` call placed before the first `from .gaussian_viewer import ...` line. The function body has exactly the structure shown above (try/except ImportError, getattr with None default, print only when version truthy).

- [ ] **Step 4: Quick syntax sanity check**

Run: `python -c "import ast; ast.parse(open('__init__.py').read()); print('OK')"`

Expected: prints `OK`. Any `SyntaxError` means the edit corrupted the file — re-read and fix.

- [ ] **Step 5: Commit**

```bash
git add __init__.py
git commit -m "feat: log ComfyUI frontend version at plugin load"
```

---

### Task 2: Update `README.md` (Chinese)

**Files:**
- Modify: `README.md` lines 7–14 (the "## 版本下载" section)
- Modify: `README.md` — insert a new "## 系统要求" section in the same location

- [ ] **Step 1: Edit `README.md` — replace the "版本下载" block**

Use Edit tool. Find this exact block (lines 7–14, including the trailing blank line):

```markdown
## 版本下载

本仓库在同一个 GitHub 项目中保留两个互不干扰的 ComfyUI 版本，README 共用这一份说明：

- 旧版 ComfyUI：下载 [`legacy-comfyui`](https://github.com/CarlMarkswx/comfyui_GaussianViewer/archive/refs/heads/legacy-comfyui.zip)
- 新版 ComfyUI：下载 [`new-comfyui`](https://github.com/CarlMarkswx/comfyui_GaussianViewer/archive/refs/heads/new-comfyui.zip)

如果你的 ComfyUI 还没有更新，请使用旧版；如果已经更新到新版 ComfyUI，请使用新版。
```

Replace with:

```markdown
## 系统要求

- ComfyUI（建议使用较新的版本）
- Python 包：numpy / torch / Pillow（通常随 ComfyUI 安装）

如果你的 ComfyUI 比较老，节点显示出现裁切（节点框比内容小、内容溢出），通常是 ComfyUI 前端版本过旧。两种解决方案：

1. **推荐**：升级到最新 ComfyUI
2. **兜底**：用 `--front-end-version` 启动参数锁定前端版本：
   ```bash
   python main.py --front-end-version Comfy-Org/ComfyUI_frontend@1.38.0
   ```

启动 ComfyUI 后，本插件会在控制台打印当前的 frontend 版本，便于排查问题。
```

- [ ] **Step 2: Verify the edit**

Run: `head -30 README.md`

Expected: The "## 版本下载" header is gone. The "## 系统要求" header appears between the license badge line and the "为 ComfyUI 提供高斯泼溅..." paragraph.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: replace 版本下载 with 系统要求 in README.md"
```

---

### Task 3: Update `README_EN.md` (English)

**Files:**
- Modify: `README_EN.md` lines 7–14 (the "## Version Downloads" section)
- Modify: `README_EN.md` — insert a new "## System Requirements" section in the same location

- [ ] **Step 1: Edit `README_EN.md` — replace the "Version Downloads" block**

Use Edit tool. Find this exact block (lines 7–14):

```markdown
## Version Downloads

This repository keeps two independent ComfyUI-compatible versions in the same GitHub project, with this README as the shared entry point:

- Older ComfyUI: download [`legacy-comfyui`](https://github.com/CarlMarkswx/comfyui_GaussianViewer/archive/refs/heads/legacy-comfyui.zip)
- Newer ComfyUI: download [`new-comfyui`](https://github.com/CarlMarkswx/comfyui_GaussianViewer/archive/refs/heads/new-comfyui.zip)

Use the legacy version if your ComfyUI has not been updated yet. Use the new version if your ComfyUI is already updated.
```

Replace with:

```markdown
## System Requirements

- ComfyUI (a recent version is recommended)
- Python packages: numpy / torch / Pillow (usually shipped with ComfyUI)

If your ComfyUI is older and the node renders truncated (the node frame is smaller than its content, and the content overflows), the ComfyUI frontend is likely too old. Two ways to fix:

1. **Recommended**: upgrade to the latest ComfyUI
2. **Fallback**: pin the frontend version via the launcher flag:
   ```bash
   python main.py --front-end-version Comfy-Org/ComfyUI_frontend@1.38.0
   ```

The plugin prints the detected frontend version to the console at load time, which helps when reporting issues.
```

- [ ] **Step 2: Verify the edit**

Run: `head -30 README_EN.md`

Expected: The "## Version Downloads" header is gone. The "## System Requirements" header appears between the license badge line and the "An all-in-one ComfyUI node plugin..." paragraph.

- [ ] **Step 3: Commit**

```bash
git add README_EN.md
git commit -m "docs: replace Version Downloads with System Requirements in README_EN.md"
```

---

### Task 4: Manual verification with ComfyUI running

**Files:** None modified. This is a verification gate before merging to `main`.

This task **must** be done by the user (Alfred). The agent cannot launch ComfyUI in this environment.

- [ ] **Step 1: Start ComfyUI normally**

User runs their usual ComfyUI startup command.

Expected console output: includes a line like `[GaussianViewer] ComfyUI frontend version: 1.46.x` (the exact version depends on the user's environment). If the line is **missing**, check whether `comfyui_frontend_package` is installed (`pip show comfyui_frontend_package`). If the package is missing, the silent ImportError is correct behavior.

- [ ] **Step 2: Verify the GaussianViewer node loads and renders correctly**

In ComfyUI:

1. Add a `GaussianViewer` node — confirm the node appears
2. Confirm the node frame is **not** undersized — the iframe viewer should fill the bottom of the node, no content overflow
3. Connect a valid PLY path
4. Adjust the camera, click `Set Camera`
5. Run the workflow — confirm a rendered image appears at the IMAGE output

Expected: all five steps succeed.

- [ ] **Step 3: Verify error path**

Connect a non-existent PLY path. Run the workflow.

Expected: the node returns a placeholder image quickly (within a few seconds, not after the full 60-second timeout). Console shows `[RenderGaussian] ===== RENDER FAILED =====`.

- [ ] **Step 4: Verify README rendering on GitHub** (optional, can be deferred until after push)

Push the branch (Task 6 covers this). Open the branch on GitHub.

Expected: README shows "## 系统要求" / "## System Requirements" — no broken markdown, no leftover "版本下载" / "Version Downloads" text.

- [ ] **Step 5: Sign off**

If all steps above passed, mark this task complete and proceed to Task 5. If any step failed, stop the plan and report the failure — do not proceed to merge.

---

### Task 5: Merge `pr-11-test` into `main`

**Files:** None. This is a git-only operation.

**Prerequisite:** Task 4 fully passed (user explicitly confirmed).

- [ ] **Step 1: Confirm working tree is clean**

Run: `git status`

Expected: `nothing to commit, working tree clean`. If dirty, stop and resolve before continuing.

- [ ] **Step 2: Confirm `pr-11-test` is ahead of `main`**

Run: `git log --oneline main..pr-11-test`

Expected: shows the spec commits and the implementation commits from this plan (Tasks 1–3). At minimum: 1 spec commit, 1 spec revision commit, 3 implementation commits = 5 commits.

- [ ] **Step 3: Switch to main and fast-forward merge**

```bash
git checkout main
git merge --ff-only pr-11-test
```

Expected: `Fast-forward` merge succeeds. If git refuses (`Not possible to fast-forward, aborting.`), `main` has diverged — stop and ask the user how to proceed (likely: rebase `pr-11-test` onto `main` first).

- [ ] **Step 4: Push `main`**

Pushing changes the public repo. **Confirm with the user before running this step.**

```bash
git push origin main
```

Expected: push succeeds.

- [ ] **Step 5: Delete the local `pr-11-test` branch**

```bash
git branch -d pr-11-test
```

Expected: branch deleted. The `-d` (lowercase) form refuses to delete unmerged branches as a safety check; since we just fast-forward-merged it, this should succeed.

If you also want to delete the remote `pr-11-test`, **confirm with the user first**, then:

```bash
git push origin --delete pr-11-test
```

---

### Task 6: Delete the redundant `new-comfyui` branch

**Files:** None. This is a git-only operation.

**Prerequisite:** Task 5 succeeded — `main` is up to date and pushed.

- [ ] **Step 1: Confirm `new-comfyui` has no commits beyond `main`**

Run: `git log --oneline main..new-comfyui`

Expected: empty output (no commits unique to `new-comfyui`).

If there **are** commits unique to `new-comfyui`, stop and report — the spec assumed identity with `main` and the situation has changed.

- [ ] **Step 2: Confirm with the user before deleting**

Both local and remote deletion are destructive. Ask:

> "About to delete `new-comfyui` (local + remote). It is identical to `main`. Confirm?"

If the user says no, stop here and leave the branch in place.

- [ ] **Step 3: Delete local `new-comfyui`**

```bash
git branch -d new-comfyui
```

- [ ] **Step 4: Delete remote `new-comfyui`**

```bash
git push origin --delete new-comfyui
```

Expected: both deletions succeed.

---

### Task 7: User review of `pr-9-test` and `legacy-comfyui` (advisory)

**Files:** None.

This task is **not** an action — it is a checkpoint where the agent presents information and waits for the user's decision. Two leftover branches need a call from the user:

- [ ] **Step 1: Show the user a summary of `pr-9-test`**

Run:

```bash
git log --oneline main..pr-9-test
git diff --stat main pr-9-test
```

Show the output to the user. Ask:

> "`pr-9-test` has 6 commits not in `main`, and the diff shows it is ~1300 lines smaller than `main` (likely a much older base). Three options:
> 1. Cherry-pick valuable commits into `main` (which ones?)
> 2. Delete the branch (local + remote)
> 3. Keep it as-is for now
>
> Which?"

Do **not** delete or modify anything until the user answers.

- [ ] **Step 2: Confirm `legacy-comfyui` should remain as archive**

Show the user:

```bash
git log --oneline main..legacy-comfyui
git log --oneline legacy-comfyui..main
```

Ask:

> "`legacy-comfyui` is preserved as an archive per the spec. Confirm we leave it untouched? (Optional: add a one-line README banner saying 'archived, use main' — say yes if you want me to do that.)"

- [ ] **Step 3: Apply user decisions**

Whatever the user chose, apply it now. If the user declined further action, mark the task complete and end the plan.

---

## Summary of all commits this plan creates (on `pr-11-test` before merge)

1. `feat: log ComfyUI frontend version at plugin load` (Task 1)
2. `docs: replace 版本下载 with 系统要求 in README.md` (Task 2)
3. `docs: replace Version Downloads with System Requirements in README_EN.md` (Task 3)

After Task 5, these are fast-forwarded into `main`.

---

## Out of scope (handled elsewhere or explicitly deferred)

- Source code changes to `gaussian_viewer.py`, `render_gaussian.py`, or any JS file — `main` already has the new implementations.
- Adding an automated test framework — the plugin has none and adding one is not the goal of this work.
- Changes to the `legacy-comfyui` branch — it is preserved as an archive.
- Future ComfyUI frontend break events — handled by repeating the research-design-merge cycle when they happen.
