# Day 11: Break and fix the cluster

**Date:** Mon 2026-11-09 · **Domain:** Troubleshooting (30%) · **Time box:** 3h

## Story

As a CKA candidate, I want to diagnose and fix broken nodes and control-plane components without a working `kubectl`, so that I can earn the heaviest-weighted domain.

## Tasks

Break each one yourself (or have someone break it), wait a minute, then fix it using the [troubleshooting playbook](../cheatsheets/troubleshooting.md):

- [ ] `systemctl stop kubelet` on the worker
- [ ] Wrong `--client-ca-file` path in `/var/lib/kubelet/config.yaml`
- [ ] Typo in a flag in `kube-apiserver.yaml` (for example `--etcd-servers` port)
- [ ] Wrong image tag in `kube-scheduler.yaml`, then create a pod and watch it stay Pending
- [ ] Break `kube-controller-manager.yaml`, then scale a deployment and watch nothing happen
- [ ] Stop containerd
- [ ] Practice `crictl ps -a`, `crictl logs`, `journalctl -u kubelet -f`, `/var/log/pods/`
- [ ] Collect cluster info: `k cluster-info`, `k get events -A --sort-by=.lastTimestamp`, `k top nodes/pods`

## Acceptance criteria

- [ ] Each of the 6 faults found and fixed in under 6 minutes
- [ ] From memory, state the order you check things for "kubectl connection refused"
- [ ] Never edit a static pod manifest without first copying it to `/tmp`

## Docs

- [Troubleshooting clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Troubleshooting applications](https://kubernetes.io/docs/tasks/debug/debug-application/)

## Notes

