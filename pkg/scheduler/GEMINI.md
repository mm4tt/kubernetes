# Scheduling Cycle and Nominations - Deep Dive Findings

Analysis performed during the investigation of pod re-queueing mechanisms revealed how `NominatedNodeName` (NNN) is managed outside of pure preemption.

## Nominal Node Lifecycle

### When is NNN set?
1. **Post-Filter (Preemption)**: The most common case. If a pod doesn't fit, preemption plugins suggest a node, and the scheduler nominates it.
2. **NominatedNodeNameForExpectation**: Since v1.30, the scheduler sets NNN *before* the pod is bound, specifically when the pod is about to wait in the `Permit` phase. This informs external components (like Cluster Autoscaler) that the node is effectively occupied.
   - See `pkg/scheduler/schedule_one.go`: `bindingCycle` method.

### When is NNN cleared?
1. **Successful Binding**: Cleared once the pod is successfully assigned to the node.
2. **Scheduling Failure**: If a pod fails at any point after `schedulingAlgorithm` (e.g., in `Permit` or `Bind`), the `handleSchedulingFailure` or `handleBindingCycleError` will explicitly clear the NNN.
   - This leads to a `non-empty -> empty` transition in the API.

## Re-queueing Side Effects
- The `eventhandlers.go` contains logic that triggers `MoveAllToActiveOrBackoffQueue` (using `EventAssignedPodDelete`) whenever a pod's nomination changes, provided the old nomination was not empty.
- This is intended to wake up *other* pods that might now fit on the node.
- Adding explicit re-queueing logic for the updated pod itself in `PriorityQueue.Update` can lead to "Double Re-queueing" if not careful.

## Integration Test Caveats
- Tests like `TestUnReservePermitPlugins` simulate Permit timeouts. 
- In these tests, the transition `node-X -> ""` (clearing NNN after failure) can trigger the new re-queueing logic, causing the pod to be immediately rescheduled. 
- If the pod is still "In-flight" (e.g., if hints are disabled and tracking is missing), this can result in multiple concurrent scheduling cycles for the same pod, causing unexpected behavior like `Permit` being called multiple times.
