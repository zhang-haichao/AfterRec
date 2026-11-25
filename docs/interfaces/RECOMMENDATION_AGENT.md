# Recommendation and agent simulation interface

[简体中文](RECOMMENDATION_AGENT.zh-CN.md) · [Documentation](../README.md) · [Module design](../modules/agent-simulation/DESIGN.md)

**Interface status:** Proposed for review by both modules. The table identifies existing repository contracts, local prototype calls and proposed handoffs.

## 1. Use case

When a researcher selects a user and the Original model, **recommendation** supplies that user’s ranked products. **Agent simulation** consumes the ranking and runs the shopping session. The **experiment platform** archives and retrieves the result by run ID. The recommender sets the order, the agent chooses an action, and the environment validates and records it. Human replay reads an external session file independently of these calls.

## 2. Operation and input/output

| Operation or artifact | Input | Output |
| --- | --- | --- |
| Get a ranking (existing Python contract) `recommender.recommend(user_id, top_k, state)` | A loaded, frozen model, user ID, requested count and session feedback | A sequence of `RecommendedItem` in rank order, including stable item ID, base score and rank |
| Initialize the store (local browser API) `beginAgent({products})` | Ordered products. Each requires a unique nonempty string `id` (matching `item_id`), a nonempty `name` and a nonnegative numeric `price` | First observation of a fresh session. Integration retains the original ranking with the session and joins display metadata by ID |
| Interact one step at a time (local browser API) `observe()`, `act(action)` | The policy reads the current screen and submits an action. A review requires the current detail item and 1–5 stars | An accepted action returns an event, and the next observation reflects the change. An invalid action returns a reason and leaves the view and event sequence intact |
| Deliver a trace (local export, platform archiving proposed) `exportTrace()` | Current shopping session | An `afterrec.objects.v1` JSON file with the product snapshot and ordered events. The platform would associate it with run, user and model IDs for replay and evaluation |

The browser methods belong to `window.afterrecStore`. For example, `window.afterrecStore.beginAgent({products: [{id: "P17", name: "Lamp", price: 99}]})` starts a local session with one product.

**When to request recommendations.** The repository’s current `run_episode` calls the recommender at each step. The proposed shopping mode requests a ranking at session start and reuses it when opening a product, rating it or returning to the list. The adapter joins display data such as prices by ID and preserves the ranking. Missing required metadata should identify the affected item and field.

**Observations and actions.** In the browser, `items` contains the results on the current page, while `inViewport` identifies items in the visible scroll area. The integrated policy should receive only the currently visible information, approved persona and prior executed actions. The repository’s `AgentAction` supports click, dislike, skip and exit. The shopping action format also describes opening details, reviewing and returning. Backend integration must preserve the action format version and trace source so replay and analysis can interpret each step. [Shared types and methods](../contracts/INTERFACES.md) · [Repository runner](../../backend/src/recforgetbench/simulation/runner.py)

**Minimal integration example.** Recommendation returns `P17` first and `P42` second. Simulation displays them in that order, opens `P17` and submits `{type: "review", id: "P17", stars: 4}`. Expect a review event for `P17` with its detail view still open. Returning restores the original list order.

