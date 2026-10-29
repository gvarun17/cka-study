# Day 8: Services and DNS

**Date:** Fri 2026-11-06 · **Domain:** Services & Networking (20%) · **Time box:** 2.5h

## Story

As a CKA candidate, I want to expose workloads and debug service discovery, so that I can make pods reachable and quickly find why they aren't.

## Tasks

- [ ] ClusterIP, NodePort, LoadBalancer, headless (`clusterIP: None`); `k expose` with `--port` and `--target-port`
- [ ] Inspect `k get endpointslices` and `k get endpoints`; break a selector and watch endpoints go empty
- [ ] DNS names: `<svc>.<ns>.svc.cluster.local`, pod A records, headless service records
- [ ] CoreDNS: its ConfigMap, its pods, and `/etc/resolv.conf` inside a pod
- [ ] Debug flow: test pod → `nslookup` → `wget -qO- <svc>:<port>` → endpoints → pod labels → targetPort

## Acceptance criteria

- [ ] Expose a deployment as NodePort on a specific nodePort and curl it from the node, in under 3 minutes
- [ ] Fix a service with a wrong selector and a wrong targetPort in under 4 minutes
- [ ] Resolve a service in another namespace from a busybox pod, first try

## Docs

- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for services and pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Debugging DNS resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Debug services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

## Notes

