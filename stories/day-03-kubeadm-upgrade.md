# Day 3: kubeadm cluster upgrade

**Date:** Sun 2026-11-01 · **Domain:** Cluster Architecture (25%) · **Time box:** 2.5h

## Story

As a CKA candidate, I want to upgrade a kubeadm cluster by one minor version, node by node, so that I can do it correctly and quickly when the exam asks.

## Tasks

- [ ] Upgrade `cp1` following the [kubeadm upgrade cheat sheet](../cheatsheets/kubeadm-upgrade.md)
- [ ] Upgrade `worker1` (drain from cp1, SSH in, `kubeadm upgrade node`, kubelet, uncordon)
- [ ] Learn the drain flags: `--ignore-daemonsets`, `--delete-emptydir-data`, `--force`, and when each is needed
- [ ] Redo the Day 2 etcd backup and restore from memory
- [ ] Rebuild the cluster at the old version (or snapshot your VMs) so you can repeat the upgrade

## Acceptance criteria

- [ ] `k get nodes` shows both nodes on the new version and `Ready`
- [ ] Control-plane upgrade in under 10 minutes; worker upgrade in under 6
- [ ] Explain why the worker runs `kubeadm upgrade node` and not `apply`
- [ ] etcd backup and restore from memory in under 10 minutes

## Docs

- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Changing the package repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
- [Safely drain a node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

## Notes

