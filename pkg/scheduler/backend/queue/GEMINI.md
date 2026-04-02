# Scheduling Queue - Deep Dive Findings

During the investigation of issue #129948, several critical aspects of the `PriorityQueue` were identified.

## Two Paths of `Update`
The `PriorityQueue.Update` method currently has two distinct logic paths based on the `SchedulingQueueHint` feature gate:

1. **Hints Enabled (Modern Path)**:
   - Uses `PodSchedulingPropertiesChange` to extract specific events from pod changes.
   - Relies on `isPodWorthRequeuing` and registered `QueueingHint` functions from plugins.
   - **Crucially**, it tracks "In-flight" pods using the `inFlightPods` map. If a pod is being scheduled, updates are queued as events rather than moving the pod back to the active queue.

2. **Legacy Path (Fallback)**:
   - Used when hints are disabled or when no plugin handles the specific update event.
   - Uses `isPodUpdated(oldPod, newPod)` to decide if a pod should be moved to the active queue.
   - **Historical Bug**: The `strip` function inside `isPodUpdated` was explicitly removing `NominatedNodeName` from the comparison, meaning nomination changes alone never triggered a re-queue in this path.

## In-Flight Tracking and Feature Gates
- In-flight tracking (the ability to know if a pod is currently being scheduled) is tightly coupled with `isSchedulingQueueHintEnabled`.
- If hints are disabled, the `inFlightPods` map in `ActiveQueue` remains empty. This can lead to race conditions where a pod is simultaneously being scheduled and exists in the scheduling queue if an `Update` event occurs during the scheduling cycle.

## Scheduling Queue Pools and Backoff Logic

The `PriorityQueue` manages pods across three main internal structures: `activeQ`, `backoffQ`, and `unschedulablePods`.

### `unschedulablePods` pool
- **Purpose**: Stores pods that failed scheduling and are waiting for a **cluster event** to become schedulable again.
- **Backoff State**: Pods in this pool **can be in backoff** (if their last attempt was recent).
- **Movement**: When an event occurs, the scheduler checks if the pod is still backing off using `p.backoffQ.isPodBackingoff(pInfo)`. 
  - If it is backing off -> it moves to `backoffQ`.
  - If not -> it moves to `activeQ`.

### `backoffQ` (Backoff Queue)
- **Purpose**: A "waiting room" for pods that have **already been triggered by an event** but cannot move to `activeQ` yet because their backoff time hasn't expired.
- **Logic**: These pods no longer wait for events; they only wait for the **clock**.
- **Movement**: A background goroutine (`flushBackoffQCompleted`) periodically moves pods whose backoff has completed from `backoffQ` to `activeQ`.

### Re-queueing Strategy
When re-queueing a pod (e.g., after a nomination update), it is crucial to use a strategy that honors these pools:
- **`queueAfterBackoff`**: The standard choice. Honors the backoff period by placing the pod in `backoffQ` if necessary.
- **`queueImmediately`**: Forcefully moves the pod to `activeQ`, bypassing the backoff check. This should be used sparingly as it can lead to tight scheduling loops (as seen in integration test failures).

## Pod Nomination Re-queueing
- A pod's nomination can be cleared by the `preemption.Executor` when a higher-priority pod is scheduled on the same node.
- Contrary to previous documentation, clearing this nomination did not automatically move the pod to the active queue, causing it to stay in `unschedulablePods` until the next periodic flush (5 min).
- The fix involves explicitly detecting the `oldPod.NNN != "" && newPod.NNN == ""` transition.
- **Caution**: The re-queueing must respect backoff to avoid race conditions and redundant scheduling cycles, especially when a pod's nomination is cleared as part of its own scheduling failure (e.g., in `handleSchedulingFailure`).
