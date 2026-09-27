# 意拍 IntentShot｜技术架构设计

**版本：** v1.0  
**状态：** 技术方案 / 待真机验证  
**核心架构：** Apple 本地模型 + Vision + 摄影规则引擎 + AVFoundation；JEV 为可选决策增强服务。

---

## 1. 架构目标与原则

### 1.1 目标
构建一套以自然语言为入口、以真实相机执行为落点的摄影系统，支持意图解析、场景感知、方案生成、参数执行、成片评估与迭代。

### 1.2 原则
1. 本地优先：取景、图像分析和基础摄影决策尽量在设备端完成。
2. 模型与执行分离：模型输出意图或候选策略，不直接调用相机底层接口。
3. 规则兜底：参数边界、设备能力、用户锁定和安全条件由确定性逻辑控制。
4. 实时链路轻量化：不让大语言模型逐帧参与取景处理。
5. 可降级：本地模型或网络不可用时仍能使用基础拍摄能力。
6. 可观测：记录模型/规则版本、决策过程摘要、参数应用结果和用户反馈。
7. 以公开 API 为边界：不假定可控制系统相机的全部计算摄影链路。

---

## 2. 总体架构

```text
┌───────────────────────────────────────────────┐
│                 iOS 用户体验层                 │
│  自然语言输入 / 方案摘要 / 取景器 / 拍摄 / 反馈 │
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│                  会话与任务编排                │
│  CaptureSession / 状态机 / 上下文 / 方案历史    │
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│                  AI 理解与判断层               │
│  Foundation Models：意图解析、局部调整、澄清    │
│  Vision：主体/人脸/场景特征                    │
│  JEV（可选云端）：低频复杂策略判断              │
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│                 摄影策略与规则层               │
│  意图归一化 / 候选策略 / 规则库 / 参数优化       │
│  设备能力适配 / 约束校验 / 用户锁定 / 降级       │
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│                 相机执行与媒体层               │
│  AVFoundation / CaptureDevice / PhotoOutput     │
│  参数应用 / 对焦与曝光 / 拍照 / 录像（后续）     │
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│                 结果评估与反馈层               │
│  图像质量分析 / 目标达成评估 / 用户反馈 / 撤销   │
└───────────────────────┬───────────────────────┘
                        └────── 回到会话编排
```

---

## 3. 模块职责

### 3.1 App Shell & UI
- SwiftUI 构建页面和状态展示。
- 取景器作为核心工作区。
- 管理输入、方案预览、参数状态、拍摄和反馈交互。
- 显示 AI 正在分析、已应用、部分支持、需要用户确认等状态。

### 3.2 CaptureSession / Task Orchestrator
维护一次拍摄任务的上下文与状态。建议状态：

`idle → intent_input → intent_parsed → scene_analyzing → plan_ready → applying → ready_to_capture → capturing → evaluating → adjustment_ready / completed`

异常路径包含 `degraded`、`permission_blocked`、`capture_failed` 和 `cancelled`。

职责：
- 管理会话生命周期。
- 保留用户目标、约束、锁定参数和历史版本。
- 取消过期的模型任务。
- 防止并发方案覆盖当前有效方案。
- 处理应用中断与相机状态变化。

### 3.3 Foundation Models Adapter
职责：
- 自然语言到 CaptureIntent 的解析。
- 局部修改指令解析。
- 需求冲突提示与澄清问题生成。
- 基于已验证数据生成简短方案说明。

约束：
- 输出必须遵循版本化 Schema。
- 不能直接输出未经校验的硬件参数。
- 模型不可用或输出不合规时，回退到预设/规则流程。
- 具体模型能力、上下文限制、系统版本要求与性能必须在目标设备验证。

### 3.4 Vision / Core ML Perception
职责：
- 人脸检测、人物/主体定位。
- 按需进行主体分割、构图特征或场景特征提取。
- 对拍摄结果进行清晰度、曝光等基础分析。
- 将图像转换为结构化 SceneState，而非把连续原始帧交给语言模型。

性能策略：
- 轻量指标低频/连续获取。
- 重型视觉请求按需触发。
- 对结果设置时间戳与有效期，过期状态不可用于新方案。
- 对帧率、温升、电量和内存进行基准测试。

### 3.5 JEV Decision Adapter（可选）
定位：复杂、低频、结构化的策略判断增强模块，不是相机控制器。

适用任务：
- 多个候选策略之间的比较。
- 多目标冲突下的策略选择。
- 复杂失败场景的诊断路径选择。

不适用任务：
- 每帧实时分析。
- 直接计算快门/ISO/曝光补偿等数值。
- 直接控制 AVFoundation。
- 作为基础拍摄的在线依赖。

部署建议：
- 若采用云端 API，iOS 客户端不得持有服务密钥。
- 由 Backend 代理、鉴权、限流、超时、缓存和审计。
- 只传必要的结构化状态；默认不上传原始照片或视频。
- 结果必须经过本地 Schema、候选集和规则校验。
- 超时或失败时本地规则直接接管。

### 3.6 Photography Policy Engine
这是系统核心领域模块，负责将用户目标转化为设备可执行的策略。

子模块：
1. Intent Normalizer：意图字段标准化与默认值补齐。
2. Capability Adapter：动态读取设备支持能力。
3. Scene Policy：根据场景选择候选策略。
4. Constraint Solver：处理硬约束、软目标、冲突与优先级。
5. Parameter Planner：生成参数/策略方案。
6. Validator：检查范围、模式兼容、用户锁定和状态有效性。
7. Stabilizer：阈值、冷却、滞回和防抖。
8. Fallback Policy：无法执行时选择降级路径并生成说明。

### 3.7 Camera Execution Adapter
- 封装 AVFoundation 的相机配置和拍摄调用。
- 将策略对象映射到公开 API 支持的参数。
- 在正确的 session queue 上执行配置。
- 检查设备锁定、模式支持和设置结果。
- 记录 requested state 与 actual state，识别部分应用。
- 对无法控制的系统计算摄影行为明确标注为“不可保证”。

### 3.8 Result Evaluator
- 将 CaptureIntent 与实际照片特征对照。
- 对可量化目标给出 achieved / partial / not-achieved / unknown。
- 对主观风格只给出辅助判断，不将模型评分当作客观事实。
- 生成局部调整候选，不直接写入相机。
- 收集用户满意/不满意反馈。

---

## 4. 核心数据契约

以下为产品内部建议 Schema 的简化示例；应在工程实现中使用 Codable/JSON Schema 或等价类型，并进行版本化。

### 4.1 CaptureIntent

```json
{
  "schema_version": "1.0",
  "subject": {"type": "person", "target": "primary_subject"},
  "style": {"color": "cinematic_warm", "depth": "shallow"},
  "goals": ["face_bright", "subject_sharp", "background_dark"],
  "constraints": ["flash_off"],
  "priority": ["subject_sharp", "face_bright", "background_dark"],
  "clarification_needed": false
}
```

### 4.2 SceneState

```json
{
  "schema_version": "1.0",
  "timestamp": "runtime_timestamp",
  "scene": {"category": "portrait", "lighting": "backlit"},
  "subject": {
    "type": "person",
    "tracking": "stable",
    "normalized_position": [0.52, 0.43],
    "face_detected": true
  },
  "image_metrics": {
    "face_brightness": "low",
    "overall_brightness": "medium",
    "motion_level": "low"
  },
  "camera_state": {
    "active_lens": "runtime_value",
    "focus_state": "runtime_value"
  }
}
```

### 4.3 CameraCapabilities

```json
{
  "schema_version": "1.0",
  "available_modes": [],
  "available_lenses": [],
  "exposure_controls": [],
  "focus_controls": [],
  "white_balance_controls": [],
  "photo_output_capabilities": []
}
```

能力必须由运行时检测获得，不应仅凭机型名称推断。

### 4.4 CapturePlan

```json
{
  "schema_version": "1.0",
  "plan_id": "uuid",
  "strategy": "portrait_face_priority",
  "mode": "runtime_supported_mode",
  "lens_strategy": "runtime_supported_lens",
  "focus_strategy": "primary_face",
  "exposure_strategy": "protect_face",
  "color_strategy": "warm_cinematic",
  "locked_by_user": [],
  "unsupported_goals": [],
  "explanation": "优先保证人物清晰与面部曝光"
}
```

### 4.5 CaptureResult / Evaluation

记录计划版本、实际设备状态、照片分析、目标达成度、用户反馈和错误码；避免保存不必要的敏感图像内容。

---

## 5. 关键流程

### 5.1 生成拍摄方案
1. 接收用户输入。
2. Foundation Models 解析为 CaptureIntent。
3. Schema 校验与需求冲突检查。
4. 获取 CameraCapabilities 与最新 SceneState。
5. Policy Engine 生成候选策略。
6. 简单场景直接走规则；复杂场景可选调用 JEV。
7. 约束求解、参数校验与稳定性检查。
8. 生成 CapturePlan，供 UI 预览。

### 5.2 应用方案
1. 检查方案是否仍然有效（场景/设备状态时间戳）。
2. 检查用户锁定项与能力约束。
3. Camera Adapter 执行配置。
4. 读取实际相机状态并与请求状态对照。
5. 更新 UI 为已应用、部分应用或失败。
6. 失败时回退上一个有效配置或基础模式。

### 5.3 实时取景调整
1. 轻量感知更新 SceneState。
2. 判断场景变化是否超过阈值。
3. 检查是否处于冷却期、用户锁定或拍摄中。
4. Policy Engine 生成有限调整。
5. 校验并应用。
6. 记录变化原因，防止反复震荡。

### 5.4 拍摄后反馈
1. 保存 CaptureIntent、CapturePlan 与实际状态的关联。
2. 分析照片。
3. 评估目标达成度。
4. 展示结果与局部建议。
5. 用户接受后生成新方案；否则保留当前方案。
6. 用户可撤销并保存最终方案。

---

## 6. JEV 集成与实验设计

### 6.1 服务边界
`iOS App → IntentShot Backend → JEV API`

Backend 负责：
- 用户授权与配额。
- 请求验证、去重、超时、限流。
- API Key 安全管理。
- 结果 Schema 校验、日志与成本统计。
- 不存储原始图像，除非用户明确同意且确有必要。

### 6.2 输入原则
只发送已脱敏、必要的结构化特征，例如场景类别、用户目标、候选策略和已知约束。不要把原始视频流作为决策请求输入。

### 6.3 输出原则
JEV 结果只在候选策略范围内被采纳；本地校验器检查结构、候选项、约束与用户优先级。JEV 的概率/置信度不得直接当作实际正确率。

### 6.4 A/B 对照
- A：本地模型 + Vision + 规则引擎。
- B：相同架构 + 仅在复杂策略任务调用 JEV。

比较：策略正确率、参数可执行率、目标达成度、用户满意度、拍摄耗时、延迟、失败率、单位成本。若增益不显著，则不进入默认链路。

---

## 7. iOS 技术选型

| 领域 | 建议技术 | 说明 |
|---|---|---|
| UI | SwiftUI | 页面、状态与交互 |
| 相机 | AVFoundation | 采集、参数控制与拍照 |
| 本地语言模型 | Foundation Models（满足系统/设备条件时） | 意图解析与结构化输出 |
| 视觉 | Vision | 人脸、主体与图像分析 |
| 自定义推理 | Core ML | 后续专用视觉模型 |
| 图像处理 | Core Image / Metal | 图像处理和高性能计算 |
| 本地数据 | SwiftData / SQLite | 方案、偏好、会话历史 |
| 云端 | 轻量 Backend | JEV 代理、远程配置、实验 |
| 观测 | 自建事件/诊断体系 | 性能、错误、实验数据 |

以上为候选技术栈，最终以部署目标的 SDK、系统版本与 API 能力验证为准。

---

## 8. 稳定性与安全机制

### 参数防抖
- 最小变化阈值。
- 更新冷却时间。
- 滞回区间。
- 单次只调整有限维度。
- 用户手动锁定优先级最高。

### 冲突处理
- 硬约束：必须遵守，例如闪光灯关闭、设备支持范围。
- 软目标：清晰度、背景亮度、氛围等，可权衡。
- 不可实现目标：明确说明并给出替代方案。

### 降级层级
1. 本地模型 + Vision + 规则引擎。
2. 预设方案 + Vision + 规则引擎。
3. 纯规则摄影辅助。
4. 基础相机拍摄。

### 安全边界
- 不绕过系统权限。
- 不使用私有 API。
- 不让模型直接生成并执行任意相机调用。
- 云端密钥只存服务端。
- 明确显示无法保证的成像效果。

---

## 9. 性能与质量验证

真机测试至少覆盖：
- 意图解析延迟、Schema 合规率、意图准确率。
- 取景帧率、内存、功耗、温升。
- 相机参数应用成功率、配置耗时和失败恢复。
- Vision 检测准确性及不同光线/主体条件下的表现。
- 模型不可用、权限拒绝、相机占用、网络超时等异常。
- JEV 开关前后的策略增益、延迟和成本。

在获得实测基线前，不设未经验证的性能承诺。

---

## 10. 建议工程模块目录

```text
IntentShot/
├── App/
├── Features/
│   ├── Camera/
│   ├── IntentInput/
│   ├── CapturePlan/
│   ├── ResultReview/
│   ├── Presets/
│   └── Settings/
├── Domain/
│   ├── Models/
│   ├── UseCases/
│   └── Policies/
├── AI/
│   ├── FoundationModelsAdapter/
│   ├── VisionPerception/
│   └── JEVAdapter/
├── CameraEngine/
│   ├── CapabilityAdapter/
│   ├── SessionController/
│   ├── ParameterValidator/
│   └── CaptureExecutor/
├── Persistence/
├── BackendContracts/
└── Tests/
    ├── IntentParsing/
    ├── PolicyEngine/
    ├── CameraCapabilities/
    └── Evaluation/
```

---

## 11. 技术风险清单

| 风险 | 验证/缓解方式 |
|---|---|
| 目标 iPhone 的本地模型实际能力不确定 | 真机检查可用性、延迟、结构化输出 |
| 相机控制 API 不覆盖预期功能 | 建立 capability matrix，按公开 API 做 Spike |
| 视觉算法耗电/发热 | 按需触发、低频分析、测量热状态 |
| 参数震荡 | 阈值、冷却、滞回与锁定机制 |
| 结果评价主观性强 | 可量化指标与用户主观反馈分开 |
| JEV 网络延迟/成本 | 低频可选调用、缓存、降级和 A/B |
| 不同设备效果差异 | 运行时能力适配与分设备测试 |

---

## 12. 技术决策摘要

- **默认主链路：** Foundation Models（可用时）+ Vision + 摄影规则引擎 + AVFoundation。
- **决策核心：** 摄影规则引擎与参数校验器，而非单一模型。
- **JEV 定位：** 可选的云端复杂决策增强，需通过对照实验验证价值。
- **首发场景：** 人像摄影优先。
- **关键里程碑：** 先跑通一个可测量的拍摄闭环，再扩展夜景、静物和视频。
