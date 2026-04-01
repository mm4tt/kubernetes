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

## Pod Nomination Re-queueing
- A pod's nomination can be cleared by the `preemption.Executor` when a higher-priority pod is scheduled on the same node.
- Contrary to previous documentation, clearing this nomination did not automatically move the pod to the active queue, causing it to stay in `unschedulablePods` until the next periodic flush (5 min).
- The fix involves explicitly checking for `oldPod.NNN != "" && newPod.NNN == ""` during the `Update` process.
