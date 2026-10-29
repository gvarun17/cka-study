# Day 10: PV, PVC and StorageClass

**Date:** Sun 2026-11-08 · **Domain:** Storage (10%) · **Time box:** 2h

## Story

As a CKA candidate, I want to provision and attach persistent storage, statically and dynamically, so that I can bind volumes correctly and debug PVCs stuck in Pending.

## Tasks

- [ ] Static: create a hostPath PV, then a PVC that binds to it. Match on capacity, accessModes and storageClassName.
- [ ] Dynamic: install local-path-provisioner; create a StorageClass with `reclaimPolicy`, `volumeBindingMode: WaitForFirstConsumer` and `allowVolumeExpansion`
- [ ] Mount a PVC into a pod and write data; delete the pod, recreate it, and check the data is still there
- [ ] Reclaim policies Retain vs Delete; what happens to a Released PV
- [ ] Set a default StorageClass (annotation)
- [ ] Expand a PVC

## Acceptance criteria

- [ ] PV + PVC + pod with the volume mounted at a given path, in under 6 minutes
- [ ] Given a Pending PVC, name the mismatch within 1 minute
- [ ] Explain RWO vs ROX vs RWX vs RWOP without notes

## Docs

- [Persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Configure a pod to use a PersistentVolume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
- [Storage classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

## Notes

