# Day 2: kubeadm internals and etcd backup/restore

**Date:** Sat 2026-10-31 · **Domain:** Cluster Architecture (25%) · **Time box:** 3h

## Story

As a CKA candidate, I want to understand how a kubeadm control plane runs and how to back up and restore etcd, so that I can recover a cluster's state under exam time pressure.

## Tasks

- [ ] List and read every manifest in `/etc/kubernetes/manifests/`. Explain what each flag in `kube-apiserver.yaml` that points at a cert or etcd does.
- [ ] Map `/etc/kubernetes/pki/` and `/etc/kubernetes/*.conf`: which component uses which file
- [ ] Find the kubelet's config (`/var/lib/kubelet/config.yaml`) and its `staticPodPath`
- [ ] Create your own static pod on `worker1`, then delete it the right way (remove the file)
- [ ] `kubeadm certs check-expiration`
- [ ] Back up etcd, create a few objects, restore, and confirm the objects are gone ([etcd cheat sheet](../cheatsheets/etcd.md))

## Acceptance criteria

- [ ] etcd snapshot plus verified status in under 4 minutes, with cert paths read from the manifest rather than memorized
- [ ] Full restore (new data dir, manifest edited, API server back) in under 8 minutes
- [ ] Say from memory which 4 control-plane components run as static pods, and where their logs live when `kubectl` is down

## Docs

- [Operating etcd clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Static pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [PKI certificates](https://kubernetes.io/docs/setup/best-practices/certificates/)
- [Certificate management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)

## Notes

