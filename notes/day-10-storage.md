# Day 10 Lesson: PersistentVolumes, PersistentVolumeClaims and StorageClasses

**Story:** [day-10-storage](../stories/day-10-storage.md) · **Read time:** ~30 min · **Then:** 1.5 hours in the lab

**Why this matters:** Storage is only **10%** of the exam, but it's the most predictable 10%. Expect one "create a PV and a PVC and mount it in a pod at /data" task and maybe one "this PVC is Pending, fix it" or "expand this PVC" task. They're quick points if you know the binding rules, and slow ones if you don't, because **there's no `kubectl create pv` or `kubectl create pvc`**. You copy YAML from the docs and edit it. A Pending PVC also turns up in **Troubleshooting (30%)**, disguised as a Pending pod: `pod has unbound immediate PersistentVolumeClaims`.

On EKS you set `storageClassName: gp3` and the EBS CSI driver did the rest. Today you do the matching by hand with a static PV, then watch a provisioner do it for you, and learn the rules the PV controller uses so you can tell in one look why a claim won't bind.

> **Version notes (checked today).**
> - The exam is on **Kubernetes v1.37** (see Day 2). Your lab is on **v1.35**. Nothing in today's lesson differs between the two.
> - **ReadWriteOncePod** has been GA since v1.29 ([blog](https://kubernetes.io/blog/2023/12/18/read-write-once-pod-access-mode-ga/)). It only works with CSI volumes.
> - **Retroactive default StorageClass assignment** has been GA since v1.28 ([blog](https://kubernetes.io/blog/2023/08/18/retroactive-default-storage-class-ga/)). A PVC created with no `storageClassName` while no default exists gets the default class **when one is set later**. Section 2 explains why that matters for static binding.
> - **VolumeAttributesClass** (changing IOPS/throughput on a live volume) has been GA since v1.34 ([blog](https://kubernetes.io/blog/2025/09/08/kubernetes-v1-34-volume-attributes-class/)). It's CSI-only and needs driver support. Recognise the field; it's unlikely to be tested.
> - **local-path-provisioner v0.0.37** is the current release ([rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)). It's not a CSI driver and **doesn't support volume expansion**. Section 5 explains what that means for the "expand a PVC" task in your lab.

---

## 1. How it works: who does what

```
  PVC (namespaced) "I need 1Gi, RWO, class X"
        │
        ▼  PV controller (inside kube-controller-manager) watches PVCs and PVs
  ┌─────────────────────────────┬──────────────────────────────────────────┐
  │ a matching Available PV     │ no match, and class X has a provisioner  │
  │ exists (static)             │ (dynamic)                                │
  │   → bind PVC ⇄ PV           │   → provisioner creates a volume + a PV  │
  │     (PV.spec.claimRef set)  │     → bind                               │
  └─────────────────────────────┴──────────────────────────────────────────┘
        │
        ▼  pod references the PVC by name → scheduler places the pod (VolumeBinding plugin)
  kubelet on that node: attach (CSI) → mount → bind-mount into the container at mountPath
```

| Object | Scope | Who creates it | Key fields |
|---|---|---|---|
| **PersistentVolume** | Cluster | Admin (static) or a provisioner (dynamic) | `capacity.storage`, `accessModes`, `storageClassName`, `persistentVolumeReclaimPolicy`, `volumeMode`, the backend (`hostPath`, `nfs`, `csi`, `local`), optional `nodeAffinity` |
| **PersistentVolumeClaim** | Namespace | User | `resources.requests.storage`, `accessModes`, `storageClassName`, optional `volumeName`/`selector` |
| **StorageClass** | Cluster | Admin | `provisioner`, `parameters`, `reclaimPolicy`, `volumeBindingMode`, `allowVolumeExpansion` |

- **Binding is one-to-one and exclusive.** One PVC binds one PV. A 10Gi PV bound to a 1Gi claim is entirely used, and the claim shows 10Gi capacity.
- A pod only names a **PVC**, never a PV. The PVC must be in the **pod's namespace**.
- **Volume protection:** a PVC in use by a pod, or a PV bound to a PVC, isn't removed when you delete it. It sits in `Terminating` (finalizers `kubernetes.io/pvc-protection` / `pv-protection`) until the pod or claim is gone. That's not a hang; delete the pod.

### Lifecycle states

| PV `STATUS` | Meaning |
|---|---|
| `Available` | Free, not bound |
| `Bound` | Bound to the claim shown in `CLAIM` |
| `Released` | Its claim was deleted, but the policy is `Retain`. The data is still there, and the PV **won't bind to a new claim** until you clear `claimRef` |
| `Failed` | Automatic reclamation failed |

PVC `STATUS` is `Pending` (no match yet, or waiting for a consumer) or `Bound`.

---

## 2. The binding rules (acceptance criterion 2)

A PVC binds to a PV only if **all** of these hold:

| Rule | Detail | Classic mismatch |
|---|---|---|
| **storageClassName equal** | Exact string match. `""` (empty) only matches PVs with no class | PV has `manual`, PVC has `standard` or no class |
| **accessModes** | The PV must offer every mode the PVC asks for | PVC asks `ReadWriteMany`, PV only has `ReadWriteOnce` |
| **capacity** | PV capacity ≥ PVC request | PVC requests 2Gi, PV is 1Gi |
| **volumeMode** | Both `Filesystem` (default) or both `Block` | Rare |
| **selector / volumeName** | If the PVC sets them, the PV must match | Label typo, wrong PV name |
| **PV is Available** | Not Bound, not Released | Old `Released` PV still holding a `claimRef` |

**The no-class trap.** If a PVC omits `storageClassName` and the cluster has a **default** StorageClass, admission fills in the default class. The PVC then won't bind to your static PV with no class (or with `manual`). Instead it waits for, or gets, a dynamically provisioned volume. Two correct fixes:

- Give the PV and the PVC the **same explicit class name** (`storageClassName: manual`, which the docs use). A class name doesn't need a StorageClass object to exist for static binding.
- Or set `storageClassName: ""` **explicitly** on the PVC to mean "no class, don't use the default".

If the task gives you a PV, copy its `storageClassName`, `accessModes` and capacity into your PVC **exactly**. `k get pv <name> -o yaml | grep -A3 -E 'accessModes|capacity|storageClassName'`.

### Access modes (acceptance criterion 3)

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by **one node**. Several pods **on the same node** can still share it |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes (NFS, CephFS, EFS. Not EBS, not local-path) |
| `ReadWriteOncePod` | RWOP | Read-write by **one pod** in the whole cluster. CSI only (GA v1.29) |

Access modes describe what the volume **can** do and are used for matching. They don't stop a second pod mounting an RWO volume on the same node. That's what RWOP is for.

---

## 3. Static provisioning: complete YAML

Start from the docs page [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/). It has exactly this PV, PVC and pod. Bookmark it.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data                    # cluster-scoped: no namespace
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain   # default for manually created PVs
  hostPath:
    path: /mnt/data                # on whichever node the pod lands on (single-node semantics)
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
  namespace: d10
spec:
  storageClassName: manual
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: d10
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-data          # the PVC, not the PV
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "date >> /data/log.txt; sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data             # "mounted at a given path" = this line
```

Fastest route for the pod: `k run writer -n d10 --image=busybox:1.36 $do --command -- sh -c 'date >> /data/log.txt; sleep 3600' > pod.yaml`, then add `volumes` and `volumeMounts`.

- **`hostPath`** is a directory on the node. It isn't shared across nodes, so on a multi-node cluster the data follows the node, not the pod. Fine for the exam's "use hostPath" tasks.
- **`local`** volumes (`local: {path: /mnt/disks/ssd1}`) are the production version: they **require** `nodeAffinity` on the PV, so the scheduler always places the pod on the right node. Use them with a `kubernetes.io/no-provisioner` StorageClass and `WaitForFirstConsumer`.
- Binding to one particular PV: set `volumeName: pv-data` on the PVC (or `selector.matchLabels` against PV labels).

### StatefulSets

A StatefulSet's `volumeClaimTemplates` creates one PVC per replica, named `<template>-<sts>-<ordinal>` (e.g. `data-db-0`). Deleting or scaling down the StatefulSet **doesn't** delete those PVCs by default, so data survives. Scale back up and `db-0` gets `data-db-0` again.

---

## 4. StorageClasses and dynamic provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # optional: make it the default (string "true")
provisioner: rancher.io/local-path          # who creates volumes (ebs.csi.aws.com on EKS)
reclaimPolicy: Delete                       # Delete (default) | Retain, copied onto every PV it creates
volumeBindingMode: WaitForFirstConsumer     # Immediate (default) | WaitForFirstConsumer
allowVolumeExpansion: true
parameters: {}                              # provisioner-specific (type: gp3, iops ...)
```

There's no `kubectl create storageclass`. Copy from the [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) page.

| Field | What to know |
|---|---|
| `volumeBindingMode: Immediate` | Volume created and bound as soon as the PVC exists, before the scheduler knows where the pod goes. With topology-bound storage (zonal EBS, local disks), the pod may then be unschedulable |
| `volumeBindingMode: WaitForFirstConsumer` | PVC stays **Pending** (event: `waiting for first consumer to be created before binding`) until a pod uses it. Then the volume is created on the pod's node or zone. Pending here is **normal** |
| `reclaimPolicy` | Applies to **dynamically created** PVs. Changing it on the class later doesn't change existing PVs; patch those directly |
| `allowVolumeExpansion` | Without it, any edit that grows a PVC of this class is rejected |
| StorageClass fields are **immutable** | Except metadata. To change `reclaimPolicy` or `volumeBindingMode`, delete and recreate the class |

### Default StorageClass

```bash
k get sc                                    # the default shows "(default)" after its name
k patch sc fast -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
k patch sc local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

If two classes are marked default, new PVCs without a class get the **most recently created** one. Don't leave two defaults: unmark the old one when the task says "make X the default".

### Reclaim policies and Released PVs

| Policy | When the PVC is deleted |
|---|---|
| `Delete` | PV **and** backing storage deleted. Default for dynamically provisioned volumes |
| `Retain` | PV goes `Released`, data kept. An admin decides what to do. Default for manually created PVs |
| `Recycle` | Deprecated. Don't use it. Dynamic provisioning replaced it |

To reuse a `Released` PV (data still on it):

```bash
k patch pv pv-data --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'   # → Available
# or: k edit pv pv-data and delete the whole claimRef block
```

To change the policy of an existing PV (e.g. protect a dynamically created volume before deleting its PVC):

```bash
k patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

---

## 5. Expanding a PVC

```bash
k patch pvc pvc-data -n d10 -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
# or: k edit pvc pvc-data and change spec.resources.requests.storage
k get pvc pvc-data -n d10        # CAPACITY changes when the resize finishes
k describe pvc pvc-data -n d10   # conditions: Resizing, FileSystemResizePending
```

- You can only **grow**. Shrinking is rejected.
- The PVC's StorageClass must have `allowVolumeExpansion: true`. Otherwise you get `only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize`. A static PV with no class can't be expanded this way.
- With a CSI driver: the external resizer grows the volume (controller side), then the kubelet grows the filesystem (node side), online in most drivers. `FileSystemResizePending` means the second step is waiting for the pod.
- **Your lab:** local-path-provisioner has no resizer. With `allowVolumeExpansion: true` on the class, the edit is **accepted** and `spec.resources.requests.storage` shows 2Gi, but `status.capacity` stays 1Gi. (local-path doesn't enforce sizes anyway.) That's enough to practise the exam steps. The exam cluster's grader typically checks the spec, but if the task cluster has a CSI driver, also watch CAPACITY change.

---

## 6. Debugging a Pending PVC (acceptance criterion 2)

```bash
k describe pvc <name> | tail -8           # the Events say which case you're in
k get pv                                  # STATUS, CAPACITY, ACCESS MODES, RECLAIM, CLAIM, STORAGECLASS on one line
k get pvc <name> -o yaml | grep -A6 spec
k get sc
```

| PVC event / state | Cause | Fix |
|---|---|---|
| `no persistent volumes available for this claim and no storage class is set` | Static, and no PV matches | Compare class / modes / size with `k get pv`; fix the PVC (it's mostly immutable: delete and recreate it) |
| `storageclass.storage.k8s.io "X" not found` | Class name typo, or class doesn't exist | Use an existing class name |
| `waiting for first consumer to be created before binding` | `WaitForFirstConsumer` | Normal. Create the pod |
| `waiting for a volume to be created, either by external provisioner "X" or manually` | Provisioner isn't running | Check the provisioner's pods (`k -n local-path-storage get pods`) and logs |
| No events, PV `Released` with a matching spec | Old `claimRef` | Remove `claimRef` from the PV |
| PVC picked up a class you didn't write | Default class filled in | `storageClassName: ""` or match the PV's class |

On the pod side: `pod has unbound immediate PersistentVolumeClaims` → go to the PVC. `node(s) had volume node affinity conflict` → the PV is tied to a node the pod can't use (local volume or zone). `persistentvolumeclaim "x" not found` → wrong name or wrong namespace.

Most PVC `spec` fields are **immutable** after creation (except the storage request for expansion). `k get pvc x -o yaml > pvc.yaml`, fix it, `k delete pvc x`, `k apply -f pvc.yaml`. Or `k replace --force -f pvc.yaml`.

---

## 7. Hands-on walkthrough on cp1 + worker1

From cp1: `k create ns d10 && k config set-context --current --namespace=d10`.

**Step 1: static PV + PVC + pod (task 1, acceptance criterion 1).** ⏱ Target **< 6 min**, starting from an empty editor with only the docs page open.

```bash
# --- start the timer ---
vim pv.yaml       # PV pv-data, manual, 1Gi, RWO, Retain, hostPath /mnt/data
vim pvc.yaml      # PVC pvc-data, manual, 1Gi, RWO
k apply -f pv.yaml -f pvc.yaml
k get pv,pvc      # both Bound
k run writer --image=busybox:1.36 $do --command -- sh -c 'date >> /data/log.txt; sleep 3600' > pod.yaml
vim pod.yaml      # add volumes + volumeMounts at /data
k apply -f pod.yaml ; k exec writer -- cat /data/log.txt
# --- stop the timer ---
```

**Step 2: persistence (task 3).**

```bash
k get pod writer -o wide                      # on worker1
k delete pod writer --force ; k apply -f pod.yaml
k exec writer -- cat /data/log.txt            # two lines now: the data survived
ssh worker1 sudo cat /mnt/data/log.txt        # same file on the node
```

**Step 3: reclaim policies (task 4).**

```bash
k delete pod writer ; k delete pvc pvc-data
k get pv pv-data                              # Released, CLAIM still d10/pvc-data
k apply -f pvc.yaml ; k get pvc               # Pending: a Released PV doesn't rebind
k patch pv pv-data --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
k get pv,pvc                                  # Available → Bound
```

**Step 4: Pending PVC drills (acceptance criterion 2).** ⏱ Each **< 1 min** to name the mismatch. Delete the PVC, change one field in `pvc.yaml`, apply, and run `k describe pvc` + `k get pv`:

- `storage: 2Gi` (bigger than the PV)
- `accessModes: [ReadWriteMany]`
- `storageClassName: manul`
- remove `storageClassName` entirely: Pending (no default class exists yet, and an empty class doesn't match `manual`)

Finish by re-applying the original `pvc.yaml` so `pvc-data` is Bound to `pv-data` again. Then create one more claim, `pvc-noclass` (500Mi, RWO, no `storageClassName`), and leave it Pending. After step 5 sets a default class, it picks up `fast` on its own. That's the retroactive default assignment in action.

**Step 5: dynamic provisioning (tasks 2 and 5).**

```bash
k apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
k -n local-path-storage get pods              # local-path-provisioner Running
k get sc local-path -o yaml | grep -E 'provisioner|reclaim|binding'   # rancher.io/local-path, Delete, WaitForFirstConsumer
```

Create your own class from section 4 (`fast`, `Retain`, `WaitForFirstConsumer`, `allowVolumeExpansion: true`), then make it the default:

```bash
k apply -f sc-fast.yaml
k patch sc fast -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
k get sc                                       # fast (default)
k get pvc pvc-noclass                          # STORAGECLASS now fast (retroactive default), Pending until a pod uses it
```

A PVC with no class, then a pod using it:

```bash
# pvc-dyn.yaml: name pvc-dyn, RWO, 500Mi, no storageClassName
k apply -f pvc-dyn.yaml ; k get pvc pvc-dyn    # STORAGECLASS fast, Pending: waiting for first consumer
# pod using pvc-dyn at /data (copy pod.yaml, change names)
k get pvc pvc-dyn ; k get pv                   # Bound; a pvc-<uid> PV with RECLAIM Retain, created by the provisioner
ssh worker1 sudo ls /opt/local-path-provisioner/   # the directory local-path created for it
```

**Step 6: expand (task 6).**

```bash
k patch pvc pvc-dyn -p '{"spec":{"resources":{"requests":{"storage":"1Gi"}}}}'
k get pvc pvc-dyn -o jsonpath='{.spec.resources.requests.storage} {.status.capacity.storage}{"\n"}'   # 1Gi 500Mi (no resizer on local-path)
k patch pvc pvc-data -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'                     # rejected: static, class "manual" has no StorageClass
```

**Cleanup:** `k delete ns d10`, then `k get pv` and delete the leftover Retained PVs (`k delete pv pv-data pvc-...`). Unset `fast` as default and delete it, or keep local-path as the default for the mocks: `k patch sc local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`.

---

## 8. Traps that cost points

- **Namespace on the PV.** PVs are cluster-scoped. A `namespace:` in PV metadata is ignored, and people then look for it in the wrong place. The **PVC** must be in the pod's namespace.
- **No `storageClassName` on the PVC** in a cluster with a default class. It won't bind to your static PV. Match the PV's class or set `""`.
- **Capacity units.** `1Gi` (gibibyte) and `1G` (gigabyte) differ. A PVC asking for `1Gi` won't bind to a PV of `1G`, because 1G < 1Gi. Copy the unit from the task.
- **Treating `WaitForFirstConsumer` Pending as a fault.** It binds when a pod uses it. Read the event before you change anything.
- **`claimName` pointing at the PV name.** The pod references the PVC.
- **Editing an immutable PVC field.** `k edit` refuses; delete and recreate.
- **Expecting a `Released` PV to rebind.** Remove `claimRef` first.
- **Deleting a PVC with `Delete` policy** when the task said "keep the data". Patch the PV to `Retain` first.
- **Two default StorageClasses.** Unmark the old one.
- **`is-default-class: true` without quotes.** Annotations are strings; YAML `true` unquoted fails validation in `annotations`. Use `"true"`.
- **Wrong mount path.** "Mounted at /var/www/html" means `mountPath: /var/www/html` in the container, not `hostPath.path`. Check with `k exec <pod> -- df -h /var/www/html`.
- **Stuck in `Terminating`.** A PVC used by a running pod waits for the pod. Delete the pod; don't strip finalizers.

---

## 9. Self-check: answer before you open the lab

1. A PV is `1Gi, RWO, storageClassName: manual`. A PVC asks for `500Mi, RWO` and has no `storageClassName`. The cluster has a default StorageClass. Does it bind to the PV? What should the PVC say?
2. Explain RWO, ROX, RWX and RWOP. Can two pods on the same node both write to an RWO volume?
3. A PVC with class `local-path` has been Pending for 5 minutes. What's the first event you hope to see, and what does it mean?
4. You delete a PVC whose PV has `Retain`. What state is the PV in, can a new PVC bind to it, and how do you make it reusable?
5. Write the commands to make `fast` the default StorageClass in a cluster where `local-path` is currently the default.
6. A PVC is bound to a 1Gi PV. Can it bind to a different, larger PV if you edit it? What are the conditions for growing it in place?
7. What's the difference between `volumeBindingMode: Immediate` and `WaitForFirstConsumer`, and why does it matter for local or zonal storage?
8. A pod is Pending with `pod has unbound immediate PersistentVolumeClaims`. List your next three commands.

<details>
<summary>Answers</summary>

1. No. Admission fills in the default class, so the PVC's class doesn't equal `manual` and it waits for or gets a dynamic volume. Set `storageClassName: manual` on the PVC. (Capacity 1Gi ≥ 500Mi and the access modes match, so it then binds, with a capacity of 1Gi.)
2. RWO: read-write, mounted by a single **node**. ROX: read-only, many nodes. RWX: read-write, many nodes. RWOP: read-write by exactly **one pod** (CSI only, GA v1.29). Yes, with RWO two pods on the **same** node can both mount and write it. RWOP is the mode that prevents that.
3. `waiting for first consumer to be created before binding`. The class uses `WaitForFirstConsumer`, so nothing is provisioned until a pod that uses the PVC is scheduled. It's normal. Create the pod. If a pod exists and the event is `waiting for a volume to be created ... by external provisioner "rancher.io/local-path"`, check the provisioner pod in `local-path-storage`.
4. `Released`. A new PVC **can't** bind while `spec.claimRef` still points at the old claim. Remove it: `k patch pv <pv> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'`. The PV goes `Available` with the old data still on it (clean it first if the task says so).
5. ```bash
   k patch sc local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
   k patch sc fast -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   k get sc     # only fast shows (default)
   ```
6. No. A bound PVC stays bound to its PV, and most of its spec is immutable. To grow it in place: the PVC's StorageClass must have `allowVolumeExpansion: true`, the volume must have been provisioned by a driver that supports resizing, and you can only increase `spec.resources.requests.storage`. Capacity updates when the resizer (and, for filesystems, the kubelet) finishes.
7. `Immediate` provisions and binds as soon as the PVC is created, before the scheduler has picked a node. With node-local or zonal storage, the volume can end up somewhere the pod can't run (`volume node affinity conflict`). `WaitForFirstConsumer` delays provisioning until the pod is scheduled, so the volume is created where the pod lands, taking taints, affinity and resources into account.
8. `k describe pvc <claim>` (events), `k get pv` (compare class, modes, size, status), and `k get sc` (does the class exist, is there a default, which provisioner). Then the provisioner's pods if it's a dynamic class.

</details>

---

**Done when:** every [Day 10 acceptance criterion](../stories/day-10-storage.md#acceptance-criteria) passes: PV + PVC + pod with the volume mounted at the given path in < 6 min; the mismatch behind a Pending PVC named in < 1 min; and RWO / ROX / RWX / RWOP explained without notes. Tomorrow (Day 11) is break-and-fix on nodes and the control plane, and a PVC-stuck-Pending case is one of the "pod won't start" causes you'll run through there and in both mocks.
