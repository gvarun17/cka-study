# Day 1: Lab cluster and speed setup

**Date:** Fri 2026-10-30 · **Domain:** Setup · **Time box:** 3h

## Story

As a CKA candidate, I want my own kubeadm cluster and a fast kubectl workflow, so that I can practice control-plane tasks that EKS never exposes.

## Tasks

- [ ] Create 2 Ubuntu VMs (EC2 t3.medium or Multipass, 2 vCPU / 4 GB each): `cp1` and `worker1`
- [ ] On both: disable swap, load `overlay` + `br_netfilter`, set sysctl `net.ipv4.ip_forward=1`
- [ ] On both: install containerd and set `SystemdCgroup = true` in `/etc/containerd/config.toml`
- [ ] On both: add the pkgs.k8s.io repo for **one minor version below the exam version** (so Day 3 has something to upgrade to), then install kubeadm, kubelet and kubectl and `apt-mark hold` them
- [ ] `kubeadm init --pod-network-cidr=192.168.0.0/16` on `cp1`, and copy the admin kubeconfig
- [ ] Install a CNI (Calico or Cilium), then `kubeadm join` on `worker1`
- [ ] Apply everything in [speed-setup](../cheatsheets/speed-setup.md)
- [ ] Bookmark the docs pages listed below

## Acceptance criteria

- [ ] `k get nodes` shows 2 nodes `Ready`
- [ ] Without notes, generate YAML for a pod, deployment, service, configmap, secret, job and cronjob using `$do`, in under 5 minutes total
- [ ] Find the kubeadm install page and the etcd backup page in the docs in under 30 seconds each

## Docs

- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

## Notes

