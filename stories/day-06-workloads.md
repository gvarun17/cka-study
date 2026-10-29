# Day 6: Deployments, config and resources

**Date:** Wed 2026-11-04 · **Domain:** Workloads & Scheduling (15%) · **Time box:** 2.5h

## Story

As a CKA candidate, I want to deploy, configure, roll out and constrain applications fluently, so that workload tasks take minutes and leave time for troubleshooting.

## Tasks

- [ ] Deployments: create, scale, `set image`, `rollout status/history/undo --to-revision`, `rollout pause/resume`; change the strategy (maxSurge/maxUnavailable)
- [ ] ConfigMaps and Secrets from literals and files; consume them as env (`envFrom`, `valueFrom`) and as mounted volumes
- [ ] Requests and limits; see the QoS class in `k describe pod`
- [ ] LimitRange and ResourceQuota in a namespace; watch a pod get rejected
- [ ] HPA: `k autoscale deploy`; check metrics-server with `k top`
- [ ] Jobs and CronJobs: completions, parallelism, backoffLimit, schedule

## Acceptance criteria

- [ ] Roll out a bad image, notice it, and roll back to a named revision in under 3 minutes
- [ ] Pod using one ConfigMap key as env and a Secret as a volume, verified with `exec`, in under 5 minutes
- [ ] Explain Guaranteed / Burstable / BestEffort QoS without notes

## Docs

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Limit ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

## Notes

