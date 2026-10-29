# Day 4: RBAC and kubeconfig

**Date:** Mon 2026-11-02 · **Domain:** Cluster Architecture (25%) · **Time box:** 2.5h

## Story

As a CKA candidate, I want to grant exactly the permissions a user or ServiceAccount needs, and to switch contexts cleanly, so that I can answer access-control tasks fast and verify them.

## Tasks

- [ ] Create Role + RoleBinding imperatively: `k create role`, `k create rolebinding` (with `--serviceaccount=ns:name` and `--user`)
- [ ] Create ClusterRole + ClusterRoleBinding; then bind a ClusterRole with a namespaced RoleBinding, and explain the result
- [ ] Create a ServiceAccount, attach it to a deployment, and check its permissions with `k auth can-i --as=system:serviceaccount:<ns>:<sa>`
- [ ] Practice `k auth can-i --list --as=<user> -n <ns>`
- [ ] kubeconfig: `k config get-contexts`, `use-context`, `set-context --current --namespace=`, `view --minify`
- [ ] Optional: CSR flow (create a user cert, approve it with `k certificate approve`, build a kubeconfig entry)

## Acceptance criteria

- [ ] Five RBAC tasks of different shapes in under 3 minutes each, each verified with `auth can-i`
- [ ] Explain Role vs ClusterRole vs RoleBinding vs ClusterRoleBinding without notes
- [ ] Zero wrong-context mistakes across the day's drills

## Docs

- [Using RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Configure ServiceAccounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Organizing cluster access using kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [Certificate signing requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

## Notes

