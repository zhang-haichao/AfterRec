# AfterRec Agent Simulation Environment Technical Design

[简体中文](README.zh-CN.md)

Revised draft · November 20, 2026

This document proposes an implementation for the AfterRec simulation environment. The repository contains a minimal simulation runner and action types. The browser prototype and standalone BPR inference were verified outside this repository. The environment interfaces, event formats, and service routes below are proposed for development and integration. Example IDs are fictional.

[Project architecture](../../docs/architecture/OVERVIEW.md) · [Shared simulation protocol](../../docs/research/SIMULATION_PROTOCOL.md)

The first version lets an agent browse products in the order supplied by the recommendation model. Each round first displays one page of products. The agent then decides which products to view, how to rate them, and whether to continue or exit. The environment executes and records these decisions. The interface uses the same records for live display and historical replay. Later versions can support different recommendation models, different agents, and more granular actions.

## 1 Components of the Environment

| Component | Responsibility |
| --- | --- |
| Recommendation integration | Obtains an ordered product ranking for a user from the recommendation module. Imported rankings also support integration testing. Retrained and unlearned models provide recommendations through the same interface. |
| Session execution | The environment maintains the current round, the products on the current page, executed actions, and the termination reason. The runner calls the recommender, agent, and environment in sequence, and handles progress, timeouts, and cancellation. A run can contain multiple sessions, each corresponding to one user and one model arm. |
| Agent adapters | Pass currently visible products and behavioral history to a rule-based agent, local model, or remote model, and convert their responses. The environment validates each response before executing it. |
| Trace storage and display | Stores the products shown, actions, and results for each round. The frontend updates the display from events. Replay and metric computation use the same records. Human records enter replay through a separate importer. |

**Verified in the prototype.** The browser prototype can import ordered rankings, run a rule-based agent, replay the same simulation, and export JSON. It can also import and replay human trajectories from UserArena. Sample rankings support demonstrations. Actual BPR inference has been completed in a separate RecBole command-line workflow. The LLM request flow has been tested with mock responses.

**To add in the first backend release.** Connect to the recommendation service, move task execution, action validation, and trace storage to the server, and validate an actual agent model. Reuse AfterRec's FastAPI, Pydantic, and Python simulation skeleton, together with the existing replay logic. The recommendation module will continue to use RecBole for model loading and scoring. [AfterRec shared interfaces](../../docs/contracts/INTERFACES.md) · [Simulation runner](../../backend/src/recforgetbench/simulation/runner.py) · [RecBole](https://github.com/RUCAIBox/RecBole)

## 2 Integrating Recommendation Models

In the proposed backend, the runner will use `model_id` to locate a registered model artifact. The environment will call `recommend(user_id, top_k, state)` in the recommendation module. The inputs are the user ID, the number of products requested, and session feedback. The output contains product IDs, recommendation scores, and optional product information in rank order. The recommendation module provides the model's ID mappings, item catalog, and candidate rules.

```text
model_id=bpr-01, recommend(user_id=u01, top_k=4, state={})
→ [{"item_id":"p01","rank":1,"score":0.82}, {"item_id":"p02","rank":2,"score":0.71},
   {"item_id":"p03","rank":3,"score":0.68}, {"item_id":"p04","rank":4,"score":0.56}]
```

**A fixed ranking is the default policy for the first version.** Request the ranking once at the start of each session. For example, a session with a maximum of two pages and two products per page requests four products and preserves their returned order. Continuing the session takes the next page directly from that ranking. The backend design allows a final page with fewer products. An empty ranking ends the session with `no_items`. If the model cannot find the user or the request fails, record the specific reason. A run can evaluate the Original model on its own. When Retrain and Unlearned are available, compare the three arms with the same candidate rules and browsing budget, with separate sessions and agent memory for each arm. Keep model parameters frozen by default.

**Feedback-based recommendations can be added later.** When a feedback policy is selected, send confirmed clicks, dislike feedback, or previously viewed products to the recommendation module before the next page begins. The recommendation module returns a new order for that page. The trace records the original ranks and the actual display ranks separately. Each run saves the recommendation policy and its version so that the conditions producing the results remain clear. The fixed-ranking and feedback-based policies each determine where the next page comes from. Deletion intent is used for agent interviews, and product association labels are used for evaluation. The recommendation interface continues to use the agreed session feedback. [AfterRec recommendation contract](../../docs/contracts/INTERFACES.md#recommendation)

## 3 Agent Actions and Display Updates

The first version requests one decision from the agent per page. The decision includes which products on that page to view, their ratings where applicable, and whether to continue or exit after viewing them. The environment validates the entire decision before committing it as a single unit. The example below uses two pages from the same session.

| Behavior | Environment and display handling |
| --- | --- |
| View or click | Only products on the current page can be selected. The environment records the opened products. The interface displays their details in the order specified by the decision, then returns to the product list or proceeds directly to the next page. |
| Rate a product | In original-model mode, viewed products can receive ratings from 1 to 5. Each rating is saved with its product and displayed on the detail view or product card. The recommendation module supplies recommendation scores. The agent supplies product ratings. |
| Skip or continue | The agent can continue without viewing any products. The environment retrieves the next batch and updates the page. A skip in the earlier unlearning comparison mode maps to this case. |
| Exit or end | Browsing ends when the agent exits, the page limit is reached, or the ranking is exhausted. Save the reason and collect satisfaction or interview responses according to the mode. Satisfaction rates the overall session experience on a 1–10 scale. |

For example, on the page `[p01, p02]`, the agent chooses to open `p01`, rate it 4, and continue. The frontend displays the first page, the details and rating for `p01`, and then the second page `[p03, p04]`. The detail views expand the same page decision in sequence. The trace records one decision for that page. A click from the earlier unlearning comparison mode can be converted into one view, with a missing rating left empty.

**Extending the action set.** If later versions need further decisions on a product detail page, or support back navigation, dislike feedback, or adding items to a cart, add the corresponding states and actions to the environment. Each action needs defined parameters, valid screens, a resulting state, and logging behavior. Decisions made one action at a time use a separate action version and action budget, so that a whole-page decision and a single click are counted at their respective levels of granularity.

## 4 A Shared Interface for Different Agents

**Runner → agent adapter.** In the proposed backend, `AgentPolicy.act(observation)` will receive an `observation_id`, a user profile, currently visible products, executed actions, and the rules for that decision. It returns a candidate decision. A rule-based agent can return it directly. An LLM adapter constructs the request and parses the response. End-of-session questions use a separate stage, requesting satisfaction, an open-ended response, or a prompted response according to the experiment mode.

```text
observation_id=o1, items=[p01,p02], action_schema=page_v1
decisions=[next,exit], rating_range=[1,5]
→ {"viewed":[{"item_id":"p01","rating":4}],"decision":"next","comment":"I'd like to keep browsing"}
```

The observation contains only products visible at that point and actions that have already occurred. Model-arm identifiers and product association labels are kept in run or evaluation records. Comparisons use the same profiles and prompting rules. When adding an agent, register its model, prompt version, and supported input types. An agent that supports images can receive references to images of the current screen and return decisions under the same action contract. The backend design provides a limited number of retries for timeouts, formatting errors, and invalid actions, as specified by the run configuration. On failure, retain the reason and any committed trace.

**Runner → environment.** The proposed `reset(config, user_id, model_id)` initializes a session and returns the first page observation. `step(trace_id, observation_id, action_id, decision)` checks the page number, duplicate products, and rating ranges, then saves the accepted event and new state together. It returns the committed event and either the next page observation or a termination reason. Retrying the same action ID with unchanged content returns the original result. Submitting different content under the same ID, or acting on an outdated page, returns an error. Check again for cancellation before committing.

```text
reset(config=run_config, user_id=u01, model_id=bpr-01) → trace=t01, observation=o1
step(t01, o1, a01, viewed=[p01:4], decision=next) → event_seq=2, observation=o2
```

## 5 Recording Traces for Live Display and Replay

The proposed first backend release will record three event types. Before querying the agent, `page_shown` saves the products actually displayed and their order. `page_decided` saves the accepted decision for the whole page. `session_ended` saves the termination reason and session-level results. Interview responses are saved by question stage. Each event uses `trace_id` to identify its session and `seq` to determine playback order. Errors and rejected actions go into diagnostic records.

```json
{"trace_id":"t01","seq":2,"type":"page_decided","observation_id":"o1",
 "viewed":[{"item_id":"p01","rating":4}],"decision":"next"}
```

Here, `seq=1` is the exposure event for `o1` in the same session and stores `[p01, p02]`. After receiving `seq=2`, the frontend can display the details and rating for `p01`. It changes the product list when the next exposure event arrives. Live display and historical replay use the same rules. After a refresh, the proposed backend will support reading subsequent events using the session and the last received `seq`. Replay reads saved product snapshots and events.

| Interface or artifact | Contract |
| --- | --- |
| Start and inspect a run | The frontend calls `POST /api/v1/simulation-runs` with models, users, the agent, the maximum page count `max_pages`, and the number of items per page `page_size`, and receives a `run_id`. The run ID is used to inspect status and references to traces and results. Trace reads pass `trace_id` and `after_seq`. These routes are part of the proposed backend design. |
| Run directory | `config.json` saves model artifact IDs, dataset and split versions, the forget-manifest ID when applicable, the Git revision, agent and prompt versions, the seed, budgets, and the recommendation policy. `traces.jsonl` saves ordered events. `metrics.json` saves metrics and sample counts. All three files share the same `run_id`. |
| Compatibility with earlier records | Read records according to `schema_version`. A converter turns `rounds` from browser exports into replay events. Fields required by newer formats remain missing when the original records do not provide them. Future actions and metrics have separately registered action and evaluation protocol versions. |

## 6 Sources and Processing of Human Trajectories

Human replay uses the official UserArena `data/sessions.jsonl`. The existing prototype has loaded its 535 sessions. Each line is one session. `session_id` identifies the session, `feed_catalog` contains product information, and `trajectory` contains actions and observations in their recorded order. A generic JSON import entry point is also available. When changing data sources, a converter maps them into a common format. [UserArena data documentation](https://github.com/RecGPT-Team/UserArena#data-and-environment)

```text
Example input: action=scroll_down, scroll_top=0, scroll_delta_px=320
Converted event: type=scroll, direction=down, offset_before_px=0, delta_px=320
```

In this example, the user scrolls down 320 pixels from a position of 0 pixels. The step uses `visible_items` to display the products visible before the action. Later records provide the screen after the action. `gui_image` is the reconstructed screenshot for that step. `feed_long_image` is the full-page feed image for the session. When images are unavailable, product information is used to reconstruct the display. When timing is missing, playback uses evenly spaced steps. Preserve original action names and file provenance so that the conversion can be checked. Obtain UserArena data separately under its non-commercial research license. This repository uses fictional examples. [UserArena data license](https://github.com/RecGPT-Team/UserArena/blob/main/LICENSE)

Side-by-side replay of human and agent trajectories supports inspection of behavioral differences. Before computing action frequencies or behavioral similarity, select sessions with comparable tasks and product contexts, and align the granularity of actions. Records that contain only an exposure state can be replayed. Behavioral statistics count actual actions.

## 7 Development Sequence and Review Decisions

| Stage | Effort | Acceptance criteria |
| --- | --- | --- |
| 1 Align interfaces | 1 day | Agree with the recommendation and unlearning modules on model artifacts, candidate rules, and ranking fields. Use a shared example with four products for integration testing. |
| 2 Connect the first version | 3 days | First run the flow with file-based rankings and a rule-based agent, then connect BPR from the recommendation module. Check product order, ratings, pagination, and exit behavior. |
| 3 Storage and replay | 3 days | Live display and historical replay present the same trace. Verify repeated actions, reading subsequent events after refresh, cancellation, and the order of products in human viewports. |
| 4 Extend integration | 3 days | Connect an actual LLM and verify timeouts and malformed responses. Once the unlearning module's artifacts are ready, run the three-arm comparison and independently recalculate metrics. |

The initial estimate is 10 working days for one developer. During review, first confirm candidate filtering, the actions allowed in the first version, and the basis for human comparisons. Then decide when to develop feedback-based reranking and decisions made one action at a time on product detail pages.
