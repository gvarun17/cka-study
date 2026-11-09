# Day 7 Lesson: Scheduling, multi-container pods and a mini mock

**Story:** [day-07-scheduling](../stories/day-07-scheduling.md) · **Read time:** ~35 min · **Then:** 1 hour of drills + the 1-hour mini mock

**Why this matters:** Workloads & Scheduling is **15%** of the exam, and placement problems are also a large part of **Troubleshooting (30%)**. A Pending pod is the most common symptom on the exam, and the scheduler always tells you why in one Events line, if you know how to read it. Expect a task like "make this Deployment run one replica per node", "schedule this pod only on nodes labeled `disk=ssd`", "this pod must run on the control-plane node", or "add a sidecar that tails the app's log". Each one is a 3–5 minute task once the YAML shapes are in your fingers. None of them have an imperative flag, so the route is always `$do` → vim → apply.

On EKS you used Karpenter or managed node groups, and placement was mostly node labels plus a toleration for a GPU taint. Here there's no autoscaler to add a node when nothing fits. The pod just stays Pending, and you need to read why.

> **Version notes (checked today).**
> - The exam is on **Kubernetes v1.37** (see Day 2). Your lab is on **v1.35** after Day 3. Nothing in today's lesson differs between the two.
> - **Native sidecar containers are GA since v1.33** ([Sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)). The feature is locked on. A sidecar is an entry in `initContainers` with `restartPolicy: Always`. It starts before the app containers, keeps running alongside them, and is stopped after them. Section 6 covers it.

---

## 1. How the scheduler decides

```
 new pod (spec.nodeName empty)
      │
      ▼  scheduling queue (ordered by PriorityClass)
 FILTER: drop every node that can't run it          ← reasons appear in Events as "0/N nodes are available: ..."
   NodeUnschedulable   cordoned node
   NodeName            spec.nodeName set to another node
   TaintToleration     a NoSchedule/NoExecute taint the pod doesn't tolerate
   NodeAffinity        nodeSelector / required node affinity doesn't match
   NodeResourcesFit    requests don't fit in allocatable - already-requested
   InterPodAffinity    required pod (anti-)affinity violated
   PodTopologySpread   maxSkew violated with DoNotSchedule
   VolumeBinding       PVC unbound / PV on another node
      │
      ▼
 SCORE the survivors (preferred affinity, PreferNoSchedule, spreading, least allocated ...)
      │
      ▼
 BIND: write spec.nodeName  ──►  the kubelet on that node starts the pod
      │
 no node survived? → try PREEMPTION (evict lower-priority pods) → else Pending + FailedScheduling event
```

Three facts to hold on to:

- **The scheduler only looks at requests**, never real usage. A node at 5% CPU can still be "Insufficient cpu" if its pods' requests add up to its allocatable (Day 6).
- **`spec.nodeName` skips the scheduler completely.** The kubelet on that node just runs the pod. No taint check, no resource check (the kubelet may then reject it with `OutOfcpu`). That's how static pods and DaemonSets used to bypass placement.
- **Pending with no event at all** means the scheduler isn't running (Day 11). Pending with `FailedScheduling` means it's running and telling you why.

### Reading the Pending reason (acceptance criterion 2)

```bash
k describe pod web-xxx | tail -5
k get events --field-selector reason=FailedScheduling --sort-by=.lastTimestamp
```

```
Warning  FailedScheduling  default-scheduler  0/2 nodes are available: 1 Insufficient cpu,
  1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }.
  preemption: 0/2 nodes are available: 1 No preemption victims found for incoming pod,
  1 Preemption is not helpful for scheduling.
```

The message counts nodes per reason, and the counts add up to the total. Read it as "worker1 is out of CPU, cp1 is tainted". The `preemption:` half only says whether evicting lower-priority pods would help.

| Fragment in the event | Meaning | Usual fix |
|---|---|---|
| `had untolerated taint {key: value}` | Taint without a matching toleration | Add the toleration, or remove the taint if the task says so |
| `didn't match Pod's node affinity/selector` | `nodeSelector` or required node affinity matches no node | Fix the label on the node or the selector (typos: `disktype` vs `disk-type`) |
| `Insufficient cpu` / `Insufficient memory` | Requests don't fit | Lower requests, free space, or scale down something else |
| `didn't match pod anti-affinity rules` | Required anti-affinity: every node already has a matching pod | More nodes, fewer replicas, or `preferred` instead of `required` |
| `didn't match pod affinity rules` | Required affinity: no node has the pod it must be next to | Create or label the target pod, or fix the selector |
| `node(s) were unschedulable` | Node cordoned | `k uncordon <node>` |
| `didn't match pod topology spread constraints` | `maxSkew` would be exceeded | Raise `maxSkew` or use `ScheduleAnyway` |
| `pod has unbound immediate PersistentVolumeClaims` | PVC Pending | Day 10 |
| `node(s) had volume node affinity conflict` | PV is tied to another node | Day 10 |

---

## 2. Taints and tolerations

A **taint** on a node repels pods. A **toleration** on a pod lets it ignore a matching taint. A toleration **does not attract** a pod to that node. For "only these pods, and only on these nodes", you need a taint **plus** a toleration **plus** node affinity or a nodeSelector.

```bash
k taint nodes worker1 dedicated=gpu:NoSchedule           # add  (key=value:effect)
k taint nodes worker1 dedicated=gpu:NoSchedule-          # remove: same thing + trailing dash
k taint nodes worker1 dedicated-                         # remove every taint with that key
k describe node worker1 | grep -A3 Taints
k get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

| Effect | New pods without a toleration | Pods already running without a toleration |
|---|---|---|
| `NoSchedule` | Not scheduled | Stay |
| `PreferNoSchedule` | Avoided if possible (a score, not a filter) | Stay |
| `NoExecute` | Not scheduled | **Evicted**, immediately or after the toleration's `tolerationSeconds` |

```yaml
spec:
  tolerations:
  - key: dedicated            # Equal: key, value and effect must all match
    operator: Equal
    value: gpu
    effect: NoSchedule
  - key: node-role.kubernetes.io/control-plane   # Exists: any value
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 60     # evicted 60 s after the node becomes unreachable (default added by admission: 300)
  # - operator: Exists        # key omitted + Exists = tolerate EVERY taint (what DaemonSets like calico-node do)
```

**Why the control plane is tainted.** kubeadm's `mark-control-plane` phase adds `node-role.kubernetes.io/control-plane:NoSchedule` (Day 1) so user workloads can't starve the API server and etcd. "Run this pod on the control plane" therefore needs **both** the toleration above **and** a selector for cp1 (`nodeSelector: {node-role.kubernetes.io/control-plane: ""}` or `kubernetes.io/hostname: cp1`), or `nodeName: cp1`.

The node lifecycle controller also adds taints itself: `node.kubernetes.io/not-ready`, `unreachable` (NoExecute, Day 11), `unschedulable` (cordon, Day 3), and `memory-pressure` / `disk-pressure`.

---

## 3. nodeName, nodeSelector, node affinity

```yaml
spec:
  nodeName: worker1          # bypasses the scheduler. Taints ignored. Typo = pod never starts (no event!)
---
spec:
  nodeSelector:              # exact label match, ANDed. The simplest correct answer when it's enough
    disktype: ssd
```

```bash
k label node worker1 disktype=ssd            # add
k label node worker1 disktype=nvme --overwrite
k label node worker1 disktype-               # remove
k get nodes -L disktype                      # show as a column
k get nodes --show-labels | grep -o 'kubernetes.io/hostname=[^,]*'
```

Node affinity is nodeSelector with operators and soft rules:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:     # hard filter
        nodeSelectorTerms:                                # terms are ORed
        - matchExpressions:                               # expressions inside one term are ANDed
          - key: disktype
            operator: In                                  # In NotIn Exists DoesNotExist Gt Lt
            values: [ssd, nvme]
          - key: kubernetes.io/arch
            operator: In
            values: [amd64]
      preferredDuringSchedulingIgnoredDuringExecution:    # soft: adds weight to the score
      - weight: 80                                        # 1-100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [a]
```

- **`IgnoredDuringExecution`**: changing a node's labels later doesn't evict pods already running there.
- **Terms ORed, expressions ANDed.** It's the same pattern as Day 9's NetworkPolicy peers. A second `- matchExpressions:` is a second, alternative term.
- `NotIn` and `DoesNotExist` give you anti-affinity to nodes ("anywhere except nodes labeled `env=prod`").
- If both `nodeSelector` and `nodeAffinity` are set, **both** must match.

---

## 4. Pod affinity and anti-affinity

Pod (anti-)affinity places pods **relative to other pods**, per **topology domain**. `topologyKey` names the node label that defines a domain: `kubernetes.io/hostname` means "same node", and `topology.kubernetes.io/zone` means "same zone".

```yaml
spec:
  affinity:
    podAntiAffinity:                      # "never two of me on one node"
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: {app: web}         # which pods to keep away from (usually your own label)
        topologyKey: kubernetes.io/hostname
    podAffinity:                          # "put me on a node that already runs a cache pod"
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:                  # note the extra level in the preferred form
          labelSelector:
            matchLabels: {app: cache}
          topologyKey: kubernetes.io/hostname
```

- The `labelSelector` matches pods in the **pod's own namespace** unless you add `namespaces: [...]` or `namespaceSelector: {}`.
- In a Deployment, the affinity goes in **`spec.template.spec.affinity`**, and the selector must match the **template's** labels. A label typo means the rule matches nothing and has no effect, silently.
- **Required anti-affinity with more replicas than eligible nodes leaves the extra replicas Pending.** On your lab only worker1 is eligible (cp1 is tainted), so 3 replicas with required anti-affinity give **1 Running, 2 Pending**. Add the control-plane toleration and you get 2 Running, 1 Pending. That's correct behaviour, not a bug. The exam cluster usually has enough nodes, and the task says which form to use. If it says "at most one per node", use `required`. If it says "spread where possible", use `preferred` or a topology spread constraint.

### topologySpreadConstraints: the other way to spread

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                             # max difference in matching pods between any two domains
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule       # or ScheduleAnyway (soft)
    labelSelector:
      matchLabels: {app: web}
```

Spread constraints balance evenly (2/2/1) and allow more pods than nodes. Anti-affinity forbids sharing a node at all. Read the task's wording to choose.

---

## 5. PriorityClass and preemption

```bash
k create priorityclass high --value=100000 --description="business critical"
k create priorityclass low  --value=100 --preemption-policy=Never     # waits, never evicts others
k create priorityclass default-pc --value=1000 --global-default=true  # applies to pods with no class
k get pc            # also shows the built-ins: system-cluster-critical (2000000000), system-node-critical (2000001000)
```

```yaml
spec:
  priorityClassName: high      # admission sets spec.priority from it; the class must already exist
```

- **PriorityClass is cluster-scoped.** Higher value = higher priority. User classes can go up to 1,000,000,000. Above that is reserved for the `system-` classes.
- **Queue order:** higher-priority Pending pods are tried first.
- **Preemption:** if a high-priority pod fits nowhere, the scheduler picks a node where evicting **lower-priority** pods would make room, evicts them (gracefully, respecting PDBs on a best-effort basis), and sets `status.nominatedNodeName` on the waiting pod. Evicted pods from a Deployment are recreated and go Pending themselves.
- `preemptionPolicy: Never` gives queue priority without evicting anyone.
- `globalDefault` only affects pods created **after** it's set. Existing pods keep priority 0.

Exam pattern: "create PriorityClass X with value Y, and make Deployment Z use it". Then check with `k get pod -o custom-columns=NAME:.metadata.name,PRIO:.spec.priority`.

---

## 6. Multi-container pods: init, sidecar, shared emptyDir

| Kind | Where in spec | Runs | Typical use |
|---|---|---|---|
| **Init container** | `initContainers`, no `restartPolicy` | One at a time, in order, each **to completion**, before any app container starts. If one fails, the kubelet retries it (per the pod's `restartPolicy`) | Wait for a Service, fetch config, run a migration, fix permissions |
| **Native sidecar** (GA v1.33) | `initContainers` with `restartPolicy: Always` | Starts in order with the init containers, but doesn't need to finish. Keeps running beside the app, restarts if it dies, stops **after** the app | Log shipper, proxy, file syncer. Doesn't block a Job from completing |
| **Regular extra container** | `containers` | Starts alongside the others, no ordering | The "old-style" sidecar. Still a correct answer if the task just says "add a second container" |

Status while init containers run: `Init:0/2`, `Init:1/2`; a failing one shows `Init:Error` or `Init:CrashLoopBackOff`. Logs: `k logs <pod> -c <init-name>`.

A complete pod with all three, sharing one `emptyDir`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  volumes:
  - name: logs
    emptyDir: {}                     # lives as long as the pod; shared by every container that mounts it
  initContainers:
  - name: wait-db                    # 1. must finish before anything else runs
    image: busybox:1.36
    command: ["sh", "-c", "until nslookup db.default.svc.cluster.local; do echo waiting; sleep 2; done"]
  - name: log-shipper                # 2. native sidecar: started next, keeps running
    image: busybox:1.36
    restartPolicy: Always
    command: ["sh", "-c", "touch /var/log/app/app.log; tail -F /var/log/app/app.log"]
    volumeMounts:
    - {name: logs, mountPath: /var/log/app}
  containers:
  - name: app                        # 3. main container
    image: busybox:1.36
    command: ["sh", "-c", "while true; do date >> /var/log/app/app.log; sleep 5; done"]
    volumeMounts:
    - {name: logs, mountPath: /var/log/app}
```

```bash
k get pod app                       # Init:0/2 until a Service called db exists (k create svc clusterip db --tcp=80) → 2/2 Running
k logs app -c log-shipper -f        # the dates the app writes
k exec app -c app -- ls /var/log/app
```

Fastest route: `k run app --image=busybox:1.36 $do --command -- sh -c '...' > app.yaml`, then add `volumes`, `volumeMounts` and the extra containers in vim. `emptyDir: {medium: Memory}` gives you a tmpfs, if a task asks for it.

---

## 7. Hands-on walkthrough on cp1 + worker1

From cp1: `k create ns d7 && k config set-context --current --namespace=d7`.

**Step 1: taints (task 1).**

```bash
k taint nodes worker1 dedicated=gpu:NoSchedule
k create deploy t --image=nginx --replicas=2 ; k get pods -o wide        # Pending: read the event
k describe pod -l app=t | grep -A3 Events
k taint nodes worker1 dedicated=gpu:NoSchedule-                           # Running
k taint nodes worker1 evict=now:NoExecute ; k get pods -o wide -w        # evicted immediately, then Pending
k taint nodes worker1 evict-                                              # remove it quickly: CoreDNS on worker1 was evicted too
k describe node cp1 | grep Taints                                        # node-role.kubernetes.io/control-plane:NoSchedule
```

Then edit Deployment `t` to tolerate `evict=now:NoExecute` with `tolerationSeconds: 30`, re-add the taint, and watch the pods survive for 30 s before going.

**Step 2: nodeSelector, affinity, nodeName (tasks 2–3).**

```bash
k label node worker1 disktype=ssd
k run sel --image=nginx $do > sel.yaml        # add nodeSelector: {disktype: ssd}; apply → worker1
k run aff --image=nginx $do > aff.yaml        # required nodeAffinity disktype In [nvme] → Pending; read the event; fix to ssd
k run byname --image=nginx $do > byname.yaml  # add nodeName: cp1 → Running on cp1 despite the taint
k get pods -o wide
```

Then set `nodeName: cp2` (doesn't exist) and look: Pending forever, **no event**. That's the nodeName trap.

**Step 3: spread 3 replicas with anti-affinity (acceptance criterion 1).** ⏱ Target **< 5 min**.

```bash
# --- start the timer ---
k create deploy spread --image=nginx --replicas=3 $do > spread.yaml
vim spread.yaml     # under spec.template.spec: podAntiAffinity required, labelSelector app=spread,
                    # topologyKey kubernetes.io/hostname; plus the control-plane toleration
k apply -f spread.yaml
k get pods -l app=spread -o wide        # one on worker1, one on cp1, one Pending ("didn't match pod anti-affinity rules")
# --- stop the timer ---
```

On a 2-node lab, the third Pending replica **is** the proof the rule works. Then switch to `preferred` (watch the extra level: `weight` + `podAffinityTerm`) and see all 3 Running. Then try the `topologySpreadConstraints` version with `maxSkew: 1` and see a 2/1 split.

**Step 4: priority and preemption (task 4).** Fill worker1, then push in a high-priority pod:

```bash
k create pc low --value=10 ; k create pc high --value=100000
k describe node worker1 | grep -A6 'Allocated resources'      # note allocatable vs requested CPU
k create deploy filler --image=nginx --replicas=4 $do > filler.yaml
#   add priorityClassName: low and resources.requests.cpu so 4 replicas just about fill worker1 (e.g. 400m each on 2 CPUs)
k apply -f filler.yaml
k run vip --image=nginx $do > vip.yaml   # priorityClassName: high, requests.cpu: 800m
k apply -f vip.yaml ; k get pods -o wide -w   # filler pods evicted → Pending; vip Running on worker1
k get pod vip -o jsonpath='{.spec.priority}{"\n"}'
```

**Step 5: multi-container (task 5).** Apply the pod from section 6, watch it stay at `Init:0/2`, create the `db` Service, watch it reach `2/2`. Break the init container's command and look at `Init:CrashLoopBackOff` and `k logs app -c wait-db`.

**Cleanup:** `k delete ns d7`, `k label node worker1 disktype-`, `k delete pc low high`, check `k describe node worker1 | grep Taints` → `<none>`.

---

## 8. Mini mock (1 hour, no notes, docs only)

Set a timer for 60 minutes. Do them in any order, flag and skip anything over 8 minutes. Use a fresh namespace per task where none is named. Score each one only by the check in the right column.

| # | Task | Check |
|---|---|---|
| 1 | (Day 2) Take an etcd snapshot of the cluster to `/opt/backup/etcd-d7.db` and print its status | `sudo etcdutl snapshot status /opt/backup/etcd-d7.db -w table` shows a revision |
| 2 | (Day 4) Create ServiceAccount `ci` in ns `build`, allowed only to `get,list,create` Deployments in `build` | `k auth can-i create deploy -n build --as=system:serviceaccount:build:ci` → yes; `delete` → no; `-n default` → no |
| 3 | (Day 4) Create a kubeconfig context `dev` for user `kubernetes-admin` with default namespace `build`, without switching to it | `k config get-contexts dev` shows namespace build; current context unchanged |
| 4 | (Day 5) Install Helm chart `bitnami/nginx` (or any repo chart you've added) as release `web` in ns `helm-test` with `replicaCount=2` | `helm ls -n helm-test` deployed; 2 pods |
| 5 | (Day 6) Deployment `api` (nginx:1.27, 3 replicas), then update it to nginx:1.28 and roll back to revision 1 | `k rollout history` has 3 revisions; image is nginx:1.27 |
| 6 | (Day 6) Pod `cfg` that gets env var `MODE` from ConfigMap `app` key `mode`, and mounts Secret `db` at `/etc/db` | `k exec cfg -- env \| grep MODE`; `k exec cfg -- ls /etc/db` |
| 7 | (Day 7) Deployment `ha` with 2 replicas that must never share a node and must also run on the control plane | `k get pods -l app=ha -o wide`: one on cp1, one on worker1 |
| 8 | (Day 7) Pod `sidecar` with main container writing to `/data/out.log` every 5 s and a native sidecar tailing it, via a shared emptyDir | `k logs sidecar -c <sidecar>` shows lines; `k get pod sidecar -o jsonpath='{.spec.initContainers[0].restartPolicy}'` → Always |

Score 6/8 or better to pass (acceptance criterion 3). Copy every miss into the **Day 8 Notes** with its cause: knowledge gap, slow, misread, or wrong context.

---

## 9. Traps that cost points

- **Toleration without a selector** when the task says "run on node X". The pod may still land elsewhere. A toleration permits; it doesn't attract.
- **Taint removal without the trailing `-`** re-adds or errors out. And `key=value:Effect` is case-sensitive: `NoSchedule`, not `noschedule`.
- **Affinity at the wrong level** in a Deployment: `spec.affinity` instead of `spec.template.spec.affinity`. `k apply` rejects it as an unknown field, but only if you read the error.
- **Preferred-form indentation.** `preferredDuringScheduling...` needs `weight` + `podAffinityTerm` (pod) or `weight` + `preference` (node). The required form has neither.
- **labelSelector that matches no pods** (`app: web` vs the template's `app: web-app`). The anti-affinity silently does nothing and all replicas pile onto one node.
- **Wrong `topologyKey`.** `hostname` instead of `kubernetes.io/hostname` matches no node label, so every pod is Pending (`required`) or the rule is ignored.
- **`nodeName` typo.** No event, no error, Pending forever. And `nodeName` ignores taints, so it "works" on cp1 where a nodeSelector wouldn't.
- **Assuming Pending means "not enough resources".** Read the event. It names the reason and the count of nodes per reason.
- **PriorityClass referenced before it exists.** The pod is rejected at admission (`no PriorityClass with name X was found`). Create the class first.
- **Init container that never finishes** (e.g. `sleep infinity` without `restartPolicy: Always`). The app containers never start: `Init:0/1` forever.
- **Sidecar in `containers` when the task explicitly says "native sidecar" / "init container with restartPolicy Always"**. Read the wording; the checker may look at `initContainers`.
- **Forgetting the `volumeMounts`** in one of the two containers. Each container must mount the shared volume itself.

---

## 10. Self-check: answer before you open the lab

1. A pod's event says `0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient memory.` What's wrong, and what are two possible fixes?
2. What's the difference between `NoSchedule` and `NoExecute` for a pod that's **already running** on the node when the taint is added?
3. You add a toleration for `dedicated=gpu:NoSchedule` to a pod. Is it guaranteed to run on the GPU node? What else do you need?
4. Write the anti-affinity block (inside a Deployment template) that keeps replicas of `app=web` on different nodes. Where exactly does it go?
5. What happens with that block if the Deployment has 3 replicas and only 2 schedulable nodes? How would you change it so all 3 run while still spreading where possible?
6. How does `nodeName` differ from `nodeSelector`? Name two things `nodeName` skips.
7. A high-priority pod is Pending and `status.nominatedNodeName` is set. What's happening?
8. A pod has two init containers and one app container. Its status shows `Init:1/2`. What has happened so far, and how do you see why it's stuck?

<details>
<summary>Answers</summary>

1. The control-plane node is tainted and the pod doesn't tolerate it, and the two workers don't have enough unrequested memory for the pod's **request**. Fixes: lower the pod's memory request; free memory on a worker (scale down or delete something); or, if the task allows it, tolerate the control-plane taint so it can use cp1. `k describe node <worker> | grep -A8 'Allocated resources'` shows how much is requested.
2. `NoSchedule` only stops new pods. The running pod stays. `NoExecute` also **evicts** running pods that don't tolerate it, immediately, or after `tolerationSeconds` if the toleration sets one.
3. No. A toleration only allows it onto the tainted node, and it can still land on any untainted node. Add a `nodeSelector` (or required node affinity) for the GPU node's label too.
4. Under `spec.template.spec.affinity`:
   ```yaml
   podAntiAffinity:
     requiredDuringSchedulingIgnoredDuringExecution:
     - labelSelector:
         matchLabels: {app: web}
       topologyKey: kubernetes.io/hostname
   ```
5. Two replicas run (one per node) and the third stays Pending with `didn't match pod anti-affinity rules`. To run all three and still spread: use `preferredDuringSchedulingIgnoredDuringExecution` with `weight: 100` and a `podAffinityTerm` wrapping the same selector and topologyKey, or a `topologySpreadConstraints` entry with `maxSkew: 1`, `whenUnsatisfiable: DoNotSchedule` (2/1 split is allowed).
6. `nodeName` assigns the pod directly and the scheduler never sees it. `nodeSelector` is a filter the scheduler applies. `nodeName` skips the taint/toleration check, the resource-fit check, and affinity rules. A wrong node name leaves the pod Pending with no event.
7. Preemption: no node had room, so the scheduler chose that node, is evicting lower-priority pods there (graceful termination), and will bind the pod once room appears. Check with `k get events` for `Preempted` and see the victims' pods go Pending.
8. The first init container finished successfully. The second is running, failing, or restarting, and the app container hasn't started. `k describe pod <pod>` (State of each init container) and `k logs <pod> -c <second-init-name>`.

</details>

---

**Done when:** every [Day 7 acceptance criterion](../stories/day-07-scheduling.md#acceptance-criteria) passes: 3 replicas spread with anti-affinity in < 5 min (on the 2-node lab: one per node, the third Pending for the right reason); the reason for a Pending pod stated from its Events in < 1 min; and 6/8 or better on the mini mock, with every miss copied into Day 8 Notes. Tomorrow (Day 8, Services and DNS) builds on today's labels: a Service finds its pods with the same label selectors your affinity rules use, and a selector typo breaks it just as silently.
