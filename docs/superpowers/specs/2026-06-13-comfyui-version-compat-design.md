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

**结论**：新版分支的代码对 frontend v1.38+ 全部兼容，不需要运行时分支。两个分支的合并是低风险动作。

## 目标

- 把代码合并为单一主干 `main`，对应当前 `new-comfyui` 的内容
- 用户只需下载一份代码
- 在 ComfyUI 加载插件时探测前端版本，对低于 v1.38 的环境给出**友好警告**（不阻断加载）
- README 清晰说明系统要求和兜底方案

## 非目标

- 不再向下兼容 frontend < v1.38。低于此版本的用户被引导升级 ComfyUI 或使用 `--front-end-version` 锁版
- 不引入运行时 feature detection（探测 widget 是否支持 options-bag）。研究已确认无必要
- 不修改 `legacy-comfyui` 分支的代码内容（仅作为存档保留）
- 不引入新的测试框架（本插件目前没有自动化测试体系）

## 版本边界

| 项 | 值 |
|---|---|
| 最低支持 ComfyUI frontend | **v1.38.0**（约 2026-01-13） |
| 对应 ComfyUI 主仓库版本 | v0.9.0+ |
| 警告阈值 | frontend major.minor < (1, 38) |

## 改动范围

### 1. 分支策略

- `main`：保持当前状态（已是新版代码 commit `5da86e7`），作为唯一开发分支
- `new-comfyui`：合并完成后作为历史 tag 保留或在远端删除（由用户决定，本 spec 不强制）
- `legacy-comfyui`：作为存档保留。可选地在该分支添加一个仅修改 README 的 commit，注明"已归档，请使用 main"
- `pr-9-test` / `pr-11-test`：清理已合并的本地分支（由用户决定）

### 2. `__init__.py`：版本探测与警告

在文件顶部、子模块 import **之前**添加 `_check_frontend_version()` 函数并调用：

```python
def _check_frontend_version():
    """检查 ComfyUI frontend 版本，过低时打印友好警告。

    Why: frontend < v1.38 的 Vue Nodes 渲染器会把定义了 computeSize 的 widget
    锁定为固定尺寸，导致本插件的 DOM 内容溢出被裁切。本插件已采用 options-bag
    API，对 v1.38+ 兼容。
    """
    MIN_MAJOR, MIN_MINOR = 1, 38
    try:
        import comfyui_frontend_package
        version = getattr(comfyui_frontend_package, "__version__", None)
        if not version:
            return
        parts = version.split(".")
        major = int(parts[0])
        minor = int(parts[1]) if len(parts) > 1 else 0
        if (major, minor) < (MIN_MAJOR, MIN_MINOR):
            print(
                f"[GaussianViewer] Warning: detected ComfyUI frontend v{version}, "
                f"which is older than the supported minimum v{MIN_MAJOR}.{MIN_MINOR}.0. "
                "The viewer node may render with an undersized frame. "
                "Please upgrade ComfyUI, or pin the frontend with: "
                f"--front-end-version Comfy-Org/ComfyUI_frontend@{MIN_MAJOR}.{MIN_MINOR}.0"
            )
    except (ImportError, ValueError, IndexError):
        # 静默忽略：极老的 ComfyUI 没有 comfyui_frontend_package 包；
        # 解析失败时也不应阻塞插件加载
        pass


_check_frontend_version()
```

**关键决策**：

- 数据源：`comfyui_frontend_package.__version__`（2025-03 ComfyUI 改用 PyPI 包后的标准来源）
- 失败模式：`ImportError` / 解析失败 → 静默，不警告。原因：极老的 ComfyUI 不存在该包，对其报错是噪音
- 输出：仅 `print` 到控制台，不 `raise`，不阻断插件加载
- 警告内容必须包含：检测到的实际版本、最低要求版本、症状描述、两条解决路径

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

- ComfyUI 自 2026-01 之后的版本（前端 ≥ v1.38.0）
- Python 包：numpy / torch / Pillow（通常随 ComfyUI 安装）

如果你的 ComfyUI 比较老，前端版本低于 v1.38，节点显示可能会被裁切。
两种解决方案：

1. **推荐**：升级到最新 ComfyUI
2. **兜底**：用 `--front-end-version` 启动参数锁定前端版本：
   ```bash
   python main.py --front-end-version Comfy-Org/ComfyUI_frontend@1.38.0
   ```
```

### 5. `README_EN.md` 改动

同步英文版：

**删除**："## Version Downloads" 整段

**新增** "## System Requirements" 段：

```markdown
## System Requirements

- ComfyUI from 2026-01 or later (frontend >= v1.38.0)
- Python packages: numpy / torch / Pillow (usually shipped with ComfyUI)

If your ComfyUI is older and the frontend version is below v1.38, the node may
render with an undersized frame. Two ways to fix:

1. **Recommended**: upgrade to the latest ComfyUI
2. **Fallback**: pin the frontend version via the launcher flag:
   ```bash
   python main.py --front-end-version Comfy-Org/ComfyUI_frontend@1.38.0
   ```
```

## 影响面

| 文件 | 改动类型 |
|---|---|
| `__init__.py` | 新增 ~25 行（`_check_frontend_version` + 调用） |
| `README.md` | 删除"版本下载"段，新增"系统要求"段 |
| `README_EN.md` | 同上（英文同步） |
| 其它源代码文件 | 无改动 |

## 验收测试（人工执行）

1. 当前 ComfyUI 环境（应为新版）下加载插件，确认控制台**没有**警告
2. `GaussianViewer` 节点出现，节点框尺寸正常、不被裁切
3. 加载一个 PLY 文件，调整视角，点 `Set Camera`，运行渲染
4. 渲染成功，输出图像正常
5. 故意制造一个错误（不存在的 PLY 路径），确认错误能立即返回，不再等到 timeout
6. README 显示正确，"版本下载"段已删除，"系统要求"段存在
7. **可选**：临时把 `MIN_MINOR` 改成 99，重启 ComfyUI，确认警告输出符合预期；测试完恢复为 38

## 风险

- **风险 1**：`comfyui_frontend_package.__version__` 在某些极端环境不存在或格式异常
  **缓解**：try/except 已覆盖 ImportError / ValueError / IndexError，静默退出
- **风险 2**：v1.38 之前的少量真实用户被"抛弃"
  **缓解**：警告中明确给出 `--front-end-version` 兜底方案，用户仍有可执行的退路
- **风险 3**：未来 ComfyUI 进一步演进破坏当前 options-bag API
  **缓解**：本 spec 不解决未来问题。届时再走相同的"研究 → 设计 → 合并"流程

## 后续工作（不在本 spec 范围）

- `legacy-comfyui` 分支的 README 加"已归档"说明（可选，由用户决定是否做）
- 监控 ComfyUI frontend 后续版本，发现新破坏时按相同流程响应
