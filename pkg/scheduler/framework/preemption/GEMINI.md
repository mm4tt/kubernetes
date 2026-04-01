# Preemption Executor - Deep Dive Findings

Analysis of the preemption framework reveals discrepancies between documentation and implementation regarding pod re-scheduling.

## Executor Responsibility
The `preemption.Executor` is responsible for:
1. Identifying victims to be preempted.
2. Clearing the `NominatedNodeName` of lower-priority pods that were previously nominated to the target node.

## Discrepancy in Automatic Re-queueing
- **Documentation**: A long-standing comment in `executor.go` stated that removing a pod's nomination would "move them to the active queue," allowing the scheduler to find a new place for them sooner.
- **Reality**: Before the fix in #129948, this was **not true**. Pods stayed in the `unschedulablePods` pool because the `PriorityQueue` did not consider nomination changes as significant updates that warranted an immediate move to the active queue.

## Fixing the Behavior
To align reality with documentation, the `PriorityQueue.Update` must explicitly detect the transition where `NominatedNodeName` becomes empty (while having a previous non-empty value) and trigger a re-queueing (respecting backoff).
- This ensures that pods displaced by higher-priority preemptors are reconsidered immediately for other nodes.
