# 推荐与遗忘接口约定


[团队文档目录](https://github.com/zhang-haichao/AfterRec/blob/main/docs/README.zh-CN.md) · [遗忘模块设计](../modules/unlearning/DESIGN.md)

## 1. 对接场景

- **提供方**为遗忘模块（B）。推荐模块提供 SASRec 的训练、加载和打分能力。
- **使用方**为实验运行器和 B 模块页面。页面通过实验平台的 HTTP 接口发起任务并查询结果。
- **用途**是在原 SASRec 已训练且遗忘请求已确认后，使用 Retain 从头训练，取得可重载的新模型引用。

## 2. 调用方式与输入输出

**Python 调用**

| 调用方向和方法 | 输入 | 输出 |
| --- | --- | --- |
| 运行器 → B · `UnlearningMethod.unlearn(original, request, config)` | 原 `ModelArtifact`、冻结的 `ForgetRequest`、方法配置 | 校验通过的新 `ModelArtifact` |
| B → 推荐模块 · `Recommender.train(dataset, split, config)` | 原版本 `DatasetRef`、基于 Retain 的派生 `SplitRef`、从头训练配置 | 引用新检查点的 `ModelArtifact` |
| B 或评测 → 推荐模块 · `Recommender.load(artifact)` | 指定模型的 `ModelArtifact` | 加载供后续打分，返回 `None` |

请求的 `split_id` 指 Original 的训练切分。派生切分的 Train 指向 `retain_manifest_uri`，Valid、Test 沿用原引用。首轮拟在运行记录中保存实际训练清单和序列样本来源的校验信息。B 复核数据版本、映射与清单，重载和打分通过后将产物交给评测与平台。发现输入不一致时结束本次运行并记录原因。 [共享接口协议](https://github.com/zhang-haichao/AfterRec/blob/main/docs/contracts/INTERFACES.md)

**HTTP 调用（拟议）**

| 方法与路径 | 输入 | 输出 |
| --- | --- | --- |
| `POST /api/v1/unlearning-runs` | 原模型 ID `original_model_id`、已冻结的请求 ID `forget_request_id`、方法 `method`（首轮为 `retrain`） | `202 Accepted`，返回本次任务的 `run_id` 和状态 `status`（`pending`） |
| `GET /api/v1/unlearning-runs/{run_id}` | 提交时返回的 `run_id` | 返回 `run_id` 和 `status`。成功时附新模型的 `model_id`，失败时返回 `error_message`。 |

POST 沿用团队规划的路径。后端按 ID 查找原模型和冻结请求，在后台调用 `unlearn()`。页面用 `run_id` 查询，状态从 `pending` 或 `running` 进入 `succeeded` 或 `failed`。查询路径和请求、响应字段待联调确认。 [前后端接口](https://github.com/zhang-haichao/AfterRec/blob/main/docs/interfaces/FRONTEND_BACKEND.md)

**SASRec 示例。** 页面用 `{"original_model_id":"sasrec-original-01","forget_request_id":"req-01","method":"retrain"}` 提交任务。后端按 `req-01` 找到下面的 `ForgetRequest`，并与原模型一起传给 B。示例中的 ID 和路径均为演示值。

```json
{
  "request_id": "req-01",
  "split_id": "split-01",
  "strategy": "interaction",
  "forget_manifest_uri": "requests/req-01/forget.jsonl",
  "retain_manifest_uri": "requests/req-01/retain.jsonl",
  "seed": 2026
}
```

训练成功后，B 返回以下 `ModelArtifact`。`split-01-retain` 是实际训练的派生切分。示例 `config` 只列运行来源，实际产物还要保留推荐模块重载所需的模型配置。运行记录关联清单核对信息。

```json
{
  "model_id": "sasrec-retrain-01",
  "dataset_id": "clicks-01",
  "split_id": "split-01-retain",
  "model_name": "SASRec",
  "checkpoint_uri": "models/sasrec-retrain-01/checkpoint.pt",
  "id_mapping_uri": "datasets/clicks-01/v1/id_mapping.json",
  "config": {
    "method": "retrain",
    "fresh_initialization": true,
    "source_split_id": "split-01"
  },
  "parent_model_id": "sasrec-original-01",
  "forget_request_id": "req-01"
}
```

