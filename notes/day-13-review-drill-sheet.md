# Day 13 Lesson: Review day, a rapid-fire drill sheet for Days 2–11

**Story:** [day-13-review](../stories/day-13-review.md) · **Read time:** ~25 min (sections 1–2), then the sheet is something you **do**, not read · **Then:** ~2.5 h of drills

**Why this matters:** Today covers every domain, weighted the way the exam is: Troubleshooting **30%**, Cluster Architecture **25%**, Services & Networking **20%**, Workloads & Scheduling **15%**, Storage **10%**. There's nothing new to learn. Two days out, a new topic is worth less than making the things you already know fast and automatic. Yesterday's mock showed where you lose points. Today you close those gaps, and you re-time the two heavy control-plane tasks (etcd restore, kubeadm upgrade), which are often the two highest-weighted questions on the paper.

> **Version notes (checked today).**
> - The [CNCF curriculum repo](https://github.com/cncf/curriculum) still lists `CKA_Curriculum_v1.37.pdf`, so the exam is on **Kubernetes v1.37** (same as Days 2–12). Your lab is on v1.35. Nothing on this sheet differs between them.
> - etcd **3.6** is what recent Kubernetes ships, and there `snapshot status` and `snapshot restore` are **`etcdutl`** commands. `snapshot save` stays in `etcdctl` ([etcd v3.6 announcement](https://etcd.io/blog/2025/announcing-etcd-3.6/)). On an exam node, run `etcdctl version` first. If it says 3.5, `etcdctl snapshot restore` also works.
> - Helm 4 renamed `--atomic` to `--rollback-on-failure` and `--force` to `--force-replace` (Day 5). If a flag errors, check `helm <cmd> --help`. Don't guess.

---

## 1. How today works

The story has four tasks. Here is the order, with time boxes that fit the 3 hours:

| Block | Time | What | Story task |
|---|---|---|---|
| A | 10 min | Fill in or tidy the miss log, then pick your 2 weakest domains (section 2) | 1, 2 |
| B | 25 min | **etcd backup + restore from memory**, timed, twice if the first run is over 8 min (section 3) | 3 |
| C | 30 min | **kubeadm control-plane + worker upgrade from memory**, timed, on the `v134` snapshot (section 3) | 3 |
| D | 45 min | Redo every killer.sh miss and slow question, **without** opening the solution | 1 |
| E | 30 min | The drill sheet (section 4) for your 2 weakest domains, then any card you can't answer in 10 seconds | 2 |
| F | 30 min | 3 fault injections from Day 11, random order (section 5) | 4 |
| G | 10 min | Walk the earlier stories' acceptance criteria and tick what passes (section 6) | AC 3 |

Rules for every block: **timer on, docs only (kubernetes.io/docs, helm.sh/docs), no notes, no lessons.** You'll check yourself against the answers afterward, not during.

### Redoing a killer.sh question properly

The killer.sh environment stays open for 36 hours after activation, so you can redo the questions in the same environment today. For each miss:

1. Read the question again. **Don't** look at the solution.
2. Before typing anything, say the approach in one sentence ("create a Role and a RoleBinding in `ns`, check with `auth can-i --as`").
3. Do it with the timer running. Verify it the way the grader would.
4. Only then compare with the solution. If yours is slower, type their route twice.
5. Update the miss-log row: **fixed** (from memory, under the cap) or **again** (redo it once more before Block G).

Session 2 tomorrow has the **same questions**, so don't memorise answers. Memorise the **method**. If you only remember that Q7's answer was `targetPort: 8080`, that's useless on Friday. If you remember "empty endpoints → compare selector with `--show-labels`, then `targetPort` with `containerPort`", it works on any question.

---

## 2. Picking your two weakest domains

Your story's Notes on [Day 12](../stories/day-12-mock-1.md#notes) are the input. If that table is thin or empty, spend 5 minutes filling it from the killer.sh score page now, one row per miss or slow question:

```
| Q# | Domain | What went wrong (specific) | Root cause | Day-13 result |
| 7  | Networking | NetPol egress, forgot DNS 53, 9 min | Knowledge gap | |
```

Then count misses **weighted by how often the domain appears**. A rough rule is to multiply each domain's misses by its exam weight:

| Domain | Weight | Your misses | × weight | Rank |
|---|---|---|---|---|
| Troubleshooting | 30 | | | |
| Cluster Architecture | 25 | | | |
| Services & Networking | 20 | | | |
| Workloads & Scheduling | 15 | | | |
| Storage | 10 | | | |

The top two are today's focus for Block E. If two domains tie, choose the one where your misses were **knowledge gaps** over the one where they were **misreads**. Misreads are fixed by habits, and those habits get drilled in every block anyway.

If the mock went well everywhere (no domain with more than one miss), use the **90-second test** instead: for each domain, take the first three cards in section 4 and answer them out loud. Any card that takes longer than 10 seconds marks a weak domain.

---

## 3. The two heavy tasks, from memory

These are the story's timed criteria: **etcd restore under 8 minutes, control-plane upgrade under 10.** Do each one cold first. Then compare with the reference below and repeat only the part where you hesitated.

### etcd backup and restore (Day 2)

Setup so you can prove the restore worked:

```bash
k create ns before-snap
# ... take the snapshot (below) ...
k create ns after-snap
# ... restore (below) ...
k get ns          # before-snap present, after-snap GONE = restore really happened
```

Reference (on cp1, `sudo -i`):

```bash
# 1. read the flags, don't remember them
grep -E -- '--(cert-file|key-file|trusted-ca-file|listen-client-urls|data-dir)' /etc/kubernetes/manifests/etcd.yaml

# 2. save
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
etcdutl snapshot status /opt/etcd-backup.db -w table

# 3. restore into a NEW directory
etcdutl snapshot restore /opt/etcd-backup.db --data-dir /var/lib/etcd-restore

# 4. point the static pod at it: change ONLY the hostPath of the etcd-data volume
cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
vim /etc/kubernetes/manifests/etcd.yaml        # volumes: - hostPath: path: /var/lib/etcd-restore

# 5. wait for the new etcd and apiserver
watch -n2 'crictl ps --name "etcd|kube-apiserver"'
k get ns
```

Where the time goes, so you know what to speed up:

| Step | Target | If you're slower |
|---|---|---|
| Find cert paths | 30 s | You're scrolling the manifest. Use the grep. |
| Save + status | 1.5 min | Typing the flags. Type them three times today. |
| Restore | 30 s | |
| Edit manifest | 1.5 min | You're editing `--data-dir` too. Only the hostPath needs to change, because the container still sees `/var/lib/etcd`. |
| Wait for recovery | 1–3 min | Normal. If it's over 3 min, check `crictl ps -a --name etcd` and its logs. Don't wait blindly. |

If the API server doesn't come back after 3 minutes: `crictl ps -a --name etcd` → `crictl logs <id>`. The usual mistakes are a typo in the path (etcd starts with an **empty** data dir, so the cluster looks wiped), or you edited the `--data-dir` flag **and** the hostPath so they don't match the mountPath anymore.

### kubeadm upgrade (Day 3)

Restore your **`v134`** VM snapshots (Day 3 step 0 / runbook 3.1) so you can do a real 1.34 → 1.35 upgrade again. Don't take the lab to 1.36: Ubuntu's containerd 1.7 isn't supported after 1.35 (Day 3 version note). On the exam the numbers will be 1.36 → 1.37. The sequence is identical.

Reference, control plane (cp1, `sudo -i`):

```bash
sed -i 's/v1.34/v1.35/' /etc/apt/sources.list.d/kubernetes.list   # check it first: cat the file
apt-get update && apt-cache madison kubeadm | head -3             # pick the exact patch
apt-mark unhold kubeadm && apt-get install -y kubeadm='1.35.x-*' && apt-mark hold kubeadm
kubeadm version
kubeadm upgrade plan
kubeadm upgrade apply v1.35.x -y
k drain cp1 --ignore-daemonsets
apt-mark unhold kubelet kubectl && apt-get install -y kubelet='1.35.x-*' kubectl='1.35.x-*' && apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
k uncordon cp1
```

Worker (worker1): same repo edit and kubeadm install, then **`kubeadm upgrade node`** (not `plan`/`apply`). Drain from cp1 (`k drain worker1 --ignore-daemonsets --delete-emptydir-data`), install kubelet + kubectl, restart kubelet, uncordon from cp1.

Done means all four of these are true. Check them in this order:

```bash
kubeadm version -o short          # on each node: v1.35.x
k get nodes                       # both nodes v1.35.x (this column is the KUBELET), both Ready
k get nodes | grep -c SchedulingDisabled     # 0, nothing left cordoned
k get pods -n kube-system         # all Running
```

The most common half-finish is "`apply` succeeded, kubelet not upgraded". `k get nodes` still shows the old version, and the grader marks it wrong. The second most common is forgetting `uncordon`.

---

## 4. The drill sheet

How to use it: read the card, answer **out loud or in a scratch terminal**, then open the answers. Mark each card ✔ (under 10 s and right), ~ (right but slow), or ✘. Redo every ~ and ✘ at the end of Block E. Anything that's a command, type it on the lab. Don't just say it.

### Day 2: static pods, PKI, etcd

1. Where are the static pod manifests, and what decides that path?
2. Which cert does the API server present to clients, and which one does it use to talk to etcd?
3. Name the four etcd flags you need for `snapshot save`.
4. You restored a snapshot, but the cluster has no workloads at all, not even the old ones. What did you probably get wrong?
5. How do you check when the cluster certificates expire?
6. `kubectl` is down. Where do you read the API server's logs? (Two places.)

<details>
<summary>Day 2 answers</summary>

1. `/etc/kubernetes/manifests`. It's set by `staticPodPath` in `/var/lib/kubelet/config.yaml`.
2. Serving: `apiserver.crt` / `apiserver.key` (`--tls-cert-file`, `--tls-private-key-file`). To etcd: `apiserver-etcd-client.crt` / `.key` (`--etcd-certfile`, `--etcd-keyfile`) with `--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt`.
3. `--endpoints`, `--cacert`, `--cert`, `--key`. Read their values from `etcd.yaml` (`--trusted-ca-file`, `--cert-file`, `--key-file`, `--listen-client-urls`).
4. The hostPath doesn't point at the restored directory (typo or wrong volume), so etcd started on an empty data dir. Compare the `etcd-data` volume's `hostPath.path` with what you passed to `--data-dir` during the restore.
5. `kubeadm certs check-expiration`. Renew with `kubeadm certs renew all`, then restart the static pods (move the manifests out and back, or restart the kubelet).
6. `crictl ps -a --name kube-apiserver` + `crictl logs <id>`, or `/var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log`. If there's no container at all: `journalctl -u kubelet`.

</details>

### Day 3: kubeadm upgrade

1. Order of components and nodes for an upgrade?
2. Why does `apt-get install kubeadm=1.37.1-*` fail with "version not found" on a 1.36 node?
3. What's the difference between `kubeadm upgrade apply` and `kubeadm upgrade node`?
4. Which `drain` flags will you almost always need?
5. `k get nodes` still shows the old version after `upgrade apply` succeeded. Why?

<details>
<summary>Day 3 answers</summary>

1. Control plane node first (kubeadm → `upgrade apply` → drain → kubelet + kubectl → restart → uncordon), then each worker (kubeadm → `upgrade node` → drain → kubelet + kubectl → restart → uncordon). One minor version at a time.
2. pkgs.k8s.io has **one repo per minor**. Edit `/etc/apt/sources.list.d/kubernetes.list` from `v1.36` to `v1.37`, then `apt-get update`.
3. `apply` upgrades the control plane: it rewrites the static pod manifests, upgrades CoreDNS and kube-proxy, and renews certs. It runs once, on the first control-plane node. `node` upgrades that node's local kubelet config (and the static pods on extra control-plane nodes). Run it on every other node.
4. `--ignore-daemonsets`, and often `--delete-emptydir-data`. Add `--force` only for bare pods with no controller, and only when the task allows it.
5. That column is the **kubelet** version. You haven't upgraded and restarted the kubelet yet.

</details>

### Day 4: RBAC and kubeconfig

1. One-liner: ServiceAccount `bot` in `ci` that can `get,list,watch` pods in `ci`, plus the check.
2. Same permissions, but in **every** namespace. Which objects change?
3. A ClusterRole bound with a RoleBinding. What does that grant?
4. How do you change the `roleRef` of an existing binding?
5. Add a context `jane@dev` for user `jane` on cluster `kubernetes` in namespace `dev`.
6. CSR flow for user `jane` in group `dev`: the four steps.

<details>
<summary>Day 4 answers</summary>

1. `k create sa bot -n ci; k create role pod-reader --verb=get,list,watch --resource=pods -n ci; k create rolebinding bot-pods --role=pod-reader --serviceaccount=ci:bot -n ci; k auth can-i list pods --as=system:serviceaccount:ci:bot -n ci` → `yes`. Check a negative too: `-n default` → `no`.
2. `k create clusterrole` + `k create clusterrolebinding` (`--serviceaccount=ci:bot`).
3. The ClusterRole's rules, but **only in the RoleBinding's namespace**. This is how you reuse one role definition across namespaces.
4. You can't, because `roleRef` is immutable. Delete the binding and create it again.
5. `k config set-context jane@dev --cluster=kubernetes --user=jane --namespace=dev` (the user needs `k config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true` first).
6. `openssl genrsa -out jane.key 2048; openssl req -new -key jane.key -subj "/CN=jane/O=dev" -out jane.csr` → a `CertificateSigningRequest` with `request: $(base64 -w0 jane.csr)`, `signerName: kubernetes.io/kube-apiserver-client`, `usages: [client auth]` → `k certificate approve jane` → `k get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt`.

</details>

### Day 5: Helm, Kustomize, CRDs

1. Install chart `bitnami/nginx` as release `web` into a new namespace `web`, with `replicaCount=2`.
2. Which values does a chart take, and what values does a running release use?
3. Roll release `web` back to revision 1.
4. Render a kustomize overlay without applying it, then apply it.
5. Change the image tag in a kustomization without the `kustomize` binary.
6. An operator added a CRD. How do you find its kind and the fields its spec takes?

<details>
<summary>Day 5 answers</summary>

1. `helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update && helm install web bitnami/nginx -n web --create-namespace --set replicaCount=2`. Verify: `helm list -n web`, `k get deploy -n web`.
2. `helm show values <chart>` (defaults). `helm get values web -n web` (what you set). Add `-a` for all computed values.
3. `helm history web -n web` → `helm rollback web 1 -n web`.
4. `k kustomize <dir>`, then `k apply -k <dir>`.
5. In `kustomization.yaml`: `images: [{name: nginx, newTag: "1.27"}]` (`name` matches the image name in the base, without the tag). Then `k apply -k`.
6. `k get crd | grep <word>`, `k api-resources --api-group=<group>` for the kind and short names, then `k explain <kind>.spec --recursive` (or `k get crd <name> -o yaml` for the schema).

</details>

### Day 6: workloads, config, resources

1. Roll deployment `api` back to the previous version, then to revision 3.
2. Mount ConfigMap `app-cfg` at `/etc/app` in a Deployment. Which YAML keys?
3. Env var `DB_PASS` from key `password` of Secret `db`. Which YAML keys?
4. Requests 100m/64Mi, limits 200m/128Mi on all containers of a deployment, fastest route.
5. QoS classes: how is each one assigned?
6. HPA min 2, max 5, target 50% CPU. What must be true of the pods?
7. A CronJob every 15 minutes that keeps 2 successful jobs.

<details>
<summary>Day 6 answers</summary>

1. `k rollout undo deploy/api`; `k rollout history deploy/api` → `k rollout undo deploy/api --to-revision=3`. Verify with `k rollout status`.
2. `spec.template.spec.volumes: [{name: cfg, configMap: {name: app-cfg}}]` and on the container `volumeMounts: [{name: cfg, mountPath: /etc/app}]`.
3. `env: [{name: DB_PASS, valueFrom: {secretKeyRef: {name: db, key: password}}}]`.
4. `k set resources deploy/api --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi`.
5. **Guaranteed**: every container has cpu and memory limits and requests equal to the limits. **Burstable**: not Guaranteed, but at least one request or limit is set. **BestEffort**: nothing set anywhere. QoS is fixed when the pod is created.
6. `k autoscale deploy web --min=2 --max=5 --cpu=50%` (or `--cpu-percent=50`). The pods need a **CPU request**, and metrics-server has to be running. Without them, TARGETS shows `<unknown>`.
7. `k create cronjob tick --image=busybox:1.36 --schedule="*/15 * * * *" $do -- date > cj.yaml`, add `successfulJobsHistoryLimit: 2` under `spec`, apply.

</details>

### Day 7: scheduling and multi-container pods

1. A pod is Pending with `0/2 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 1 Insufficient memory`. What do you do?
2. Toleration that lets a pod run on cp1.
3. Only on nodes labeled `disk=ssd`, two ways. Which one is the hard requirement with operators?
4. One replica per node for a Deployment.
5. Sidecar that tails `/var/log/app.log` from the main container. Where does it go in the YAML?
6. A pod is Pending and has **no events at all**. What's broken?

<details>
<summary>Day 7 answers</summary>

1. Read both reasons. Either lower the pod's memory request so it fits on worker1, or add a control-plane toleration (if the task allows that), or free capacity on worker1. Don't touch the scheduler. It's working and has told you exactly why.
2. `tolerations: [{key: node-role.kubernetes.io/control-plane, operator: Exists, effect: NoSchedule}]`. Add `nodeSelector: {node-role.kubernetes.io/control-plane: ""}` if it must run **only** there.
3. `nodeSelector: {disk: ssd}`, or `affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[].matchExpressions: [{key: disk, operator: In, values: [ssd]}]`. Node affinity is the one with operators (`In`, `NotIn`, `Exists`, `Gt`…).
4. `podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution: [{labelSelector: {matchLabels: {app: web}}, topologyKey: kubernetes.io/hostname}]`. Or `topologySpreadConstraints` with `maxSkew: 1`, `topologyKey: kubernetes.io/hostname`, `whenUnsatisfiable: DoNotSchedule`.
5. In `initContainers` with `restartPolicy: Always` (native sidecar), sharing an `emptyDir` mounted in both containers at `/var/log`. Command: `tail -F /var/log/app.log`. (If the task says "add a container", a second entry in `containers` is also accepted.)
6. The scheduler. Either it isn't running (`k get pods -n kube-system | grep scheduler`), or the pod has a `schedulerName` that nothing serves, or `nodeName` points at a node that doesn't exist.

</details>

### Day 8: Services and DNS

1. Expose deployment `web` (container port 8080) on port 80 inside the cluster, then as NodePort 30080.
2. The debug chain for "frontend can't reach backend", in order.
3. FQDN for Service `api` in namespace `prod`? For a pod with IP `10.0.1.5` in `prod`?
4. `nslookup api` works from namespace `prod` but not from `default`. Why?
5. Every lookup in the cluster fails. Where do you look?
6. What's a headless Service and what does DNS return for it?

<details>
<summary>Day 8 answers</summary>

1. `k expose deploy web --port=80 --target-port=8080 --name=web`. For the NodePort, generate it with `k expose ... --type=NodePort $do`, set `nodePort: 30080`, apply (or `k patch svc web -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":8080,"nodePort":30080}]}}'`).
2. **DNS → Service → EndpointSlice → pod labels → container port.** `nslookup <svc>.<ns>` from a test pod → `k get svc` (port, selector) → `k get endpointslices -l kubernetes.io/service-name=<svc>` (empty?) → `k get pod --show-labels` against the selector → `targetPort` against `containerPort` / what the app actually listens on.
3. `api.prod.svc.cluster.local`. Pod: `10-0-1-5.prod.pod.cluster.local`.
4. The short name `api` only expands with the **client's** namespace from the search list in its `/etc/resolv.conf`. From `default`, use `api.prod`.
5. `k get pods -n kube-system -l k8s-app=kube-dns` (Running? Ready?), `k logs -n kube-system -l k8s-app=kube-dns`, the `coredns` ConfigMap (Corefile), the `kube-dns` Service and its endpoints, and the `nameserver` IP in a pod's `/etc/resolv.conf` against the kubelet's `clusterDNS`.
6. `clusterIP: None`. No virtual IP and no kube-proxy rules. DNS returns the **pod IPs** directly (A records per ready pod). StatefulSets use one for stable `pod-0.svc.ns.svc.cluster.local` names.

</details>

### Day 9: NetworkPolicy, Ingress, Gateway API, CNI

1. Default-deny all ingress in namespace `app`.
2. You add `policyTypes: [Egress]` with one egress rule to the DB. Now nothing resolves. Why, and what's the fix?
3. Peer with `namespaceSelector` and `podSelector` in **one** list item vs two list items?
4. How do you select namespace `monitoring` by name?
5. Ingress for `shop.example.com/` → Service `shop:80`, class `nginx`, one line.
6. Minimum objects for Gateway API routing, and the field that links a route to a gateway.
7. Send 80% to `v1`, 20% to `v2` with Gateway API.
8. Where do CNI binaries and configs live on a node?

<details>
<summary>Day 9 answers</summary>

1. `podSelector: {}`, `policyTypes: [Ingress]`, no `ingress:` rules.
2. Once Egress is in `policyTypes`, anything not allowed is denied, including DNS. Add an egress rule to kube-system's DNS pods on **UDP and TCP 53**: `to: [{namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}, podSelector: {matchLabels: {k8s-app: kube-dns}}}]`, `ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]`.
3. One item = **AND** (those pods in those namespaces). Two items = **OR**. A `podSelector` alone always means the policy's own namespace.
4. `namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: monitoring}}`. That label is set automatically on every namespace.
5. `k create ingress shop --class=nginx --rule="shop.example.com/=shop:80"` (`/*` instead of `/` gives `pathType: Prefix`; `/` alone gives `Exact`).
6. A `GatewayClass` (usually exists already: `k get gatewayclass`), a `Gateway` with a listener, and an `HTTPRoute` whose `spec.parentRefs` names the Gateway (and its namespace if it's different). `backendRefs` point to Services.
7. One rule with `backendRefs: [{name: v1, port: 80, weight: 80}, {name: v2, port: 80, weight: 20}]`.
8. Binaries in `/opt/cni/bin`, config in `/etc/cni/net.d` (the kubelet/containerd uses the first file by name). No config there → node `NotReady` with `cni plugin not initialized`.

</details>

### Day 10: storage

1. What has to match for a PVC to bind to a static PV?
2. PVC Pending, PV Available, both look right. The first two things to check?
3. StorageClass has `volumeBindingMode: WaitForFirstConsumer`, and the PVC is Pending with no pod. Is it broken?
4. Expand PVC `data` from 1Gi to 2Gi. What must be true, and what's the command?
5. A PV with `Retain` shows `Released` after its PVC was deleted. How do you make it bindable again?
6. RWO vs RWX vs RWOP in one line each.

<details>
<summary>Day 10 answers</summary>

1. `storageClassName` (same name, or both `""`), access modes (the PV must offer the requested ones), capacity (PV ≥ request), `volumeMode`, and the PVC's `selector` if set. A PVC with **no** `storageClassName` gets the default class, so it won't bind to a PV with `""` or a different class.
2. `k describe pvc` events, then compare `storageClassName` on both (`k get pv,pvc -o wide`). A default StorageClass silently putting its name on the PVC is the classic cause.
3. No. It binds when a pod that uses it is scheduled. Create the pod.
4. The StorageClass needs `allowVolumeExpansion: true`. Then `k patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'` (or `k edit`). You can only make it bigger.
5. `k patch pv <pv> --type=json -p '[{"op":"remove","path":"/spec/claimRef"}]'` (or edit and delete `claimRef`). The data is still there.
6. RWO: mounted read-write by one **node** (several pods on that node can use it). RWX: read-write by many nodes. RWOP: read-write by exactly one **pod** (CSI only).

</details>

### Day 11: break and fix the cluster

1. The order for "kubectl: connection refused". Steps 1–4.
2. The order for "node NotReady".
3. The kubelet is `activating (auto-restart)`. What's your next command?
4. kube-apiserver.yaml has a YAML indentation error. What does `crictl ps -a` show, and where's the evidence?
5. Pods stay Pending with no events. Pods don't come back after deletion and `scale` does nothing. Which component is down in each case?
6. The rule before editing any static pod manifest?

<details>
<summary>Day 11 answers</summary>

1. Right context and server address (`k config view --minify | grep server`) → anything listening on 6443 (`ss -tlnp | grep 6443`) → `crictl ps -a --name kube-apiserver` → `crictl logs <id>`. Then the kubelet journal, the kubelet itself, etcd, certs.
2. `k describe node` conditions → ssh, `sudo -i` → `systemctl status kubelet` → `journalctl -u kubelet -n 50` → `systemctl status containerd` → fix, restart, `k get nodes -w`.
3. `journalctl -u kubelet -n 50 --no-pager`. The first `E…` line names the file and the field (for example the `clientCAFile` path).
4. Nothing: no container for it at all. The evidence is in `journalctl -u kubelet` (`Could not process manifest file`).
5. No events → **scheduler**. No reconciliation → **controller-manager**.
6. `cp` it to `/tmp` first (never a backup inside `/etc/kubernetes/manifests`, because the kubelet would run it as a second pod). If you give up, restore the backup.

</details>

---

## 5. Three fault injections (Block F)

Snapshot both VMs first (runbook 1.10 style, `--name day13`). Pick three of these at random: write the numbers 1–6 on paper, close your eyes, pick. Better still, write the three inject commands in a script, run it, and wait 2 minutes so you don't know which ones you picked.

| # | Inject (on the named node, `sudo -i`) | Target |
|---|---|---|
| 1 | worker1: `systemctl stop kubelet` | 3 min |
| 2 | worker1: `sed -i 's#ca.crt#ca.crtx#' /var/lib/kubelet/config.yaml && systemctl restart kubelet` | 5 min |
| 3 | cp1: `sed -i 's#127.0.0.1:2379#127.0.0.1:2380#' /etc/kubernetes/manifests/kube-apiserver.yaml` | 6 min |
| 4 | cp1: `sed -i 's#\(kube-scheduler:\).*#\1v9.9.9#' /etc/kubernetes/manifests/kube-scheduler.yaml` | 5 min |
| 5 | cp1: `sed -i 's#--leader-elect=true#--leader-elect=maybe#' /etc/kubernetes/manifests/kube-controller-manager.yaml` | 5 min |
| 6 | worker1: `systemctl stop containerd` | 4 min |

Back up the files before injecting (`cp ... /tmp/inject-bak/`), so you have a known-good copy if a fix goes wrong. That's the same habit the exam needs, the other way round.

**Score each run**: time to the first piece of evidence, time to fixed, and whether you **proved** it (a pod scheduled, a scale that worked, the node Ready for a full minute with flat restart counts). A fix without the proof counts as a miss today, because it's the habit that costs points on Friday.

If any fault takes over its target, do the Day 11 fixed order for that symptom out loud once, then inject the same fault again.

---

## 6. "No earlier criterion still failing" (Block G)

The third acceptance criterion is the sweep. Go through the stories and tick each criterion you **can do now, from memory, inside its time**. Don't redo all of them. Spot-check one per day and only redo what you can't say with certainty.

| Day | Spot-check (timed) | Target |
|---|---|---|
| 2 | etcd backup + restore (done in Block B) | 8 min |
| 3 | Control-plane upgrade (done in Block C) | 10 min |
| 4 | SA + Role + RoleBinding + 2 `auth can-i` checks | 3 min |
| 5 | Helm install with `--set`, upgrade, rollback | 4 min |
| 6 | Rollback to a revision + ConfigMap volume mount | 4 min |
| 7 | Pod on cp1 only (toleration + nodeSelector) | 3 min |
| 8 | Expose + debug chain on a Service with a wrong selector | 4 min |
| 9 | Default deny + allow one pod, including DNS egress, tested both ways | 6 min |
| 10 | PV + PVC + pod mount, `Bound`, file written | 5 min |
| 11 | One fault (done in Block F) | 6 min |

Anything still over its time goes to the top of the Day 14 story's Notes, so you do it before session 2.

---

## 7. Traps that cost points (review day)

- **Re-reading lessons instead of typing.** Reading feels productive and doesn't make you faster. Every card on the sheet that's a command, type it.
- **Memorising killer.sh answers.** Session 2 is the same paper. A memorised answer gives you a nice score tomorrow and nothing on Friday. Fix the cause.
- **Upgrading the lab past 1.35.** Practise on the `v134` snapshot. 1.36 needs containerd 2.x on Ubuntu, and you don't want a lab rebuild two days before the exam.
- **Restoring etcd over the live data dir.** Always a new `--data-dir`. Restoring into `/var/lib/etcd` while etcd is running fails, or worse.
- **Skipping the proof on a fixed fault.** "It looks fixed" isn't a result. Scale something, schedule something, watch restarts for a minute.
- **A long session the night before.** Today and tomorrow are both 3-hour days. Stop when the time box ends. Being rested on Friday is worth more than one extra drill.

---

## 8. Self-check: answer before you open the lab

1. During the etcd restore you changed the `etcd-data` hostPath to `/var/lib/etcd-restore`. Why don't you also need to change `--data-dir` in the command?
2. You're on a 1.36 control plane and need 1.37. Write the first three commands, before `kubeadm upgrade plan`.
3. A RoleBinding in `dev` references the ClusterRole `view`. Can its subject list pods in `prod`?
4. An egress NetworkPolicy allows TCP 5432 to the DB pods. The app logs `could not translate host name "db"`. What's missing?
5. A PVC with no `storageClassName` stays Pending next to an Available PV with `storageClassName: ""`. Why?
6. `kubectl` returns connection refused. `crictl ps -a --name kube-apiserver` shows nothing at all. Where do you look next, and what are you expecting?
7. Your killer.sh log shows three misses, all "misread" (wrong namespace, wrong port, wrong file path). Which domain do you drill today, and what do you actually change?
8. What are the four checks that prove a kubeadm upgrade is complete?

<details>
<summary>Answers</summary>

1. `--data-dir` is the path **inside the container**, `/var/lib/etcd`, and that's the volume's `mountPath`. The hostPath is the path on the node. Changing the hostPath mounts the restored directory where etcd already looks. If you change `--data-dir` as well, you also have to change the `volumeMounts.mountPath` to match it. Changing only one of those two is the usual way people break the restore.
2. `sed -i 's/v1.36/v1.37/' /etc/apt/sources.list.d/kubernetes.list` (cat it first), `apt-get update`, then `apt-mark unhold kubeadm && apt-get install -y kubeadm='1.37.x-*' && apt-mark hold kubeadm` (pick x with `apt-cache madison kubeadm`).
3. No. A ClusterRole bound by a RoleBinding only grants its rules in the binding's namespace, `dev`. For `prod` you need a RoleBinding in `prod` or a ClusterRoleBinding.
4. DNS egress: UDP **and** TCP 53 to the kube-dns pods in kube-system. Once `Egress` is in `policyTypes`, DNS is denied unless you allow it.
5. A PVC with no `storageClassName` gets the **default** StorageClass's name (retroactively too, since v1.28), so it no longer matches a PV with `""`. Set `storageClassName: ""` on the PVC explicitly, or give both the same class.
6. `journalctl -u kubelet -n 50 | grep -i apiserver`. No container means the kubelet never started one, usually because the manifest doesn't parse (`Could not process manifest file`) or the image can't be pulled. If the journal is quiet, check the kubelet itself is running (`systemctl status kubelet`).
7. Not a domain. They're habit misses, so they fix themselves only if you change how you work: copy names from the task with Ctrl+Shift+C/V instead of retyping, re-read the task's last line before `exit`, and put `-n <ns>` in every command. Drill those habits through every block today, and pick the weak domains from your knowledge-gap and slow misses instead.
8. `kubeadm version` shows the new version on each node, `k get nodes` shows the new **kubelet** version on every node, no node is `SchedulingDisabled`, and all kube-system pods are Running.

</details>

---

**Done when:** every [Day 13 acceptance criterion](../stories/day-13-review.md#acceptance-criteria) passes: each Day 12 miss solved from memory, etcd restore under 8 minutes and control-plane upgrade under 10, and nothing from an earlier story still failing (section 6). Tomorrow (Day 14) is killer.sh session 2 with the same questions: the goal is every task correct and verified in under 90 minutes, followed by the exam-day checklist.
