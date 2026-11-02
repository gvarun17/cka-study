# Day 4 Lesson: RBAC, ServiceAccounts and kubeconfig

**Story:** [day-04-rbac](../stories/day-04-rbac.md) · **Read time:** ~35 min · **Then:** 2 hours in the lab

**Why this matters:** RBAC falls under Cluster Architecture, Installation & Configuration (25%), and nearly every exam has one or two RBAC tasks: "create a ServiceAccount that can only list pods in namespace X", or "user jane must be able to create deployments in Y". They're cheap points. With `k create role` + `k create rolebinding` + `k auth can-i`, each one takes 2–3 minutes. They're also easy to lose: a typo in a resource name or a binding in the wrong namespace gives you zero, and nothing errors to warn you. kubeconfig matters for a different reason. **Every exam question starts with a context command.** Skipping it means doing a correct answer on the wrong cluster, and that scores zero.

On EKS you mapped IAM roles to Kubernetes groups through `aws-auth` or access entries, and then RBAC took over. Everything after "who are you" is the same here. The difference is that **you** issue the identities: client certificates signed by the cluster CA, and ServiceAccount tokens.

> **Version note.** The exam targets Kubernetes v1.37 (see Day 2). Nothing in RBAC (`rbac.authorization.k8s.io/v1`) or CSRs (`certificates.k8s.io/v1`) has changed in the versions between your v1.35 lab and the exam. `kubectl auth whoami` (SelfSubjectReview API, GA since v1.28) works on your cluster, though its kubectl help text still says "Experimental". Since v1.24, ServiceAccounts **don't** get an auto-created long-lived token Secret. Pods get short-lived projected tokens, and `kubectl create token <sa>` mints one on demand. The CSR walkthrough now lives at [Issue a Certificate for a Kubernetes API Client Using a CertificateSigningRequest](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/). Bookmark that path, not the old reference page.

---

## 1. How a request gets authorized

Every API request goes through the same pipeline in kube-apiserver:

```
request ──► Authentication ──► Authorization ──► Admission ──► etcd
            "who are you?"      "may you do this?"  "is the object OK?"
            x509 cert, SA token  Node, RBAC          mutating/validating
            → user + groups      (--authorization-   webhooks, quotas...
                                  mode=Node,RBAC)
```

- **Authentication** turns credentials into a **username** plus a list of **groups**. Kubernetes has **no User object**. A "user" is just a string the authenticator returns:
  - **x509 client cert:** username = cert `CN`, groups = cert `O` fields. Signed by `/etc/kubernetes/pki/ca.crt`.
  - **ServiceAccount token:** username = `system:serviceaccount:<ns>:<name>`, groups = `system:serviceaccounts`, `system:serviceaccounts:<ns>`, `system:authenticated`.
- **Authorization:** kubeadm sets `--authorization-mode=Node,RBAC` (check `/etc/kubernetes/manifests/kube-apiserver.yaml`). The Node authorizer handles kubelets. RBAC handles everyone else. RBAC is **allow-only, deny by default**: no rule matches, the answer is 403. There are no deny rules, so permissions only add up.
- **`system:masters`** skips authorization entirely. That's why kubeadm's `super-admin.conf` is break-glass, and why `admin.conf` uses the `kubeadm:cluster-admins` group instead (Day 2).

### What an RBAC request looks like

The authorizer sees a tuple and looks for a rule that matches it:

| Field | Example | Rule field |
|---|---|---|
| verb | `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection` | `verbs` |
| API group | `""` (core), `apps`, `batch`, `networking.k8s.io` | `apiGroups` |
| resource | `pods`, `deployments`, `pods/log`, `pods/exec`, `deployments/scale` | `resources` |
| name | `app-config` (only for get/update/patch/delete) | `resourceNames` |
| namespace | `dev` (or none, for cluster-scoped) | where the Role/binding lives |
| non-resource URL | `/healthz`, `/metrics` | `nonResourceURLs` (ClusterRole only) |

Two details catch people. `kubectl logs` is `get` on `pods/log`. `kubectl exec` is `create` on `pods/exec` (plus `get` on pods). And `list` can't be limited by `resourceNames`. A list request has no name to match.

---

## 2. The four objects, and the one combination that confuses everyone

| Object | Scope | Says |
|---|---|---|
| **Role** | One namespace | "These verbs on these resources, **in this namespace**" |
| **ClusterRole** | Cluster | "These verbs on these resources", reusable in any namespace or cluster-wide. It's also the **only** way to grant cluster-scoped resources (nodes, PVs, namespaces, CSRs, StorageClasses) and non-resource URLs |
| **RoleBinding** | One namespace | "These subjects get that Role **or ClusterRole**, **in this namespace only**" |
| **ClusterRoleBinding** | Cluster | "These subjects get that ClusterRole **in every namespace**, plus cluster-scoped resources" |

The **binding decides the scope**, not the role:

```
                       RoleBinding (ns: dev)          ClusterRoleBinding
Role (ns: dev)         ✔ perms in dev                 ✘ not allowed
ClusterRole            ✔ perms in dev ONLY            ✔ perms in all namespaces
                         (the ClusterRole is a          + cluster-scoped resources
                          reusable template)
```

**ClusterRole + RoleBinding** is today's "explain the result" task. You get the ClusterRole's rules, **limited to the RoleBinding's namespace**. Rules for cluster-scoped resources in that ClusterRole (nodes, say) **grant nothing**, because a namespaced binding can't reach outside its namespace. This is the normal way to hand someone `edit` in a single namespace: `k create rolebinding dev-edit --clusterrole=edit --user=jane -n dev`.

A RoleBinding's `roleRef` can point at a Role in **its own** namespace or at any ClusterRole. It can't point at a Role in another namespace.

### Subjects

```yaml
subjects:
- kind: User                      # x509 CN, OIDC user...
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers                # x509 O, or system:serviceaccounts:dev
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount            # no apiGroup; namespace is REQUIRED
  name: deployer
  namespace: dev
```

### Built-in ClusterRoles worth knowing

| ClusterRole | Gives | Note |
|---|---|---|
| `cluster-admin` | Everything | With a RoleBinding, everything **in that namespace** (including Roles and bindings) |
| `admin` | Read/write most namespaced resources, incl. Roles/RoleBindings | No ResourceQuota writes, no namespace itself |
| `edit` | Read/write workloads, Services, ConfigMaps, **Secrets** | Can't read or modify Roles/bindings |
| `view` | Read most namespaced resources | **Not Secrets** (reading a SA token would escalate) |

`k get clusterrole | grep -v system:` lists them. `k describe clusterrole edit` is the fastest way to answer "which verbs on which resources" in the exam terminal.

### Immutable roleRef, and escalation protection

- **`roleRef` is immutable.** To bind a different role, delete the binding and create it again. `k edit` will refuse. Subjects can be edited.
- **Escalation prevention:** you can only create or update a Role whose permissions you already hold, and only bind a role you hold (unless you have the `escalate`/`bind` verbs). As admin on the exam you never hit this. It does explain why a namespace `admin` can't hand out `cluster-admin`.
- **Aggregated ClusterRoles:** a ClusterRole with `aggregationRule.clusterRoleSelectors` gets its rules filled in by the controller from every ClusterRole carrying a matching label. `admin`/`edit`/`view` are built this way, so a CRD's ClusterRole labelled `rbac.authorization.k8s.io/aggregate-to-edit: "true"` lands in `edit` automatically. You'll see it again tomorrow with CRDs.

---

## 3. Exam commands: imperative first

```bash
# Role: verbs and resources are comma lists. Use plural resource names (or short names). Group is inferred.
k create role pod-reader -n dev --verb=get,list,watch --resource=pods
k create role deploy-mgr -n dev --verb=get,list,create,update,patch,delete --resource=deployments,replicasets
k create role cm-one -n dev --verb=get,update --resource=configmaps --resource-name=app-config
k create role log-reader -n dev --verb=get --resource=pods,pods/log

# ClusterRole: same flags, plus cluster-only extras
k create clusterrole node-reader --verb=get,list,watch --resource=nodes
k create clusterrole health --verb=get --non-resource-url=/healthz,/livez
k create clusterrole agg --aggregation-rule="team=a"           # empty rules, filled in by the controller

# Bindings: one --role or --clusterrole, then any mix of subjects (each flag repeatable)
k create rolebinding pr-jane -n dev --role=pod-reader --user=jane
k create rolebinding pr-sa   -n dev --role=pod-reader --serviceaccount=dev:deployer   # ns:name, colon!
k create rolebinding dev-view -n dev --clusterrole=view --group=developers
k create clusterrolebinding nr-jane --clusterrole=node-reader --user=jane

# ServiceAccount, plus wiring it to a workload
k create sa deployer -n dev
k set serviceaccount deploy/web deployer -n dev               # patches spec.template.spec.serviceAccountName
k create token deployer -n dev --duration=1h                  # short-lived token, e.g. for a kubeconfig

# Verify. Always.
k auth can-i list pods -n dev --as=jane                                  # yes
k auth can-i list pods -n dev --as=system:serviceaccount:dev:deployer    # yes
k auth can-i delete pods -n dev --as=jane                                # no
k auth can-i get nodes --as=jane                                         # cluster-scoped: no -n
k auth can-i get pods --subresource=log -n dev --as=jane
k auth can-i update configmaps/app-config -n dev --as=jane               # resourceNames check
k auth can-i --list -n dev --as=jane                                     # everything jane may do in dev
k auth can-i list pods -n dev --as=jane --as-group=developers            # test group membership
k auth whoami                                                            # who am I right now?

# Inspect
k get role,rolebinding -n dev
k describe rolebinding pr-jane -n dev
k get clusterrolebinding -o wide | grep jane                             # wide shows ROLE, USERS, GROUPS, SERVICEACCOUNTS
```

**Resource names:** `--resource` takes the plural (`deployments`), a short name (`deploy`) or `resource.group` (`deployments.apps`). If you're unsure of a group, run `k api-resources | grep -i <thing>`. The `APIVERSION` column gives the group, and the `NAMESPACED` column tells you Role or ClusterRole.

### Complete YAML (what `$do` gives you, tidied)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deploy-mgr
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]                     # core group = empty string
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-mgr
  namespace: dev
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: dev
roleRef:                              # immutable
  kind: Role                          # or ClusterRole
  name: deploy-mgr
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: dev
automountServiceAccountToken: true    # set false to stop the token mount (also settable per pod)
```

Multi-group rules (like the one above) are the main reason to drop to YAML. The fastest route is `k create role ... $do > r.yaml`, add the second rule in vim, then `k apply -f r.yaml`.

In a pod or Deployment, the field is `spec.serviceAccountName` (pod template for Deployments). **You can't change it on a running pod.** For a Deployment, `k set sa` triggers a rollout. For a bare pod, delete and recreate it.

---

## 4. kubeconfig: clusters, users, contexts

```
~/.kube/config  (or $KUBECONFIG, a colon-separated list that kubectl merges)
├── clusters:   name → server URL + CA cert
├── users:      name → client cert/key, or token, or exec plugin   (on EKS: the aws eks get-token exec plugin)
├── contexts:   name → (cluster, user, default namespace)
└── current-context: <context name>
```

A context is just a triple. It doesn't hold permissions. Switching context switches the credentials and cluster you talk to.

```bash
k config get-contexts                               # * marks current
k config current-context
k config use-context kubernetes-admin@kubernetes   # the exam gives you this line per question. Run it every time.
k config set-context --current --namespace=dev     # change the default ns of the current context
k config view --minify                              # only the current context's cluster/user (certs redacted)
k config view --minify --raw -o jsonpath='{.clusters[0].cluster.server}'
k config get-users ; k config get-clusters

# Build a new user + context (what the CSR flow ends with)
k config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true
k config set-context jane@kubernetes --cluster=kubernetes --user=jane --namespace=dev
k --context=jane@kubernetes get pods               # one-off use without switching
k config delete-context jane@kubernetes

# Token-based user (e.g. for a ServiceAccount)
k config set-credentials deployer --token=$(k create token deployer -n dev)
```

`--embed-certs=true` copies the cert data into the file (`client-certificate-data`). Without it the kubeconfig holds file paths, and moving the file breaks it. Use `--kubeconfig=/path/file` with any `k config` command to write somewhere other than `~/.kube/config`, which is common when a task says "create a kubeconfig at /opt/jane.conf".

---

## 5. Creating a user with the CSR API

```
jane.key ──openssl req──► jane.csr ──base64──► CertificateSigningRequest
                                                 signerName: kubernetes.io/kube-apiserver-client
                                                        │ k certificate approve jane
                                                        ▼
              kube-controller-manager (csrsigning, uses pki/ca.crt + ca.key) fills status.certificate
                                                        │
                                       base64 -d ──► jane.crt ──► kubeconfig user
```

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=developers"   # CN = username, O = group

cat <<EOF | k apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(base64 -w0 < jane.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400          # optional; 1 day
  usages:
  - client auth
EOF

k get csr                           # jane   Pending
k certificate approve jane          # (or: k certificate deny jane)
k get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
openssl x509 -in jane.crt -noout -subject -enddate                    # subject=O=developers, CN=jane
```

Then the `set-credentials` / `set-context` pair from section 4, plus a binding so jane can do something.

**Shortcut that also works on a kubeadm control plane** (you have the CA key there): `openssl x509 -req -in jane.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out jane.crt -days 1`. Use the CSR API when the task names it. The verifier may check that the CSR object is `Approved,Issued`.

Pending CSRs for **kubelets** (`kubernetes.io/kube-apiserver-client-kubelet`) are approved automatically during bootstrap. Kubelet **serving** certs (`kubernetes.io/kubelet-serving`) aren't, so `k get csr` on a fresh cluster may show those Pending. Leave them unless a task asks.

---

## 6. Hands-on walkthrough on cp1 + worker1

Your lab is now v1.35 after yesterday. None of this depends on the version.

**Step 0: the context habit.** Today's third acceptance criterion is zero wrong-context mistakes. Make it mechanical. Before each drill below, write the drill's context line and run it, the way the exam shows one per question:

```bash
k config get-contexts
k config rename-context kubernetes-admin@kubernetes admin    # optional: shorter to type in the lab
```

Later in the day you'll have a `jane` context too, and the drills alternate between them. That's the real test.

**Step 1: Role + RoleBinding, imperatively (task 1).** ⏱ Each block is one "exam task". Target < 3 min including verification.

```bash
k create ns dev
# Task A: user jane may get/list/watch pods and read pod logs in dev
k create role pod-reader -n dev --verb=get,list,watch --resource=pods,pods/log
k create rolebinding pod-reader-jane -n dev --role=pod-reader --user=jane
k auth can-i list pods -n dev --as=jane                   # yes
k auth can-i get pods --subresource=log -n dev --as=jane  # yes
k auth can-i list pods -n default --as=jane               # no: binding is in dev only
k auth can-i delete pods -n dev --as=jane                 # no

# Task B: SA ci in dev may manage deployments in dev
k create sa ci -n dev
k create role deploy-mgr -n dev --verb=create,get,list,update,patch,delete --resource=deployments
k create rolebinding deploy-mgr-ci -n dev --role=deploy-mgr --serviceaccount=dev:ci
k auth can-i create deployments.apps -n dev --as=system:serviceaccount:dev:ci   # yes
k auth can-i create deployments -n dev --as=ci                                  # no! wrong identity string
```

That last `no` is on purpose. `--as=ci` is a *user* called ci, not the ServiceAccount. Keep that output in mind.

**Step 2: ClusterRole + ClusterRoleBinding, then a ClusterRole in a RoleBinding (task 2).**

```bash
# Task C: jane may list nodes and PVs cluster-wide
k create clusterrole infra-reader --verb=get,list,watch --resource=nodes,persistentvolumes,pods
k create clusterrolebinding infra-reader-jane --clusterrole=infra-reader --user=jane
k auth can-i list nodes --as=jane                         # yes
k auth can-i list pods -n kube-system --as=jane           # yes: CRB = every namespace

# Task D: the same ClusterRole, but for bob via a RoleBinding in dev
k create rolebinding infra-reader-bob -n dev --clusterrole=infra-reader --user=bob
k auth can-i list pods -n dev --as=bob                    # yes
k auth can-i list pods -n kube-system --as=bob            # no
k auth can-i list nodes --as=bob                          # no: cluster-scoped rules can't flow through a RoleBinding
```

Write one sentence in the story Notes explaining bob's three results. That's your "explain the result" answer.

**Step 3: ServiceAccount on a Deployment, seen from inside the pod (task 3).**

```bash
# Task E: deployment web in dev runs as SA ci; prove the pod can list deployments but not secrets
k create deploy web -n dev --image=curlimages/curl -- sleep infinity
k set sa deploy/web ci -n dev
k -n dev get pod -l app=web -o jsonpath='{.items[0].spec.serviceAccountName}{"\n"}'   # ci
k -n dev exec deploy/web -- ls /var/run/secrets/kubernetes.io/serviceaccount          # ca.crt namespace token

# Call the API with the pod's own projected token
k -n dev exec -it deploy/web -- sh
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  API="https://kubernetes.default.svc"
  curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" $API/apis/apps/v1/namespaces/dev/deployments | head   # DeploymentList
  curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" $API/api/v1/namespaces/dev/secrets                    # 403 ... "system:serviceaccount:dev:ci"
  exit
k auth can-i --list -n dev --as=system:serviceaccount:dev:ci
```

This is exactly what an in-cluster client (client-go, `kubectl` inside a pod, an operator) does. The 403 message tells you exactly which identity, verb, resource and namespace failed. On the exam, an error like that in a pod's logs is a troubleshooting hint that a binding is missing.

**Step 4: `can-i --list` (task 4).** Compare these three and note the differences:

```bash
k auth can-i --list -n dev --as=jane
k auth can-i --list -n dev --as=bob
k auth can-i --list -n dev --as=system:serviceaccount:dev:default   # a fresh SA: only the self-review/discovery basics
```

**Step 5: kubeconfig (task 5), then a real user (task 6, optional but do it).** Run the full CSR flow from section 5 for jane (`O=developers`), then:

```bash
k config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true
k config set-context jane --cluster=kubernetes --user=jane --namespace=dev
k config use-context jane
k auth whoami                       # Username jane, Groups [developers system:authenticated]
k get pods                          # works (dev, via pod-reader)
k get deploy                        # Forbidden
k config view --minify              # only jane's bits
k config use-context admin          # BACK. Check with: k config current-context
```

Then grant the group instead of the user and watch it apply through the cert's `O`:

```bash
k create rolebinding dev-view -n dev --clusterrole=view --group=developers
k --context=jane get deploy         # now works: jane is in group developers
```

**Step 6: the five-task drill (acceptance criterion 1).** Clean up (`k delete ns dev; k delete clusterrole infra-reader; k delete clusterrolebinding infra-reader-jane`), then do these timed, < 3 min each, verified with `can-i`, no notes. Switch context between each one: odd tasks from `admin`, even tasks start with `k config use-context admin` after checking `current-context`.

1. SA `monitor` in ns `ops` may `get,list,watch` pods and services in `ops` only.
2. User `alice` may `create,delete` secrets in ns `app`, but only the secret named `app-tls` for delete. (Hint: two rules → YAML.)
3. Group `auditors` may read everything readable (built-in `view`) in all namespaces.
4. SA `scaler` in `app` may scale deployments in `app` (`deployments/scale`, `patch`/`update`) and nothing else.
5. User `bob` may list nodes and `get` `/healthz`.

---

## 7. Traps that cost points

- **Wrong context.** Run the context line the question gives you, every time, even if you think you're already there.
- **`--serviceaccount=name`** instead of `--serviceaccount=ns:name`. kubectl rejects it, but the YAML route lets you forget `namespace:` on the subject, and then the binding never matches.
- **`--as=<saname>`** to test a ServiceAccount. That tests a user. Use `--as=system:serviceaccount:<ns>:<name>`.
- **Role in the wrong namespace.** `k create role` without `-n` lands in your context's default namespace. The RoleBinding and the Role must be in the namespace the task names.
- **Singular or misspelled resources in YAML.** `resources: ["pod"]` or `["deployment"]` is accepted silently and matches nothing. Imperative `k create role` validates resource names, which is another reason to use it.
- **Wrong apiGroup.** Deployments are `apps`, not `""`. Ingress and NetworkPolicy are `networking.k8s.io`, Jobs/CronJobs are `batch`. Imperative creation fills this in for you.
- **Forgetting subresources.** "may view logs" needs `pods/log`. "may exec" needs `pods/exec` with `create`. "may scale" needs `deployments/scale`.
- **Trying to edit `roleRef`.** It's immutable. Delete and recreate the binding.
- **Using Role for cluster-scoped resources.** Nodes, PVs, namespaces and StorageClasses need a ClusterRole + ClusterRoleBinding.
- **Changing a running pod's ServiceAccount.** Not possible. Change the Deployment (rollout), or recreate the pod.
- **Not verifying.** `k auth can-i` takes five seconds. Check one allowed and one denied action per task.
- **Leaving your context on the test user.** The next question then fails with Forbidden, and you lose minutes working out why.
- **base64 with line wraps** in a CSR's `spec.request`. Use `base64 -w0`.

---

## 8. Self-check: answer before you open the lab

1. A ClusterRole `infra-reader` grants `list` on `pods` and `nodes`. You bind it to bob with a RoleBinding in `dev`. What can bob do, and where?
2. What username and groups does the API server see for the ServiceAccount `ci` in namespace `dev`? How do you test its permissions with `can-i`?
3. You need to let user jane read pod logs in `dev`. What exact `--resource` value(s) does the Role need, and what verb?
4. Why does the built-in `view` ClusterRole not include Secrets, while `edit` does?
5. You made a RoleBinding with the wrong `roleRef`. How do you fix it?
6. Where does a username come from for an x509 client cert, and where do its groups come from? What signs the cert in the CSR flow?
7. `k create role r --verb=get --resource=nodes -n dev` succeeds. Will a RoleBinding to it let jane `get nodes`? Why or why not?
8. What does `k config set-context --current --namespace=dev` change, and what does it not change?

<details>
<summary>Answers</summary>

1. bob can list pods **in `dev` only**. The `nodes` rule grants nothing: nodes are cluster-scoped, and a RoleBinding only grants permissions within its own namespace. Listing pods in other namespaces or listing nodes needs a ClusterRoleBinding.
2. Username `system:serviceaccount:dev:ci`, groups `system:serviceaccounts`, `system:serviceaccounts:dev` and `system:authenticated`. Test with `k auth can-i <verb> <resource> -n dev --as=system:serviceaccount:dev:ci`.
3. `--resource=pods,pods/log` with `--verb=get` (add `list` if she has to find the pod names first, which she usually does: `--verb=get,list`). `kubectl logs` is a `get` on the `pods/log` subresource.
4. Reading Secrets includes reading ServiceAccount token Secrets, which would let a read-only user act as more powerful identities. That's privilege escalation, so `view` leaves Secrets out. `edit` is meant for people who already run workloads, which can read Secrets anyway by mounting them in a pod.
5. `roleRef` is immutable. Delete the binding and create it again with the right role (`k delete rolebinding X -n ns` then `k create rolebinding ...`).
6. Username = the cert subject's `CN`, groups = its `O` fields. In the CSR flow, kube-controller-manager's csrsigning controller signs it with the cluster CA (`/etc/kubernetes/pki/ca.crt` + `ca.key`) after you approve it, for signer `kubernetes.io/kube-apiserver-client`.
7. No. A Role is namespaced and can only grant access to namespaced resources in its namespace. Rules about `nodes` in a Role match nothing. Use a ClusterRole + ClusterRoleBinding.
8. It sets the default namespace on the **current context** in the kubeconfig. It doesn't change the cluster, the user or credentials, or which context is current, and `-n` on a command still overrides it.

</details>

---

**Done when:** every [Day 4 acceptance criterion](../stories/day-04-rbac.md#acceptance-criteria) passes: five RBAC tasks under 3 minutes each verified with `auth can-i`, the four objects explained from memory, and zero wrong-context runs. Tomorrow (Day 5, Helm, Kustomize, CRDs and operators) builds on today: installing a chart creates its own ServiceAccounts and ClusterRoles, and CRD-based resources need RBAC rules with their own API group (and often an `aggregate-to-*` label) before anyone but admin can use them.
