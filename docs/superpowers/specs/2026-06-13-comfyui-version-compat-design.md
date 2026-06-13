# 设计文档：ComfyUI 版本兼容统一

**日期**：2026-06-13
**作者**：Alfred Lee（与 Claude 协作）
**状态**：草案，待用户审阅

## 背景

`comfyui_GaussianViewer` 插件目前在 GitHub 上以两个独立分支分发：

- `legacy-comfyui`：面向旧版 ComfyUI 的实现
- `new-comfyui`：面向新版 ComfyUI 的实现（同时也是当前 `main` 的内容）

用户在下载时无法判断自己应该选哪一个分支，因为：

1. 作者无法确定 `legacy-comfyui` 兼容到哪个 ComfyUI 版本为止
2. 作者无法确定 `new-comfyui` 向下兼容到哪个版本

实际症状：在较新的 ComfyUI 上使用旧版分支时，节点的容器框被锁定在很小的尺寸，但内部 DOM 内容按"实际所需尺寸"渲染，导致可视区域被裁切（节点显示一半）。这一症状与 ComfyUI Vue Nodes 渲染器的演进相关，并非简单的"前后端 API 替换"。

## 研究结论

通过对 ComfyUI 上游变更的调研，确认以下事实：

1. **ComfyUI 拆分前端**（独立的 `Comfy-Org/ComfyUI_frontend` 仓库）发生于 **2024-08-21（ComfyUI v0.1.0）**。
2. 本插件首次开发于 **2026-01**，远晚于前端拆分，因此两个分支均针对独立前端体系编写，"旧版"与"新版"的差异并非"内置前端 vs 独立前端"。
3. 新版分支使用的 `addDOMWidget` options-bag API（`getMinHeight` / `getHeight` / `onResize` / `afterResize`）在 **frontend v1.38.0 即已存在**，且签名稳定至 v1.46。
4. 真正引发"节点显示一半"症状的，是 frontend v1.38 → v1.46 之间 Vue Nodes 渲染器的渐进硬化：当 widget 上定义了 `widget.computeSize()` 时，新渲染器会把 widget 当作"固定大小的 legacy widget"处理，从而锁死容器尺寸。
5. ComfyUI 自带 `--front-end-version Comfy-Org/ComfyUI_frontend@<version>` 命令行参数，用户可手动锁定前端版本作为兜底方案。

**结论**：新版分支的代码对 frontend v1.38+ 全部兼容，不需要运行时分支。两个分支的合并是低风险动作。研究无法精确给出"v1.38 是 break point"的下界证明，仅能确认"v1.38 时新 API 已经存在"，因此本 spec 不在代码中硬编码版本下界。

## 目标

- 把代码合并为单一主干 `main`，对应当前 `new-comfyui` 的内容
- 用户只需下载一份代码
- 在 ComfyUI 加载插件时打印当前前端版本号，作为信息提示，便于用户/作者排查问题
- README 清晰说明系统要求和兜底方案

## 非目标

- 不在代码中硬编码最低支持版本。研究无法精确给出 break point，硬编码版本号反而可能误伤可正常工作的环境
- 不阻断插件加载。任何环境下插件都尝试加载
- 不引入运行时 feature detection（探测 widget 是否支持 options-bag）。研究已确认无必要
- 不修改 `legacy-comfyui` 分支的代码内容（仅作为存档保留）
- 不引入新的测试框架（本插件目前没有自动化测试体系）

## 改动范围

### 1. 分支策略

通过对当前仓库分支状态的核查（commit graph + diff），现状如下：

| 分支 | 当前 commit | 与 `main` 的关系 |
|---|---|---|
| `main` | `5da86e7` | 主干 |
| `new-comfyui` | `5da86e7` | 与 `main` 完全相同（冗余） |
| `legacy-comfyui` | `bccf9a6` | 落后 main 3 个 commit（缺 PR #11 渲染修复 + "双分支下载"README） |
| `pr-11-test` | `28157f5`（含本 spec） | main + 本设计文档 |
| `pr-9-test` | `8ff7014` | 历史孤儿分支，6 个 commit 未合入任何分支，基于很老的代码（diff 显示比 main 少约 1300 行） |

**处理方案**：

- **保留 `main`**：作为唯一开发分支
- **保留 `legacy-comfyui`** 作为存档：原状不动。它代表 PR #11 之前的版本，是有意义的历史快照；少数仍卡在很老 ComfyUI 的用户可作为最后兜底
- **删除 `new-comfyui`**（本地 + 远程）：与 `main` 完全相同，纯冗余。删除前需先确认 README 已不再指向它的下载链接（本 spec 同时改 README）
- **`pr-9-test` 由用户审阅后决定**：6 个孤儿 commit 看 commit message 涉及"实时高斯缩放控制"、"PLY 加载错误修复"等，可能是有价值但未合并的工作，也可能是已经被其它方式合入的旧版本。本 spec 不替用户做决定，仅在执行 plan 中提醒用户审阅
- **`pr-11-test`**：作为本次工作的工作分支。所有改动 commit 在该分支上，最终通过 fast-forward 或 PR 方式合并到 `main`，合并后删除该本地分支

**所有删除分支操作（包括远程）必须在执行 plan 中显式确认后才执行，本 spec 仅做规划。**

### 2. `__init__.py`：版本探测与日志

在文件顶部、子模块 import **之前**添加 `_log_frontend_version()` 函数并调用：

```python
def _log_frontend_version():
    """在加载时打印当前 ComfyUI frontend 版本，仅作信息提示。

    Why: 本插件依赖 frontend 的 addDOMWidget options-bag API（getMinHeight /
    getHeight / onResize / afterResize），该 API 至少从 frontend v1.38 起稳定
    存在。研究无法精确给出 break point，因此不在代码里硬编码下界，仅打印
    版本供用户和作者在排查问题时参考。
    """
    try:
        import comfyui_frontend_package
        version = getattr(comfyui_frontend_package, "__version__", None)
        if version:
            print(f"[GaussianViewer] ComfyUI frontend version: {version}")
    except ImportError:
        # 极老的 ComfyUI 没有 comfyui_frontend_package 包；静默忽略
        pass


_log_frontend_version()
```

**关键决策**：

- **不做版本判断**。仅打印当前版本，让信息暴露在控制台，便于用户提 issue 时附带，也便于作者排查
- 数据源：`comfyui_frontend_package.__version__`（2025-03 ComfyUI 改用 PyPI 包后的标准来源）
- 失败模式：`ImportError` → 静默。极老的 ComfyUI 不存在该包，对其打印是噪音
- 不 `raise`，不阻断插件加载

### 3. `gaussian_viewer.py` / `render_gaussian.py` / `web/js/*.js`

**不需要改动**。当前 `main` 分支的内容已经是新版实现，包含：

- `INPUT_TYPES` 中的 `"hidden": {"node_id": "UNIQUE_ID"}`
- `get_comfy_output_file_info()` 计算 `subfolder` / `type` / `relative_path`
- `_store_render_error()` + `/geompack/render_error` HTTP 端点
- `_wait_for_render_result()` 同时检查错误队列
- JS 端 `getNodeClass()` / `makeViewUrl()` / `postJson()` helpers
- `addDOMWidget` 使用 options-bag（`getMinHeight` / `getHeight` / `onResize` / `afterResize`）

### 4. `README.md` 改动

**删除**："## 版本下载" 整段（含两个分支下载链接和"如果你的 ComfyUI 还没有更新..."的说明）

**新增**：在文件靠前位置插入 "## 系统要求" 段：

```markdown
## 系统要求

- ComfyUI（建议使用较新的版本）
- Python 包：numpy / torch / Pillow（通常随 ComfyUI 安装）

如果你的 ComfyUI 比较老，节点显示出现裁切（节点框比内容小、内容溢出），
通常是 ComfyUI 前端版本过旧。两种解决方案：

1. **推荐**：升级到最新 ComfyUI
2. **兜底**：用 `--front-end-version` 启动参数锁定前端版本：
   ```bash
   python main.py --front-end-version Comfy-Org/ComfyUI_frontend@1.38.0
   ```

启动 ComfyUI 后，本插件会在控制台打印当前的 frontend 版本，便于排查问题。
```

### 5. `README_EN.md` 改动

同步英文版：

**删除**："## Version Downloads" 整段

**新增** "## System Requirements" 段：

```markdown
## System Requirements

- ComfyUI (a recent version is recommended)
- Python packages: numpy / torch / Pillow (usually shipped with ComfyUI)

If your ComfyUI is older and the node renders truncated (the node frame is
smaller than its content, and the content overflows), the ComfyUI frontend is
likely too old. Two ways to fix:

1. **Recommended**: upgrade to the latest ComfyUI
2. **Fallback**: pin the frontend version via the launcher flag:
   ```bash
   python main.py --front-end-version Comfy-Org/ComfyUI_frontend@1.38.0
   ```

The plugin prints the detected frontend version to the console at load time,
which helps when reporting issues.
```

## 影响面

| 文件 | 改动类型 |
|---|---|
| `__init__.py` | 新增 ~15 行（`_log_frontend_version` + 调用） |
| `README.md` | 删除"版本下载"段，新增"系统要求"段 |
| `README_EN.md` | 同上（英文同步） |
| 其它源代码文件 | 无改动 |

## 验收测试（人工执行）

1. 在当前 ComfyUI 环境下加载插件，确认控制台打印 `[GaussianViewer] ComfyUI frontend version: <版本号>`
2. `GaussianViewer` 节点出现，节点框尺寸正常、不被裁切
3. 加载一个 PLY 文件，调整视角，点 `Set Camera`，运行渲染
4. 渲染成功，输出图像正常
5. 故意制造一个错误（不存在的 PLY 路径），确认错误能立即返回，不再等到 timeout
6. README 显示正确，"版本下载"段已删除，"系统要求"段存在
7. **可选**：把当前的 `comfyui_frontend_package` 临时卸载，重启 ComfyUI，确认插件仍能加载（控制台不打印 frontend 版本，但不报错）

## 风险

- **风险 1**：`comfyui_frontend_package` 在某些极端环境不存在或没有 `__version__` 属性
  **缓解**：仅捕获 `ImportError` 并静默退出；版本号通过 `getattr(..., None)` 取，缺失时不打印不报错
- **风险 2**：未来 ComfyUI 进一步演进破坏当前 options-bag API
  **缓解**：本 spec 不解决未来问题。届时再走相同的"研究 → 设计 → 合并"流程。届时控制台打印的版本号能帮助快速定位
- **风险 3**：删除 `new-comfyui` 远程分支后，如果某些用户已收藏该 zip 链接，会出现 404
  **缓解**：在删除前，确保 README 已更新且 `main` 已发布。404 不会损坏用户已下载的代码

## 后续工作（不在本 spec 范围）

- `legacy-comfyui` 分支的 README 加"已归档"说明（可选，由用户决定是否做）
- `pr-9-test` 的处理（用户审阅后决定保留或删除）
- 监控 ComfyUI frontend 后续版本，发现新破坏时按相同流程响应
