# AfterRec documentation

[简体中文](README.zh-CN.md) · [Architecture](architecture/OVERVIEW.md) · [Main README](../README.md)

Read the [architecture overview](architecture/OVERVIEW.md) first. The documents below are organized by **system responsibility**. Some modules still have placeholder design pages.

## Before implementation: design and interface templates

Use the [module design template](templates/DESIGN_TEMPLATE.md) ([中文](templates/DESIGN_TEMPLATE.zh-CN.md)) to clarify this iteration's scope, workflow, dependencies and observable deliverables before implementation. Keep it short; omit sections that do not help the decision.

When modules or frontend/backend need a new contract, use the [interface template](templates/INTERFACE_TEMPLATE.md) ([中文](templates/INTERFACE_TEMPLATE.zh-CN.md)) to agree on the call and essential input/output. Reuse existing [shared contracts](contracts/INTERFACES.md); document lifecycle and errors only when needed by consumers.

## Module design

| Module | Design document | What to document |
| --- | --- | --- |
| Data | [datasets](modules/datasets/DESIGN.md) | Supported inputs, preprocessing, dataset splits, forget/retain manifests and versioning |
| Recommendation | [recommendation](modules/recommendation/DESIGN.md) | Model lifecycle, training configuration, scoring, Top-K, checkpoints and adapters |
| Unlearning | [unlearning](modules/unlearning/DESIGN.md) | Forget requests, algorithm integration, Retrain baseline and model artifacts |
| Static evaluation | [evaluation](modules/evaluation/DESIGN.md) | Metrics, candidate/masking protocol, baselines, reporting and reproducibility |
| Agent simulation | [Agent shopping simulation and trace replay](modules/agent-simulation/DESIGN.md) | Ranked products, agent shopping actions, session state and human replay |
| Experiment platform | [platform](modules/platform/DESIGN.md) | Backend orchestration, Web workflows, experiment status and local artifacts |

## Interfaces

The existing [shared contracts](contracts/INTERFACES.md) remain the reference for current typed objects and module signatures. Fill in the following pages as interfaces are implemented:

| Interface | Define |
| --- | --- |
| [Frontend ↔ backend](interfaces/FRONTEND_BACKEND.md) | HTTP routes, payloads, errors, job status and frontend integration |
| [Data ↔ recommendation](interfaces/DATA_RECOMMENDATION.md) | Dataset refs, ID mapping, metadata and training inputs |
| [Recommendation ↔ unlearning](interfaces/RECOMMENDATION_UNLEARNING.md) | Original checkpoints, forget requests, method compatibility and outputs |
| [Model ↔ evaluation](interfaces/MODEL_EVALUATION.md) | Ranking candidates, metric inputs, protocols and result schemas |
| [Recommendation ↔ agent](interfaces/RECOMMENDATION_AGENT.md) | Ranked products, store initialization, stepwise interaction and trace delivery |
| [Experiment artifacts](interfaces/EXPERIMENT_ARTIFACTS.md) | Run IDs, file layout, model/run lineage, provenance and serialization |

## Research and progress

- [Simulation research protocol](research/SIMULATION_PROTOCOL.md)
- [Roadmap](ROADMAP.md)
