---
name: autolife-solidwork-skill
description: "AutoLife SolidWorks 自动化技能，强化机器人零部件建模前复杂度评分、前台建模、旋转切除、选边倒角和轴套类零件流程；用于 SolidWorks 零件、装配、工程图、导出和 CAD 自动化排障。每次执行零件或装配体建模/修改前，先按评分表评估模型难度并等待用户确认，再开始制作模型。"
metadata: { "short-description": "AutoLife optimized SW CAD workflow", "openclaw": { "homepage": "https://github.com/AutoLifeRobot/solidworks-automation-skill", "os": ["win32"], "requires": { "anyBins": ["python", "py"] } } }
---

# AutoLife SolidWorks 自动化技能

## 快速开始

### 环境要求

- Windows 系统 + SolidWorks 已安装并运行
- Python 3.8+ + `pywin32` / `comtypes`
- 如果通过 OpenClaw 使用，确保技能目录位于 `~/.openclaw/skills/autolife-solidwork-skill/` 或 `~/.agents/skills/autolife-solidwork-skill/`

### 入口自检

所有代理在执行 SolidWorks 自动化前，先运行技能自检：

```bash
python SKILL_DIR/scripts/sw_preflight.py
```

规则：

1. 检测到缺少 `comtypes` / `win32com` / `pythoncom` 时，向用户弹出友好确认：
   `检测到当前 Python 环境缺少 comtypes / win32com 库，是否授权 AI 自动为您配置本地环境？[Y/N]`
2. 用户输入 `Y` / `yes` 后，代理可在本地 shell 中自动执行 `python -m pip install "pywin32>=305" "comtypes>=1.2.0"` 补齐依赖；用户拒绝时停止并给出手动安装命令。
3. 检测不到 SolidWorks 安装或 COM 注册时，直接停止，不要继续生成或执行 CAD 脚本；提示用户：需要先手动安装 SolidWorks，并至少启动一次完成 COM 注册。

### 连接 SolidWorks

```python
import sys; sys.path.insert(0, r"SKILL_DIR/scripts")
from sw_connect import mm
from sw_part import sketch, sketch_circle, extrude_boss
from sw_session import SolidWorksSession

session = SolidWorksSession()
model = session.new_part()

with sketch(model, "Front Plane") as sketch_name:
    sketch_circle(model, 0, 0, mm(25))

extrude_boss(model, sketch_name, mm(50))
session.save(model, r"C:\temp\cylinder.sldprt")
session.export(model, r"C:\temp\cylinder.step")
```

> 将 `SKILL_DIR` 替换为此技能的实际安装路径。

## 核心工作流

根据用户需求选择对应模块：

| 需求 | 脚本 | 参考文档 |
|---|---|---|
| 入口自检与依赖补齐 | `scripts/sw_preflight.py` | `references/troubleshooting.md` |
| 机器人零部件建模前复杂度评分 | - | `references/robot-part-complexity-scoring.md` |
| 多模型宏生成防护 | `scripts/sw_macro_guard.py` | `references/openclaw.md` |
| 友好会话 API | `scripts/sw_session.py` | - |
| 连接与文档管理 | `scripts/sw_connect.py` | - |
| 外观与材质 | `scripts/sw_appearance.py` | `references/appearance.md` |
| 零件建模（草图+特征） | `scripts/sw_part.py` | `references/part-modeling.md` |
| 装配体操作 | `scripts/sw_assembly.py` | `references/assembly.md` |
| 工程图出图 | `scripts/sw_drawing.py` | `references/drawing.md` |
| 文件导出 | `scripts/sw_export.py` | `references/export.md` |
| 结果自审查 | `scripts/sw_review.py` | `references/review.md` |
| 未封装 API 查证 | - | `references/api-lookup.md` |
| OpenClaw 控制 SolidWorks | - | `references/openclaw.md` |
| 钣金/焊件/仿真/属性 | - | `references/advanced.md` |
| 常见错误排查 | - | `references/troubleshooting.md` |

## OpenClaw 协作方式

1. 先确认 SolidWorks 版本、界面语言、输入文件路径、输出路径，以及目标操作（建模 / 装配 / 出图 / 导出）。
2. 优先复用 `{baseDir}/scripts` 下已有模块，不要重复手写 COM 连接逻辑。
3. 在 OpenClaw 的 `exec` / `shell` 能力中执行短小、一次性的 Python 脚本，最小导入集如下：

```python
import sys
sys.path.insert(0, r"{baseDir}/scripts")
from sw_connect import connect_solidworks, mm, deg, new_document
```

4. 执行后检查返回对象是否为 `None`、保存/导出是否成功、输出文件是否落盘。
5. 生成或修改模型后必须做结果自审查：导出至少一个等轴测预览图，必要时导出前/俯/右视图，并通过截图或 BMP 目视检查几何是否符合用户意图。
6. 如果需要更完整的 OpenClaw 工作流、提示词示例和排障建议，再读取 `references/openclaw.md`。

## 建模前评分门禁

每次使用本技能处理零件或装配体的建模、修改、重建、导入后再建模、工程图出图前的模型补全，或任何会改变 CAD 几何的任务时，必须先执行复杂度评分门禁。不得在用户确认前运行 SolidWorks 自检、启动/连接 SolidWorks、生成建模脚本、修改模型、保存或导出模型。

**硬性规则**：必须等待用户确认继续；未收到明确确认前，只能评分、解释评分依据、补充询问输入信息，不能开始制作模型。

1. 先读取 `references/robot-part-complexity-scoring.md`。
2. 根据用户提供的文字、图片、草图、旧模型、尺寸、装配约束和用途，对目标模型按 8 个维度评分。
3. 向用户汇报总分、等级、逐项得分、主要理由、Codex 可承担内容、需要人工确认内容、工程风险标签，以及是否触发一票否决。
4. 明确询问用户是否确认按该等级继续建模。
5. 只有用户明确确认继续后，才进入 SolidWorks 自检和正式建模流程。

例外：如果用户只要求评分、解释、评审、排障、更新 skill 或整理文档，不需要等待建模确认；但只要下一步要制作或修改模型，就必须重新执行评分门禁。

## 使用流程

1. 对所有模型制作或修改任务，先完成“建模前评分门禁”，并等待用户确认继续。
2. 再运行 `sw_preflight.py`：缺依赖则请求用户授权自动安装；缺 SolidWorks 则停止并提示手动安装。
3. 优先用 `SolidWorksSession()` 管理连接、打开、新建、保存、导出。
4. 需要底层控制时再组合 `sw_connect.py`、`sw_part.py` 等函数。
5. 如果必须由大模型生成 VBA 宏，先使用 `sw_macro_guard.py` 做模型分流、代码校验、重试和本地模板兜底。
6. 使用 `session.export()` 或 `sw_export.py` 保存/导出文件。
7. 使用 `sw_review.py` 导出预览图并自审查；如果有 GUI/桌面截图能力，打开 SolidWorks 视图截图复核。

## 实战稳定性规则

以下规则来自真实建模排障，遇到轴套、法兰、回转件、贯穿槽、内孔环槽等任务时优先遵循：

1. **用户要求看绘制过程时，先保证前台 GUI 可见**：`SolidWorksSession(visible=True)` 不一定能得到有窗口句柄的前台实例。必要时先启动真实 `SLDWORKS.exe`，将窗口置前，再连接 `SldWorks.Application`；脚本中设置 `sw.Visible = True`、`sw.UserControl = True`，并在关键特征后调用 `ForceRebuild3(False)`、`ViewZoomtofit2()`、`GraphicsRedraw2()`，短暂停顿，让用户能看到外形、孔、槽、倒角逐步出现。
2. **轴套/法兰类零件优先采用稳定建模顺序**：先用旋转特征生成外部实心轮廓；再切法兰孔、端面异形贯穿槽等非轴对称特征；最后用旋转切除生成中心孔、内孔台阶、内侧环槽。不要过早切掉中心孔，否则后续与内孔相交的花瓣槽、R 槽或局部贯穿槽更容易失败。
3. **旋转切除优先使用专用接口**：旋转实体可用 `FeatureRevolve2`，但旋转切除不要只依赖 `FeatureRevolve2(..., IsCut=True, ...)`。在部分 SolidWorks 版本或 pywin32 COM 环境中该写法可能不返回特征；应优先尝试 `FeatureRevolveCut2(...)`，并检查返回值是否为 `None`。若方法名或参数不确定，先用最小脚本探测当前 `FeatureManager` 是否暴露该成员，再执行正式建模。
4. **用包围盒确认真实轴向和方位**：旋转草图所在平面与模型全局轴向可能不符合直觉。端面切孔、偏置面、贯穿方向失败时，先调用 `model.GetPartBox(True)` 确认实体实际范围和轴向，再决定从哪个平面或端面建草图。
5. **选面选边要选真实几何，不只依赖默认基准面名称**：中英文基准面兜底只能解决默认平面名称问题；端面孔、槽口、局部倒角等应优先用 `SelectByRay` 选择真实面/边。射线命中点必须落在材料区域，避开已经切出的孔、槽和空腔；例如法兰端面二次建草图时，选两孔之间的实体区域比选孔中心更稳定。
6. **保存覆盖前关闭同名旧文档**：如果目标 `.SLDPRT` 已在当前 SolidWorks 中打开，`SaveAs` 可能返回错误码 `1`。覆盖保存前先尝试 `sw.CloseDoc(title)` 关闭同名文档，保存后再检查返回值、文件存在性和文件大小。
7. **倒角策略要分层处理**：不要默认一次性选择所有边做 C2 倒角。对回转体外缘、内孔口、内槽边，优先直接画进旋转截面；对规则孔口，可用 `SelectByRay` 精确选边后调用 `InsertFeatureChamfer`；对花瓣槽、相交槽、复杂贯穿槽口，自动倒角可能失败，应降级为局部可建倒角或明确报告未能生成的边。
8. **COM 选择对象要有兼容兜底**：`SelectionManager.CreateSelectData()` 在部分环境中可能找不到成员。批量选边失败时，改用 `SelectByRay`、`edge.Select(True)` 或重新选择草图/特征；每次特征调用后都检查返回对象，失败时不要继续假设特征已经存在。

## GPT / Kimi / Claude 多模型策略

当代理需要让大模型生成 VBA 宏时，必须通过 `scripts/sw_macro_guard.py`：

1. **模型分流**：GPT 系列使用原有简洁提示词；Kimi / Claude / 未知模型自动加载强格式约束 Prompt，强制只输出 VBA 源码。
2. **本地模板兜底**：模型输出失败或解析失败时，不直接报错；按用户关键词（如“立方体”“圆柱”“拉伸”“草图”）选择内置 VBA 模板继续执行。
3. **输出校验**：执行前检查 `SldWorks`、`ModelDoc2`、`Sub`、`End Sub`，通过后才允许交给 SolidWorks；不通过则重试。
4. **超时/重试**：单次模型请求建议 `30s` 超时；解析失败自动重试 `1~2` 次，重试 Prompt 追加更强格式指令。

示例：

```python
from sw_macro_guard import build_prompt, fallback_macro_for_request, validate_vba_macro

prompt = build_prompt("画一个 50mm 圆柱", model_name="claude")
macro = fallback_macro_for_request("画一个 50mm 圆柱")
result = validate_vba_macro(macro)
assert result.ok, result.issues
```

## 未封装 API 规则

当任务需要调用 `scripts/` 中尚未封装的 SolidWorks API 时：

1. 先读取 `references/api-lookup.md`，再查询 SolidWorks 官方 API 文档，或本地 SolidWorks SDK / 参考资料，确认方法签名、参数含义、枚举值、返回值和版本差异。
2. 禁止凭记忆猜接口；尤其是长参数 COM 方法、`VARIANT` / by-ref 参数、枚举值、选择标记和 `SaveAs` 类接口。
3. 写代码时保留最小可运行脚本，并对每一步返回值做 `None` / `False` 检查。
4. 实现后必须真实运行，保存或导出目标文件，并使用 `sw_review.py` 生成预览图与审查报告。
5. 新发现的坑、错误码、兼容写法或稳定封装，要补充到 `references/troubleshooting.md` 或对应模块参考文档；常用逻辑再沉淀进 `scripts/`。

## 结果自审查

每次生成、修改、导入或导出 CAD 后都要做自审查，除非用户明确说不需要：

1. 检查 COM 返回值、特征对象、保存/导出返回值和输出文件大小。
2. 调用 `model.ForceRebuild3(False)`、`model.ViewZoomtofit2()` 刷新模型。
3. 用 `scripts/sw_review.py` 的 `run_review()` 导出 `isometric/front/top/right` 预览图并写入 `*_review_report.json`。
4. 读取报告里的 `evaluation.status`、`evaluation.issues`、`checks` 和预览图；通过截图或导出的 BMP 检查：主体是否存在、比例/方位是否合理、关键部件是否缺失、是否明显重叠或悬空、文件名和输出路径是否正确。
5. 若发现问题，先修脚本并重新生成，再汇报；不要只报告“保存成功”。

示例：

```python
from sw_review import run_review

model.ForceRebuild3(False)
report, report_path = run_review(
    model,
    r"C:\temp\review",
    basename="car",
    expected_outputs=[r"C:\temp\car.sldprt", r"C:\temp\car.step"],
)
print(report_path)
print(report["evaluation"])
```

## 建模后自我迭代

每次完成 SolidWorks 建模、修改、导入、工程图或导出任务后，在完成“结果自审查”之后执行自我迭代记录。所有自我迭代相关文件必须统一写入 `D:\codex_iterative_learning\AutoLife-solidwork-iteration`，不要写入 skill 安装目录或临时工作目录。

1. **固定学习根目录**：使用 `D:\codex_iterative_learning\AutoLife-solidwork-iteration` 作为唯一自我迭代根目录；若目录不存在，先创建 `cases`、`artifacts`、`proposals`、`scripts` 子目录。
2. **每次任务都要留痕**：建模成功、部分成功、失败或用户中断，都要向 `iteration-log.md` 追加一条记录；确实没有新经验时写明“无新增可沉淀经验”。
3. **按任务建案例文件夹**：复杂任务或出现失败重试时，在 `cases\YYYYMMDD-HHMMSS-brief-slug\` 下保存 `summary.md`，并把关键宏、Python 脚本、预览图、错误日志、审查报告复制或记录到该案例目录。
4. **候选规则先隔离**：新的 API 坑、SolidWorks 版本差异、选边/倒角/旋转切除稳定写法，先写入 `candidate-rules.md` 或 `proposals\YYYYMMDD-brief-title.md`，不要直接自动改 `SKILL.md`。
5. **满足条件再沉淀进 skill**：只有同类问题重复出现至少 2 次，或一次问题已被真实运行验证为通用规则，并且用户明确要求优化 skill 时，才把候选规则整理进 `D:\codex_iterative_learning\AutoLife-solidwork-skill\SKILL.md`；修改后必须重新校验并按需重新安装。
6. **记录要短而可复用**：每条复盘优先包含任务类型、图纸/输入、输出文件、SolidWorks 版本、成功路径、失败 API、修复方式、是否建议升级为 skill 规则。不要记录许可证、账号、私密路径以外的敏感信息。

## 关键注意事项

- **单位**：API 统一使用**米**。用 `mm(50)` 转换 50mm 为 0.05m，用 `deg(90)` 转换角度
- **版本**：使用 `SldWorks.Application` 自动连接，兼容所有版本
- **选择**：特征操作前需用 `SelectByID2` 选择目标实体
- **草图**：推荐用 `with sketch(model, "Front Plane") as sketch_name:` 自动进入/退出草图并获取草图名
- **外观**：对颜色要求高的模型优先拆成多零件装配体，并用 `sw_appearance.py` 设置文档级或组件级外观
- **VARIANT**：by-ref 参数必须用 `VARIANT(pythoncom.VT_BYREF | pythoncom.VT_I4, 0)` 包装
- **基准面名称**：`start_sketch()` 会自动兼容英文版 "Front/Top/Right Plane" 与中文版 "前视/上视/右视基准面"
- **草图坐标**：基于草图平面的局部坐标系，单位为米
