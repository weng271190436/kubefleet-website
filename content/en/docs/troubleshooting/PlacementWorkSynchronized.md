---
title: Work Synchronization Failure TSG
description: Troubleshooting guide for "WorkSynchronized" condition set to false (ClusterResourcePlacementWorkSynchronized / ResourcePlacementWorkSynchronized)
weight: 5
---

The `ClusterResourcePlacementWorkSynchronized` (CRP) or `ResourcePlacementWorkSynchronized` (RP) condition is `False` when the placement has been recently updated but the associated Work objects have not yet been synchronized with the latest selected resources.
> Note: In addition, it may be helpful to look into the logs for the [work generator controller](https://github.com/kubefleet-dev/kubefleet/blob/main/pkg/controllers/workgenerator/controller.go) to get more information on why the work synchronization failed.

## Common Scenarios
Instances where this condition may arise:
- The controller encounters an error while trying to generate the corresponding `work` object.
- The enveloped object is not well formatted.

### Case Study
Example: A placement attempts to propagate resources to a selected cluster, but the Work object cannot update because the member cluster namespace is terminating.

### ClusterResourcePlacement Spec (same pattern applies to ResourcePlacement with namespace-scoped resources)
```
spec:
  resourceSelectors:
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
      name: secret-reader
      version: v1
  policy:
    placementType: PickN
    numberOfClusters: 1
  strategy:
    type: RollingUpdate
 ```

### ClusterResourcePlacement Status (Representative Example)
```
spec:
  policy:
    numberOfClusters: 1
    placementType: PickN
  resourceSelectors:
  - group: ""
    kind: Namespace
    name: test-ns
    version: v1
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
status:
  conditions:
  - lastTransitionTime: "2024-05-14T18:05:04Z"
    message: found all cluster needed as specified by the scheduling policy, found
      1 cluster(s)
    observedGeneration: 1
    reason: SchedulingPolicyFulfilled
    status: "True"
    type: ClusterResourcePlacementScheduled
  - lastTransitionTime: "2024-05-14T18:05:05Z"
    message: All 1 cluster(s) start rolling out the latest resource
    observedGeneration: 1
    reason: RolloutStarted
    status: "True"
    type: ClusterResourcePlacementRolloutStarted
  - lastTransitionTime: "2024-05-14T18:05:05Z"
    message: No override rules are configured for the selected resources
    observedGeneration: 1
    reason: NoOverrideSpecified
    status: "True"
    type: ClusterResourcePlacementOverridden
  - lastTransitionTime: "2024-05-14T18:05:05Z"
    message: There are 1 cluster(s) which have not finished creating or updating work(s)
      yet
    observedGeneration: 1
    reason: WorkNotSynchronizedYet
    status: "False"
    type: ClusterResourcePlacementWorkSynchronized
  observedResourceIndex: "0"
  placementStatuses:
  - clusterName: kind-cluster-1
    conditions:
    - lastTransitionTime: "2024-05-14T18:05:04Z"
      message: 'Successfully scheduled resources for placement in kind-cluster-1 (affinity
        score: 0, topology spread score: 0): picked by scheduling policy'
      observedGeneration: 1
      reason: Scheduled
      status: "True"
      type: Scheduled
    - lastTransitionTime: "2024-05-14T18:05:05Z"
      message: Detected the new changes on the resources and started the rollout process
      observedGeneration: 1
      reason: RolloutStarted
      status: "True"
      type: RolloutStarted
    - lastTransitionTime: "2024-05-14T18:05:05Z"
      message: No override rules are configured for the selected resources
      observedGeneration: 1
      reason: NoOverrideSpecified
      status: "True"
      type: Overridden
    - lastTransitionTime: "2024-05-14T18:05:05Z"
      message: 'Failed to synchronize the work to the latest: works.placement.kubernetes-fleet.io
        "crp1-work" is forbidden: unable to create new content in namespace fleet-member-kind-cluster-1
        because it is being terminated'
      observedGeneration: 1
      reason: SyncWorkFailed
      status: "False"
      type: WorkSynchronized
  selectedResources:
  - kind: Namespace
    name: test-ns
    version: v1
```
The status shows `WorkSynchronized` `False` because the Work object cannot create new content while the namespace is terminating.

### Resolution
Potential actions:
- Select a different (healthy) cluster or remove the failing one from the policy.
- Delete the placement to allow garbage collection of associated Work objects.
- Rejoin the member cluster so its namespace can be recreated and Work can synchronize.
- Wait briefly: transient sync failures (API server unavailability, temporary RBAC errors) often resolve automatically.

## General Notes
For ResourcePlacement the investigation is identical—inspect `.status.placementStatuses[*].conditions` for `WorkSynchronized` and check the associated Work in the `fleet-member-{clusterName}` namespace.
