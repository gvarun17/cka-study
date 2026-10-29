# Day 7: Scheduling and mini mock

**Date:** Thu 2026-11-05 · **Domain:** Workloads & Scheduling (15%) · **Time box:** 3h

## Story

As a CKA candidate, I want to control where pods land and why they stay Pending, so that I can solve placement tasks and their related troubleshooting.

## Tasks

- [ ] Taints and tolerations: `k taint nodes`, remove a taint with a trailing `-`, NoSchedule vs NoExecute; why the control plane is tainted
- [ ] nodeSelector and node affinity (required vs preferred); pod affinity and anti-affinity with `topologyKey`
- [ ] `nodeName` (bypasses the scheduler) vs nodeSelector
- [ ] PriorityClass and preemption
- [ ] Multi-container pods: sidecar, init containers, shared emptyDir
- [ ] **Mini mock (1h):** 8 mixed tasks from Days 1–7, timed, no notes

## Acceptance criteria

- [ ] Spread 3 replicas across nodes with anti-affinity, in under 5 minutes
- [ ] Given a Pending pod, state the reason from `describe` Events within 1 minute
- [ ] Mini mock: 6/8 or better. Copy every miss into Day 8 Notes.

## Docs

- [Taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Assigning pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Pod priority and preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

## Notes

