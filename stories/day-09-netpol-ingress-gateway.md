# Day 9: NetworkPolicy, Ingress, Gateway API, CNI

**Date:** Sat 2026-11-07 · **Domain:** Services & Networking (20%) · **Time box:** 3h

## Story

As a CKA candidate, I want to restrict pod traffic and route external HTTP traffic, so that I can write policies that actually block what they should and set up both Ingress and Gateway API routing.

## Tasks

- [ ] NetworkPolicy: default-deny ingress and egress for a namespace; then allow by podSelector, namespaceSelector and ipBlock
- [ ] Learn the AND vs OR trap: one `from` item with both selectors vs two separate items
- [ ] Remember DNS egress (UDP/TCP 53) when you deny egress
- [ ] Test every policy with `k exec` + `wget --timeout=2`, before and after
- [ ] Ingress: install an ingress controller; path-based and host-based rules; `ingressClassName`
- [ ] Gateway API: install the CRDs and an implementation; create a Gateway and an HTTPRoute (path match, header match, weighted backends)
- [ ] CNI: know where config lives (`/etc/cni/net.d/`) and how to install one from a manifest

## Acceptance criteria

- [ ] "Only pods labeled `role=api` in namespace `frontend` may reach `db` pods on port 5432": written and verified in under 6 minutes
- [ ] HTTPRoute that splits traffic 80/20 between two services, in under 6 minutes
- [ ] Explain the AND vs OR selector difference without notes

## Docs

- [Network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Network plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

## Notes

