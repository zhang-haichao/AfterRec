# AfterRec 项目文档

[English](README.md) · [整体架构](architecture/OVERVIEW.zh-CN.md) · [项目首页](../README.zh-CN.md)

建议先阅读[整体架构](architecture/OVERVIEW.zh-CN.md)。以下文档按**功能模块和接口边界**组织，部分模块仍待补充设计。

## 开发前先统一设计和接口

开发前可参考[模块设计模板](templates/DESIGN_TEMPLATE.zh-CN.md)（[English](templates/DESIGN_TEMPLATE.md)），说明本阶段目标、整体流程、模块依赖及完成后可检查的交付结果。正文以简洁易懂为准，不要求凑字数或补齐与模块无关的章节。

跨模块或前后端需要约定新接口时，使用[接口模板](templates/INTERFACE_TEMPLATE.zh-CN.md)（[English](templates/INTERFACE_TEMPLATE.md)），提前确认调用方式和必要的输入输出。已在[公共协议](contracts/INTERFACES.md)定义的字段直接引用；异常和任务状态仅在影响对接时补充。

## 模块设计

| 模块 | 文档 | 需要补充的内容 |
| --- | --- | --- |
| 数据管理 | [datasets](modules/datasets/DESIGN.md) | 数据格式、预处理、划分、Forget/Retain 清单与数据版本 |
| 推荐模型 | [recommendation](modules/recommendation/DESIGN.md) | 模型训练、加载、打分、Top-K、Checkpoint 与适配方式 |
| 遗忘算法 | [unlearning](modules/unlearning/DESIGN.md) | 遗忘请求、算法接入、Retrain 基线与模型输出 |
| 静态评测 | [evaluation](modules/evaluation/DESIGN.md) | 指标定义、候选集、过滤规则、对照实验与结果记录 |
| Agent 模拟 | [Agent 购物模拟与轨迹回放](modules/agent-simulation/DESIGN.zh-CN.md) | 推荐榜单、Agent 购物动作、会话状态与真人回放 |
| 实验平台 | [platform](modules/platform/DESIGN.md) | 后端任务编排、前端流程、实验状态与产物管理 |

## 接口协议

现有[共享接口协议](contracts/INTERFACES.md)记录当前数据结构与方法签名。模块对接细节在以下文件补充，避免接口定义分散在各模块文档中：

| 接口 | 需要明确的内容 |
| --- | --- |
| [前后端接口](interfaces/FRONTEND_BACKEND.md) | HTTP 路由、请求响应、错误、任务状态、前端调用 |
| [数据与推荐](interfaces/DATA_RECOMMENDATION.md) | 数据引用、ID 映射、商品元数据、模型训练输入 |
| [推荐与遗忘](interfaces/RECOMMENDATION_UNLEARNING.md) | 模型产物、遗忘请求、兼容性和输出格式 |
| [模型与评测](interfaces/MODEL_EVALUATION.md) | 排序候选集、指标输入、评测协议和结果格式 |
| [推荐与 Agent](interfaces/RECOMMENDATION_AGENT.zh-CN.md) | 有序榜单、购物环境、逐步交互与轨迹交付 |
| [实验产物](interfaces/EXPERIMENT_ARTIFACTS.md) | 任务 ID、文件组织、模型关系、实验溯源 |

## 研究与进度

- [Agent 研究与评测协议](research/SIMULATION_PROTOCOL.md)
- [开发路线图](ROADMAP.md)
