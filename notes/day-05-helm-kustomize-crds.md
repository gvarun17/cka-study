# Day 5 Lesson: Helm, Kustomize, CRDs and operators

**Story:** [day-05-helm-kustomize-crds](../stories/day-05-helm-kustomize-crds.md) · **Read time:** ~35 min · **Then:** 2 hours in the lab

**Why this matters:** The Feb 2025 curriculum added three lines to Cluster Architecture, Installation & Configuration (25%): *"Use Helm and Kustomize to install cluster components"*, *"Understand extension interfaces (CNI, CSI, CRI)"* and *"Understand CRDs, install and configure operators"*. In practice this means one or two tasks like "install chart X from repo Y into namespace Z with these two values", "apply the kustomize overlay in /opt/app and change the image", or "the operator is installed. Create a `Foo` object with these fields". None of them need deep knowledge. They need you to **know where to look**: `helm show values`, `k kustomize`, `k explain`. Each is a 3–5 minute task if you know the flags and a 15-minute hole if you don't.

On EKS you've likely used Helm through Terraform, Argo CD or Flux, and EKS add-ons through the console. Today you drive the CLI by hand. The exam has no GitOps controller to lean on, and `helm` is the only tool that knows about releases.

> **Version note (checked today).** **Helm 4** is current (4.0 shipped Nov 2025, the helm.sh docs are on 4.3.x). For the exam commands nothing important changed: `repo add`, `install`, `upgrade`, `rollback`, `history`, `uninstall`, `show values` and `template` work the same. The differences you might see: **server-side apply is on by default** for new installs (`--server-side=auto|true|false`), `--wait` now takes a strategy (bare `--wait` = `watcher`), `--force` is now `--force-replace`, and `--atomic` has been renamed to `--rollback-on-failure`. Run `helm version` in the exam terminal first and check `helm upgrade --help` if a flag errors. **Kustomize:** `kubectl kustomize` / `-k` embeds kustomize v5. **`commonLabels` is deprecated since kustomize v5.0** in favour of `labels:` (it still works in `v1beta1`, and the story uses it, so know both). **cert-manager** is v1.21.x and recommends its OCI chart `oci://quay.io/jetstack/charts/cert-manager` with `--set crds.enabled=true` (the old `installCRDs=true` key is gone from the docs).

---

## 1. Three ways to put YAML in a cluster

| | Helm | Kustomize | Operator (CRD + controller) |
|---|---|---|---|
| What it is | Template engine + **release** tracker | Overlay/patch engine, no templates | An API extension + a pod that reconciles it |
| Input | Chart (templates + `values.yaml`) | `kustomization.yaml` + plain YAML | A custom resource you `k apply` |
| State | Release Secret in the target ns (`sh.helm.release.v1.<name>.v<rev>`) | None. It's `kubectl apply` | The CR's `.status`, written by the controller |
| Upgrade / rollback | `helm upgrade` / `helm rollback <rev>` | Edit + re-apply (git is your history) | Edit the CR. Controller converges |
| Exam command | `helm` | `k kustomize`, `k apply -k` | `k get crd`, `k explain`, `k apply -f` |

Helm and Kustomize are both **client-side**. They render YAML and send it to the API server like `kubectl apply` would. The API server has no idea Helm exists. The only server-side trace of a Helm release is that Secret, which is why `helm list` needs the right `-n`.

Operators are different: they're **server-side**. A CRD teaches the API server a new type, and a controller (usually installed by Helm) watches objects of that type and makes reality match. That's the same control loop as kube-controller-manager's Deployment controller, run by someone else.

---

## 2. Helm: how it works

```
 helm repo add / OCI registry
          │
          ▼
   chart (tgz)  ─────────►  templates/*.yaml + values.yaml (defaults)
                                     │
   your values  ─ -f file.yaml ─────►│  merge order (later wins):
                  --set k=v ────────►│  chart defaults < -f a.yaml < -f b.yaml < --set
                                     ▼
                           rendered manifests ──► kube-apiserver
                                     │
                                     ▼
                Secret sh.helm.release.v1.<release>.v<N>  (type helm.sh/release.v1, in -n ns)
```

- **Repo vs chart vs release:** `helm repo add bitnami URL` registers a repo locally (`~/.config/helm/repositories.yaml`). `bitnami/nginx` is a chart in it. `helm install web bitnami/nginx` creates a **release** called `web`. One chart can have many releases.
- **OCI charts** skip `repo add`: `helm install x oci://registry/path/chart --version 1.2.3`. `helm search repo` doesn't see them.
- **Revisions:** every `install`, `upgrade` and `rollback` writes a new revision. A rollback to revision 1 creates revision 3 with revision 1's content. It doesn't delete history.
- **CRDs in charts:** files in a chart's `crds/` directory are installed on first install and **never upgraded or deleted** by Helm. Charts that need managed CRDs (cert-manager) template them instead, behind a value like `crds.enabled`.

### Exam commands

```bash
helm version                                         # 4.x expected; flags below are the same in v3
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update                                     # ALWAYS after add, or search shows stale versions
helm repo list
helm search repo podinfo                             # latest chart in added repos
helm search repo podinfo --versions | head           # all versions: use when the task pins one
helm search hub nginx                                # Artifact Hub (needs internet; rarely useful on exam)

# Find the right key BEFORE installing
helm show values podinfo/podinfo | less              # / to search in less. Or:
helm show values podinfo/podinfo | grep -n -A3 -i replica
helm show chart podinfo/podinfo                      # Chart.yaml: version vs appVersion

# Install
helm install web podinfo/podinfo -n apps --create-namespace \
  --set replicaCount=2 --set ui.message="hello cka" --version <CHART VERSION from search --versions>
helm install web podinfo/podinfo -n apps -f my-values.yaml
helm install web podinfo/podinfo -n apps --dry-run=client          # validate without installing

# Preview manifests (no cluster needed)
helm template web podinfo/podinfo --set replicaCount=2 | less
helm template web podinfo/podinfo -s templates/deployment.yaml     # one template only

# Inspect
helm list -A                                         # all releases, all namespaces
helm list -n apps -a                                 # include failed/pending
helm status web -n apps
helm get values web -n apps                          # only YOUR overrides
helm get values web -n apps --all                    # merged, everything
helm get manifest web -n apps                        # what was actually applied
helm history web -n apps

# Change
helm upgrade web podinfo/podinfo -n apps --reuse-values --set ui.color="#ff0000"
helm upgrade --install web podinfo/podinfo -n apps -f my-values.yaml   # idempotent
helm rollback web 1 -n apps                          # to revision 1 (omit number = previous)
helm uninstall web -n apps                           # --keep-history to keep the record
```

**`--reuse-values` matters.** `helm upgrade` *without* it resets to chart defaults plus whatever you pass **this time**. Your earlier `--set replicaCount=2` silently goes back to 1. Either pass `--reuse-values` or repeat every value (or keep them in a `-f` file, which is the cleanest).

**`--set` syntax you'll need:**

| Want | `--set` |
|---|---|
| Nested key | `--set service.type=NodePort` |
| String that looks like a number/bool | `--set-string image.tag=1.10` |
| List item | `--set 'tolerations[0].key=dedicated'` |
| Key containing a dot | `--set 'podAnnotations.prometheus\.io/scrape=true'` |
| Several | `--set a=1,b=2` or repeat `--set` |

Equivalent `values.yaml`:

```yaml
replicaCount: 2
ui:
  message: "hello cka"
  color: "#ff0000"
service:
  type: NodePort
```

---

## 3. Kustomize: how it works

No templates, no variables. You take plain YAML (the **base**) and describe transformations in a `kustomization.yaml`. An **overlay** is just another kustomization whose `resources:` points at the base and adds changes.

```
app/
├── base/
│   ├── kustomization.yaml     resources: [deployment.yaml, service.yaml]
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    └── prod/
        ├── kustomization.yaml resources: [../../base] + namePrefix, labels, images, replicas, patches
        └── replica-patch.yaml (optional, if you patch from a file)

 k kustomize app/overlays/prod     →  prints rendered YAML (build only)
 k apply -k  app/overlays/prod     →  build + apply
 k delete -k app/overlays/prod     →  delete everything it renders
```

Order of operations doesn't matter in the file. Kustomize runs its built-in transformers (prefix/suffix, labels, images, replicas, patches) and fixes up references: if `namePrefix: prod-` renames a ConfigMap, Deployments that mount it get the new name too. Same for Service selectors when you use `includeSelectors`.

### Complete base + overlay

`app/base/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.27
        ports:
        - containerPort: 80
```

`app/base/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

`app/base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

`app/overlays/prod/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod                 # sets metadata.namespace on everything (ns must exist, or add a Namespace resource)
resources:
- ../../base
namePrefix: prod-
labels:                         # modern form
- pairs:
    env: prod
  includeSelectors: true        # also adds to selectors/matchLabels (= old commonLabels behaviour)
# commonLabels:                 # deprecated form, same effect as includeSelectors: true
#   env: prod
images:
- name: nginx                   # match on the image NAME in the base, not the container name
  newTag: "1.28"
  # newName: registry.local/nginx
replicas:
- name: web                     # the base name, before namePrefix
  count: 3
patches:
- target:
    kind: Deployment
    name: web
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/resources
      value:
        requests: {cpu: 50m, memory: 64Mi}
- patch: |-                      # strategic-merge patch, inline
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web
    spec:
      template:
        spec:
          containers:
          - name: web
            env:
            - name: TIER
              value: prod
```

**Fast route without vim** (run inside the overlay dir; uses the standalone `kustomize` binary, which the exam may not have, so be able to type the YAML too):

```bash
kustomize edit set image nginx=nginx:1.28
kustomize edit set replicas web=3
kustomize edit set nameprefix prod-
kustomize edit fix                 # converts commonLabels → labels
```

**`commonLabels` vs `labels`:** `commonLabels` always adds the label to selectors too. On an **existing** Deployment that changes `spec.selector`, which is immutable, so `k apply -k` fails with `field is immutable`. With `labels:` you choose: `includeSelectors: false` (default) labels metadata only; `includeTemplates: true` also labels pod templates.

Other generators you may meet:

```yaml
configMapGenerator:
- name: app-config
  literals: [MODE=prod]
  files: [app.properties]
secretGenerator:
- name: db
  literals: [password=s3cr3t]
# names get a content hash suffix (app-config-7g2k...), and references are rewritten.
# generatorOptions: {disableNameSuffixHash: true} turns that off.
```

---

## 4. CRDs and operators: how they work

```
  CustomResourceDefinition  certificates.cert-manager.io
  ├── group: cert-manager.io     ├── names: kind Certificate, plural certificates, shortNames [cert, certs]
  ├── scope: Namespaced          └── versions: [v1 (served, storage)] + openAPIV3Schema
  │
  │  k apply -f crd.yaml  →  apiserver starts serving  /apis/cert-manager.io/v1/namespaces/*/certificates
  ▼
  Certificate "web-tls" (a CR)   ── stored in etcd like any object, validated against the schema
           ▲   │ watch
   status  │   ▼
  cert-manager controller (Deployment in ns cert-manager)  ── creates Secret web-tls, updates .status.conditions
```

- A **CRD alone** is just storage + validation. You can create CRs and nothing happens. The **controller** is what makes it an operator. If a CR "does nothing", check the controller's pod and logs: `k -n <op-ns> logs deploy/<controller>`.
- **The CRD name is `<plural>.<group>`**, e.g. `certificates.cert-manager.io`. That's what `k get crd` lists and what you `k describe crd`.
- **`k explain` works on CRs** when the CRD has a structural schema (all modern ones do). This is your documentation in the exam: the operator's website isn't an allowed reference.
- **RBAC** (yesterday): CRs are just resources in their own API group. A Role needs `apiGroups: ["cert-manager.io"]`, `resources: ["certificates"]`. Charts often ship ClusterRoles labelled `rbac.authorization.k8s.io/aggregate-to-edit: "true"` so the built-in `edit` role picks them up.
- **Deleting a CRD deletes every CR of that type.** Don't do it to "clean up" during the exam unless the task says so.

### Exam commands

```bash
k get crd                                            # NAME = plural.group
k get crd | grep -i cert
k api-resources --api-group=cert-manager.io          # KIND, SHORTNAMES, APIVERSION, NAMESPACED
k api-resources | grep -i issuer
k explain certificate                                # resolves via discovery
k explain certificate.spec                           # one level
k explain certificate.spec --recursive | less        # whole tree
k explain certificate.spec.issuerRef
k explain certificates --api-version=cert-manager.io/v1   # if two groups have the same kind
k get certificates -A                                # or the short name: k get cert -A
k describe crd certificates.cert-manager.io | less   # versions, schema, conditions
k get crd certificates.cert-manager.io -o jsonpath='{.spec.versions[*].name}{"\n"}'

# minimal CRD, to understand the shape (rarely written by hand on the exam)
cat <<'EOF' | k apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.ops.example.com          # MUST be <plural>.<group>
spec:
  group: ops.example.com
  scope: Namespaced
  names: {kind: Backup, plural: backups, singular: backup, shortNames: [bk]}
  versions:
  - name: v1
    served: true
    storage: true                         # exactly one version is the storage version
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [schedule]
            properties:
              schedule: {type: string}
              retention: {type: integer, minimum: 1}
EOF
k explain backup.spec
```

---

## 5. Hands-on walkthrough on cp1 + worker1

Do everything from cp1 as admin. Run your context check first (`k config current-context`), Day 4 habit.

**Step 0: install Helm on cp1** (the exam terminal already has it):

```bash
HV=$(curl -s https://api.github.com/repos/helm/helm/releases/latest | grep -oP '"tag_name": "\K[^"]+')
curl -sL https://get.helm.sh/helm-${HV}-linux-amd64.tar.gz | tar xz
sudo mv linux-amd64/helm /usr/local/bin/ && helm version
source <(helm completion bash)    # add to ~/.bashrc too
```

**Step 1: Helm lifecycle (task 1, acceptance criterion 1).** ⏱ Target < 5 min from `repo add` to rolled back.

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo && helm repo update
helm show values podinfo/podinfo | grep -n -E '^replicaCount|^ui:|message|color'
helm install web podinfo/podinfo -n apps --create-namespace --set replicaCount=2 --set ui.message="rev1"
helm list -n apps ; k -n apps get deploy,svc,pod
helm get values web -n apps                          # replicaCount: 2, ui.message: rev1

helm upgrade web podinfo/podinfo -n apps --reuse-values --set ui.message="rev2"
helm history web -n apps                             # 1 superseded, 2 deployed
k -n apps exec deploy/web-podinfo -- wget -qO- localhost:9898 | grep message

helm rollback web 1 -n apps
helm history web -n apps                             # 3 deployed "Rollback to 1"
helm get values web -n apps                          # rev1 again
k -n apps get secret -l owner=helm                   # sh.helm.release.v1.web.v1..v3
```

Now see the `--reuse-values` trap with your own eyes:

```bash
helm upgrade web podinfo/podinfo -n apps --set ui.message="oops"
k -n apps get deploy web-podinfo -o jsonpath='{.spec.replicas}{"\n"}'   # 1: replicaCount=2 is gone
helm rollback web 3 -n apps
```

**Step 2: `show values` → `template` (task 2).**

```bash
helm show values podinfo/podinfo > /tmp/pi-values.yaml
vim /tmp/pi-values.yaml                              # set replicaCount: 3, service.type: NodePort
helm template web podinfo/podinfo -f /tmp/pi-values.yaml | grep -E 'kind:|replicas:|type:'
helm template web podinfo/podinfo -f /tmp/pi-values.yaml > /tmp/web.yaml   # some tasks want the rendered file
```

**Step 3: Kustomize overlay (task 3, acceptance criterion 2).** ⏱ Target < 5 min. Type the five files from section 3 into `~/app` (no copy-paste), then:

```bash
k create ns prod
k kustomize ~/app/overlays/prod | less               # check: prod-web, env: prod in selector, nginx:1.28, replicas 3
k apply -k ~/app/overlays/prod
k -n prod get deploy prod-web -o wide                # 3/3, IMAGES nginx:1.28
k -n prod get deploy prod-web -o jsonpath='{.spec.selector.matchLabels}{"\n"}'
k -n prod get svc prod-web -o jsonpath='{.spec.selector}{"\n"}'          # both have env=prod
```

Then change the tag to `1.27` and replicas to `2`, re-apply, and watch the rollout. Finally flip `includeSelectors: true` → `false`, re-apply, and read the `field is immutable` error. That's the trap from section 3. Fix it with `k delete -k ~/app/overlays/prod && k apply -k ~/app/overlays/prod`.

**Step 4: CRDs in a stock cluster (task 5).** Your cluster already has CRDs: Calico's.

```bash
k get crd | head                                     # *.crd.projectcalico.org, maybe *.operator.tigera.io
k api-resources | grep -i calico
k explain ippools.crd.projectcalico.org.spec 2>/dev/null || k explain ippool.spec
```

**Step 5: install an operator and make a CR (task 4, acceptance criterion 3).**

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  -n cert-manager --create-namespace --set crds.enabled=true
k -n cert-manager rollout status deploy/cert-manager-webhook   # wait: CRs fail until the webhook is up
k get crd | grep cert-manager
k api-resources --api-group=cert-manager.io
```

⏱ Now the criterion: < 4 min, using only `k explain`. Build a self-signed `ClusterIssuer` and a `Certificate` for `web.prod.svc` in `prod`:

```bash
k explain clusterissuer.spec | grep -A2 selfSigned
k explain certificate.spec | grep -E '^  [a-zA-Z]'   # field list
k explain certificate.spec.issuerRef
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: web-tls
  namespace: prod
spec:
  secretName: web-tls            # the controller writes the key pair here
  dnsNames:
  - web.prod.svc
  duration: 2160h
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer          # default is Issuer (namespaced): forget this and it stays not Ready
```

```bash
k apply -f cert.yaml
k -n prod get certificate web-tls                    # READY True
k -n prod get secret web-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -ext subjectAltName
```

Break it on purpose: delete `kind: ClusterIssuer` and re-apply as `web-tls2`. Run `k describe certificate web-tls2 -n prod` and read the events. That's the operator troubleshooting loop: **CR status/events → controller logs**.

**Step 6: RBAC tie-in (5 min, links Day 4).**

```bash
k auth can-i create certificates.cert-manager.io -n prod --as=jane
k get clusterrole | grep cert-manager | grep -i edit
k get clusterrole cert-manager-edit -o jsonpath='{.metadata.labels}{"\n"}'   # aggregate-to-edit: "true"
```

**Cleanup** (keep cert-manager, it's useful for Day 9 Ingress/Gateway TLS): `helm uninstall web -n apps; k delete -k ~/app/overlays/prod`.

---

## 6. Traps that cost points

- **No `helm repo update`** after `repo add` or before `search`. You install an old chart version, or the chart "isn't found".
- **Wrong `-n` on helm.** `helm list` without `-n` shows only the current context's namespace. Use `helm list -A`. `helm upgrade`/`rollback`/`uninstall` without `-n` say "release not found".
- **Forgetting `--create-namespace`** when the task's namespace doesn't exist. Install fails.
- **`--version` is the chart version**, not the app version. `helm search repo X --versions` shows both columns. Tasks that say "version 1.2.3 of the chart" mean `--version`.
- **Upgrade without `--reuse-values`** silently resets earlier `--set` values. Verify with `helm get values`.
- **Guessing value keys.** `--set replicas=3` when the chart uses `replicaCount` is accepted silently and changes nothing. Always `helm show values | grep`.
- **Task asks for manifests in a file, you installed instead.** "Generate the manifests" = `helm template ... > file`. "Install" = `helm install`. Read the verb.
- **Kustomize `images[].name` is the image name** (`nginx`), not the container name (`web`).
- **`replicas[].name` and patch targets use the base name**, before `namePrefix`.
- **`commonLabels` / `includeSelectors: true` on a live Deployment** changes the immutable selector. Apply fails.
- **`k apply -f dir/` instead of `-k dir/`** on a kustomize directory. It applies the raw files and errors on `kustomization.yaml`.
- **Relative paths in `resources:`** are relative to the kustomization file, not your shell.
- **CR created before the operator's webhook is ready** → `failed calling webhook ... connection refused`. Wait for `rollout status`, then retry.
- **Wrong `apiVersion` on a CR.** Use `k api-resources --api-group=<group>` and copy the APIVERSION column exactly.
- **Inventing fields.** CRD validation rejects unknown fields (`strict decoding error: unknown field`). `k explain` first.
- **Deleting a CRD** deletes all its objects cluster-wide.

---

## 7. Self-check: answer before you open the lab

1. Where does Helm store release state, and what does that mean for `helm list` with no flags?
2. You ran `helm install web repo/chart --set replicaCount=3`, then `helm upgrade web repo/chart --set image.tag=2.0`. How many replicas now, and how should you have run the upgrade?
3. What does `helm rollback web 1` do to the revision history?
4. A task says "install version 4.2.0 of chart `foo/bar`". The repo shows CHART VERSION 4.2.0 / APP VERSION 1.9. Which flag and value?
5. In a kustomize overlay, you want every pod of the base Deployment `web` (container `app`, image `registry/web:1.0`) to run `registry/web:1.1`. Write the `images:` entry.
6. Why might `k apply -k` fail on an existing Deployment after you add `commonLabels`, and how does `labels:` avoid it?
7. A Certificate CR has been "not Ready" for 5 minutes. What two places do you look, in order?
8. You need to know which fields `spec` of a CRD-based kind `Backup` accepts, and its exact `apiVersion`. Which two commands?

<details>
<summary>Answers</summary>

1. In Secrets named `sh.helm.release.v1.<release>.v<rev>` (type `helm.sh/release.v1`) in the release's namespace. `helm list` only looks in the current context's namespace, so a release in another namespace is invisible unless you pass `-n <ns>` or `-A`.
2. Back to the chart default (usually 1). An upgrade without `--reuse-values` starts from chart defaults plus only the values passed in that command. Use `helm upgrade web repo/chart --reuse-values --set image.tag=2.0`, or keep all values in a `-f values.yaml` and pass it every time.
3. It creates a **new** revision (e.g. 3) with revision 1's chart and values. Revisions 1 and 2 stay in history (up to `--history-max`). `helm history` shows "Rollback to 1".
4. `--version 4.2.0`. `--version` selects the chart version. The app version is informational (usually the default image tag).
5. `images: [{name: registry/web, newTag: "1.1"}]`. Match on the image name as it appears in the base, never the container name.
6. `commonLabels` also adds the label to `spec.selector.matchLabels`, and a Deployment's selector is immutable, so the apply is rejected. With `labels:` and `includeSelectors: false` (the default), only resource metadata is labelled and selectors are untouched (add `includeTemplates: true` to label pod templates too).
7. First the CR itself: `k describe certificate <name> -n <ns>` (status conditions and events, e.g. issuer not found or wrong `kind`). Then the controller: `k -n cert-manager logs deploy/cert-manager`. The CR is only data; the controller does the work.
8. `k api-resources | grep -i backup` (or `--api-group=<group>`) for the group/version, shortnames and scope; `k explain backup.spec` (add `--recursive` for the whole tree) for the fields.

</details>

---

**Done when:** every [Day 5 acceptance criterion](../stories/day-05-helm-kustomize-crds.md#acceptance-criteria) passes: chart installed with 2 overridden values, upgraded and rolled back in < 5 min; overlay changing image tag and replicas built and applied in < 5 min; a valid CR created from `k explain` alone in < 4 min. Tomorrow (Day 6, Deployments, config and resources) goes inside what today's charts and overlays render: rollouts and `rollout undo` (Helm's rollback is a layer above that), ConfigMaps and Secrets (what `configMapGenerator` produces), and requests/limits (what you just patched in with kustomize).
