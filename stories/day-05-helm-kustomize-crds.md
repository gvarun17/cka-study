# Day 5: Helm, Kustomize, CRDs and operators

**Date:** Tue 2026-11-03 · **Domain:** Cluster Architecture (25%) · **Time box:** 2.5h

## Story

As a CKA candidate, I want to install and manage components with Helm and Kustomize and work with CRDs, so that I can handle the extension-related tasks added in the 2025 curriculum.

## Tasks

- [ ] Helm: `repo add/update`, `search repo`, `install` with `--set` and `-f values.yaml`, `-n <ns> --create-namespace`, `list -A`, `upgrade`, `history`, `rollback`, `uninstall`
- [ ] `helm show values <chart>` to find the right key, then `helm template` to preview the manifests
- [ ] Kustomize: base + overlay with `namePrefix`, `commonLabels`, image override and a patch; `k kustomize <dir>` then `k apply -k <dir>`
- [ ] CRDs: `k get crd`, `k explain <kind>`; install an operator (for example cert-manager via Helm or its manifest) and create one custom resource
- [ ] Find which API group and version a CRD serves: `k api-resources | grep <name>`

## Acceptance criteria

- [ ] Install a chart into a new namespace with 2 overridden values, then upgrade and roll back, in under 5 minutes
- [ ] Build and apply a kustomize overlay that changes the image tag and replicas, in under 5 minutes
- [ ] Find a CRD's fields with `k explain` and create a valid object, in under 4 minutes

## Docs

- [Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Declarative management with Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

## Notes

