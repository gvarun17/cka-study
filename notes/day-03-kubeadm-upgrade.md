# Day 3 Lesson: Upgrading a kubeadm cluster, node by node

**Story:** [day-03-kubeadm-upgrade](../stories/day-03-kubeadm-upgrade.md) · **Read time:** ~35 min · **Then:** 2 hours in the lab

**Why this matters:** Cluster Architecture, Installation & Configuration is 25% of the exam, and "upgrade the control plane and worker to version X" is one of the most common CKA tasks. It's often one of the highest-weighted questions, it takes 10–15 minutes if you know the sequence, and it's easy to half-finish: kubeadm upgraded but kubelet not, or a worker left cordoned. The verifier checks the end state, so a half-finished upgrade can score zero.

On EKS you click "Update" (or bump `version` in Terraform), AWS replaces the control plane, and managed node groups roll new AMIs. Here **you** are the control-plane service: one binary does the cluster-wide work, then you upgrade every kubelet yourself.

> **Version note.** The current kubernetes.io [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) page covers **1.36.x → 1.37.x**, which matches the exam's v1.37 (see Day 2). For your v1.34 lab, use the versioned page for [1.34 → 1.35](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/). The procedure is the same; only the numbers change. Both pages say **skipping minor versions is unsupported**: 1.34 → 1.36 means two upgrades.
>
> Two v1.35 changes worth knowing (you won't hit them on Ubuntu 24.04): the kubelet now **refuses to start on cgroup v1 hosts** by default (`failCgroupV1: true`; check with `stat -fc %T /sys/fs/cgroup` → `cgroup2fs` is good), and v1.35 is the last minor that supports **containerd 1.x**. Ubuntu's `containerd` package is 1.7, so plan a containerd 2.x move before you take the lab past 1.35. Neither is an exam topic.

---

## 1. What "upgrading the cluster" actually changes

A kubeadm cluster has versioned parts in three places, and each is upgraded by a different thing:

| Part | Where it lives | Upgraded by |
|---|---|---|
| `kubeadm` binary | apt package on every node | `apt-get install kubeadm=…` (you) |
| kube-apiserver, controller-manager, scheduler, etcd | Static pod manifests in `/etc/kubernetes/manifests/` on the control plane | `kubeadm upgrade apply` (rewrites the image tags) |
| CoreDNS, kube-proxy | Deployment / DaemonSet in `kube-system` | `kubeadm upgrade apply` (addons phase) |
| Cluster config | ConfigMaps `kubeadm-config`, `kubelet-config` in `kube-system` | `kubeadm upgrade apply` |
| kubelet config on each node | `/var/lib/kubelet/config.yaml` | `kubeadm upgrade apply` (first CP) / `kubeadm upgrade node` (every other node) |
| `kubelet` binary | apt package + systemd unit on every node | `apt-get install kubelet=…` + `systemctl restart kubelet` (you) |
| `kubectl` | apt package | `apt-get install kubectl=…` (you) |
| CNI (Calico) | Its own operator/DaemonSet | **Nobody.** kubeadm never touches the CNI. Upgrade it separately if its docs say the new version needs it. |

The single most important fact: **`kubectl get nodes` shows the kubelet version**, not the control plane version. Right after `kubeadm upgrade apply` succeeds, cp1 still says `v1.34.x`. It changes only when you upgrade and restart the kubelet. That's why "kubeadm said SUCCESS but the node shows the old version" is the classic half-finished upgrade.

### Order, and why

```
            ┌────────── control plane first ──────────┐   ┌──── then each worker ────┐
cp1:  kubeadm pkg → upgrade plan → upgrade apply → drain → kubelet+kubectl pkg → restart → uncordon
worker1:                                            drain → kubeadm pkg → upgrade node → kubelet pkg → restart → uncordon
```

The [version skew policy](https://kubernetes.io/releases/version-skew-policy/) makes this order mandatory:

- **kube-apiserver must be the newest component.** A kubelet may be up to **three minors older** than the API server, but never newer. So the control plane goes first, kubelets after.
- **kubectl** is supported within one minor of the API server either way.
- **controller-manager / scheduler** may not be newer than the API server. kubeadm handles this by upgrading the API server before them.

### What `kubeadm upgrade apply v1.35.x` does

Run once per cluster, on the first control-plane node:

1. **Preflight:** checks that the cluster is healthy (API server reachable, nodes Ready, control-plane pods running), that the target is allowed (one minor at most, not newer than the kubeadm binary), and pulls the new images.
2. **Backups:** copies the current manifests to `/etc/kubernetes/tmp/kubeadm-backup-manifests-<date>/` and, for a local etcd that's being upgraded, the etcd data to `/etc/kubernetes/tmp/kubeadm-backup-etcd-<date>/`.
3. **Static pods, one at a time:** etcd (if its version changes), then kube-apiserver, kube-controller-manager, kube-scheduler. For each one it writes a new manifest and waits for the kubelet to restart that pod with the new hash. If a component doesn't come back, it **rolls back** to the backup manifest and fails.
4. **Certificates:** renews the control-plane leaf certs (Day 2) unless you pass `--certificate-renewal=false`.
5. **Config:** uploads the new `ClusterConfiguration` to the `kubeadm-config` ConfigMap and the new kubelet config to `kubelet-config`, then writes `/var/lib/kubelet/config.yaml` on this node.
6. **RBAC + addons:** refreshes bootstrap-token RBAC rules, then upgrades **CoreDNS** and **kube-proxy**. On a multi-control-plane cluster the addons wait until the last control-plane node has run `kubeadm upgrade node`.

It **doesn't** touch the kubelet binary, kubectl, the CNI, or any worker.

### What `kubeadm upgrade node` does (and why the worker doesn't run `apply`)

`kubeadm upgrade node` is the per-node half of the upgrade. It reads the cluster's config from the `kubeadm-config` ConfigMap, which `apply` already updated, and:

- **On a worker:** rewrites `/var/lib/kubelet/config.yaml` from the new `kubelet-config` ConfigMap. That's all.
- **On an additional control-plane node:** also upgrades that node's static pod manifests, local etcd member, and certs.

A worker can't run `apply` because `apply` is the **cluster-wide** step. It needs the static pod manifests, the PKI, local etcd and `admin.conf` credentials, none of which a worker has. And it has already been done: the cluster config and addons were upgraded once on cp1. Running `apply` again elsewhere is wrong both in principle (it's not a per-node step) and in practice (it fails preflight). That's your acceptance-criterion answer: **`apply` upgrades the cluster once; `node` syncs one node to the already-upgraded cluster.**

---

## 2. Packages: pkgs.k8s.io has one repo per minor

The community repo `pkgs.k8s.io` (the old `apt.kubernetes.io` is gone) is split **per minor version**. Your `v1.34` repo will never show a 1.35 package, so the first step of every minor upgrade is to switch the repo ([Changing the package repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)):

```bash
cat /etc/apt/sources.list.d/kubernetes.list
# deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
sudo sed -i 's#/v1.34/#/v1.35/#' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm | head -3
#   kubeadm | 1.35.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
```

The package version is `1.35.3-1.1`; the version you pass to `kubeadm upgrade apply` is `v1.35.3`. Mixing those two formats up is a common typo.

A **patch** upgrade (1.35.1 → 1.35.3) needs no repo change.

On the exam, the repo is usually already pointed at the right minor and sometimes kubeadm is already installed at the target. Check before you change anything: `kubeadm version -o short; apt-cache madison kubeadm | head -2`.

### Hold, unhold, or skip the dance

The packages are held so a routine `apt upgrade` can't bump them. Two equivalent ways to install the new version:

```bash
# The docs' way
sudo apt-mark unhold kubeadm && sudo apt-get install -y kubeadm='1.35.3-*' && sudo apt-mark hold kubeadm

# Faster: install through the hold (the package stays held afterwards)
sudo apt-get install -y --allow-change-held-packages kubeadm='1.35.3-*'
```

Use whichever you can type without thinking. Verify with `apt-mark showhold` and `kubeadm version -o short`.

---

## 3. Drain: what it does and which flags you need

`kubectl drain` = **cordon** (set `spec.unschedulable: true`, which adds the `node.kubernetes.io/unschedulable:NoSchedule` taint) + **evict** every pod on the node through the **Eviction API**, so PodDisruptionBudgets are respected. `kubectl uncordon` clears the flag. Drain never deletes pods it isn't sure about; instead it refuses and tells you which flag would make it proceed:

| Flag | Needed when | What happens to those pods |
|---|---|---|
| `--ignore-daemonsets` | **Always**, in practice. calico-node and kube-proxy are DaemonSets on every node. | Left running. DaemonSet pods tolerate the unschedulable taint, so evicting them is pointless; the controller would recreate them on the same node. |
| `--delete-emptydir-data` | Any pod uses an `emptyDir` volume (metrics-server and some Calico components do). | Evicted, and the emptyDir data is deleted with the pod. |
| `--force` | A pod has **no controller** (bare `k run` pod). | Deleted for good. Nothing recreates it. |
| `--disable-eviction` | A PDB blocks eviction and the task says to proceed anyway. | Pods are deleted directly, bypassing PDBs. Use only if told. |
| `--timeout=120s` | You don't want drain to hang on a PDB forever. | Drain gives up after the timeout. |
| `--grace-period=N` | Pods take too long to terminate. | Overrides `terminationGracePeriodSeconds`. |

Static (mirror) pods are skipped automatically. The control-plane pods on cp1 keep running while cp1 is drained, and that's what you want.

The exam-safe default is `k drain <node> --ignore-daemonsets`. If it errors, **read the error**. It names the exact pods and the flag it needs. Add `--delete-emptydir-data` or `--force` only after you've seen which pods they affect.

**PDBs:** if eviction would violate a PodDisruptionBudget, the API returns 429 and drain keeps retrying (`error when evicting pods ... Cannot evict pod as it would violate the pod's disruption budget`). With one worker, a PDB with `minAvailable: 1` on a 1-replica Deployment can never be satisfied. That's an exam scenario: scale the Deployment, relax the PDB, or use `--disable-eviction` if the question allows it.

**Two-node lab note:** when you drain worker1, cp1 still has its control-plane `NoSchedule` taint, so evicted pods (CoreDNS included) go `Pending` until you uncordon. That's expected in a two-node lab. On the exam there's usually more than one worker.

---

## 4. The exam procedure

Paste nothing. Type it, and type the version the question names, exactly.

### Control plane (target: under 10 minutes)

```bash
ssh cp1                     # if the task gives a host; then check
hostname; sudo -i           # root saves typing sudo 15 times (exit twice later)

# 1. Repo + kubeadm
sed -i 's#/v1.34/#/v1.35/#' /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-cache madison kubeadm | head -3
apt-get install -y --allow-change-held-packages kubeadm='1.35.3-*'
kubeadm version -o short                      # v1.35.3

# 2. Plan, then apply
kubeadm upgrade plan                          # read the target line and any "manual upgrade required" rows
kubeadm upgrade apply v1.35.3                 # answer y; ends with "[upgrade] SUCCESS!" (2–5 min)

# 3. Drain cp1 (kubectl works from cp1 as root via /etc/kubernetes/admin.conf, or as your user)
kubectl drain cp1 --ignore-daemonsets

# 4. kubelet + kubectl
apt-get install -y --allow-change-held-packages kubelet='1.35.3-*' kubectl='1.35.3-*'
systemctl daemon-reload && systemctl restart kubelet

# 5. Bring it back
kubectl uncordon cp1
kubectl get nodes                             # cp1 v1.35.3 Ready
```

If `kubectl` as root says `connection refused localhost:8080`, root has no kubeconfig: `export KUBECONFIG=/etc/kubernetes/admin.conf`, or exit root and use your user.

### Worker (target: under 6 minutes)

```bash
# From cp1 (or wherever kubectl works)
k drain worker1 --ignore-daemonsets

ssh worker1
sudo -i
sed -i 's#/v1.34/#/v1.35/#' /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y --allow-change-held-packages kubeadm='1.35.3-*'
kubeadm upgrade node                          # NOT apply. "[upgrade] The configuration for this node was successfully updated!"
apt-get install -y --allow-change-held-packages kubelet='1.35.3-*' kubectl='1.35.3-*'
systemctl daemon-reload && systemctl restart kubelet
exit; exit                                    # root, then ssh. Check the prompt!

k uncordon worker1
k get nodes                                   # both v1.35.3 Ready, no SchedulingDisabled
```

### Useful extras

```bash
kubeadm upgrade plan v1.35.3        # plan for a specific target
kubeadm upgrade apply v1.35.3 --dry-run   # nothing written; shows what would change
kubeadm upgrade diff v1.35.3        # manifest diffs only
kubectl get nodes -o wide           # kubelet + container runtime versions
kubectl version                     # client vs server
kubectl -n kube-system get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

`kubeadm upgrade` also accepts a config file (`--config`) holding an `UpgradeConfiguration` (`kubeadm.k8s.io/v1beta4`) where you can set `certificateRenewal`, `etcdUpgrade` and similar for `apply` and `node`. You won't need it on the exam. The flags cover everything.

### If `apply` fails

- It rolls the failing component back to the backup manifest automatically. Fix the cause (usually an unhealthy pod or a preflight error the output names), then **rerun `kubeadm upgrade apply`**. It's safe to run again.
- Manual rollback: the previous manifests are in `/etc/kubernetes/tmp/kubeadm-backup-manifests-*/`. Copy them back into `/etc/kubernetes/manifests/`, and use the Day 2 restore if etcd is involved.
- If a preflight check fails over something you've judged harmless, `--ignore-preflight-errors=<CheckName>` skips it. Don't reach for it first.

---

## 5. Hands-on walkthrough on cp1 + worker1 (v1.34 → v1.35)

**Step 0: snapshot the VMs (task 5).** This makes the upgrade repeatable. Multipass needs the instances stopped:

```bash
# on your laptop
multipass stop cp1 worker1
multipass snapshot cp1 --name v134 && multipass snapshot worker1 --name v134
multipass start cp1 worker1
# later, to repeat the drill:
multipass stop cp1 worker1 && multipass restore cp1.v134 --destructive && multipass restore worker1.v134 --destructive
```

(On EC2: AMIs or EBS snapshots of both instances.) If you can't snapshot, rebuilding at v1.34 means rerunning the Day 1 install with the `v1.34` repo and `kubeadm init`. Snapshots are much faster.

**Step 1: etcd backup and restore from memory (task 4).** Do it before the upgrade, the way you would in production. Timed, Day 2 style, no notes. Target **under 10 minutes** for save + status + restore + hostPath edit + API back:

```bash
k create ns pre-upgrade
# ⏱ start
#   etcdctl snapshot save /opt/pre-upgrade.db (certs from etcd.yaml)
#   etcdutl snapshot status
#   k create ns throwaway
#   etcdutl snapshot restore → /var/lib/etcd-restore, edit the etcd-data hostPath, wait
# ⏱ stop when `k get ns` shows pre-upgrade and not throwaway
```

If you needed the Day 2 notes, that's a failed criterion. Write the step you forgot into today's story Notes.

Then **retake the VM snapshot** so the upgrade starts from a clean, restored cluster (or leave it; `kubeadm upgrade` works fine on a restored etcd).

**Step 2: give drain something to do.**

```bash
k create deploy web --image=nginx --replicas=2
k run bare --image=nginx                       # no controller → drain needs --force
k create deploy scratch --image=nginx --dry-run=client -o yaml > scratch.yaml
# add an emptyDir volume to scratch.yaml (volumes: - name: tmp, emptyDir: {} + a volumeMount), then
k apply -f scratch.yaml
k create pdb web-pdb --selector=app=web --min-available=2
k get pods -o wide                             # note which pods are on worker1
```

**Step 3: upgrade cp1 (task 1).** ⏱ Start the timer and follow section 4's control-plane block. While `apply` runs, watch from a second terminal:

```bash
watch -n2 'sudo crictl ps --name "kube-|etcd" -o table | head; ls /etc/kubernetes/tmp/'
```

You'll see each control-plane container replaced one at a time, and a backup directory appear. After `apply` and **before** the kubelet step, run `k get nodes`: cp1 still says v1.34. Confirm it with your own eyes once. ⏱ Stop at `uncordon`. Target **< 10 min**.

**Step 4: drain worker1, and hit each flag (task 3).**

```bash
k drain worker1                                       # cordons, then lists every blocker: DaemonSet pods, bare (no controller), local storage (emptyDir)
k drain worker1 --ignore-daemonsets                   # still refuses: bare → needs --force; scratch → needs --delete-emptydir-data
k drain worker1 --ignore-daemonsets --force --delete-emptydir-data --timeout=60s
# now it loops on: Cannot evict pod as it would violate the pod's disruption budget
# (web needs 2 available, but evicted replicas can't land anywhere)
k delete pdb web-pdb                                  # or: k patch pdb web-pdb -p '{"spec":{"minAvailable":1}}'
k drain worker1 --ignore-daemonsets --force --delete-emptydir-data
k get pods -o wide                                    # bare is gone for good; web/scratch Pending
```

Write one line per flag in the story Notes: when it's needed and what happens to the pods.

**Step 5: upgrade worker1 (task 2).** ⏱ Section 4's worker block, from `ssh worker1` to `uncordon`. Target **< 6 min**. Then:

```bash
k get nodes                       # both v1.35.x Ready
k get pods -A -o wide | grep -v Running      # everything should settle to Running
k -n kube-system get ds kube-proxy -o jsonpath='{..image}{"\n"}'   # v1.35.x
```

**Step 6: repeat.** Restore both snapshots and do steps 3 and 5 again with the timer, without looking at this page. The second run is the one that counts for the acceptance criteria.

---

## 6. Traps that cost points

- **Not reading the target version.** "Upgrade to 1.35.2" when 1.35.3 exists means `1.35.2`, not the latest that `plan` suggests.
- **kubeadm upgraded, kubelet not.** `kubectl get nodes` shows the kubelet version, and the verifier usually checks exactly that. Always finish with `k get nodes`.
- **Forgetting `systemctl daemon-reload && systemctl restart kubelet`.** The new binary is on disk, but the old process keeps running and reporting the old version.
- **`kubeadm upgrade apply` on a worker.** Workers run `kubeadm upgrade node`.
- **Upgrading the worker's kubelet without `kubeadm upgrade node` first.** The new kubelet may start with a config written for the old version. Sequence: kubeadm pkg → `upgrade node` → kubelet pkg → restart.
- **Leaving a node cordoned.** `SchedulingDisabled` in `k get nodes` is an incomplete task. The uncordon is part of the answer.
- **Running drain on the worker itself.** On the exam, workers often have no kubeconfig. Drain and uncordon from wherever `kubectl` works (usually the control plane or the base host).
- **Forgetting the repo switch.** `apt-cache madison` shows only 1.34 packages. Change the minor in `kubernetes.list`, then `apt-get update`.
- **`kubeadm upgrade apply 1.35.3-1.1`.** That's the package version. `apply` takes `v1.35.3`.
- **Not exiting.** After `sudo -i` + `ssh` you need two `exit`s. Check `hostname` before the next question.
- **Throwing `--force` on blindly.** On the exam it deletes bare pods the task may want kept. Read drain's error first.
- **Upgrading two minors at once.** Not supported. One minor per `apply`.

---

## 7. Self-check: answer before you open the lab

1. `kubeadm upgrade apply` printed SUCCESS on cp1, but `k get nodes` still shows cp1 at v1.34. Is the upgrade broken? What's left?
2. Why does worker1 run `kubeadm upgrade node` and not `kubeadm upgrade apply`? What does `upgrade node` change on a worker?
3. `apt-cache madison kubeadm` shows only 1.34.x versions. What did you skip?
4. Why must the control plane be upgraded before the kubelets? Which policy says so, and how far behind may a kubelet be?
5. `k drain worker1 --ignore-daemonsets` refuses because of a pod named `bare` and a pod with an `emptyDir`. Which flags fix each, and what happens to those pods?
6. Drain hangs printing "would violate the pod's disruption budget". Name two ways forward.
7. Which components does `kubeadm upgrade apply` upgrade, and which three things must you still do yourself on cp1?
8. `apply` failed halfway through. Where are the previous manifests, and what's the first thing to try?

<details>
<summary>Answers</summary>

1. Not broken. `kubectl get nodes` reports the **kubelet** version, and `apply` doesn't touch the kubelet. Still to do: drain cp1, install the new kubelet and kubectl, `systemctl daemon-reload && systemctl restart kubelet`, uncordon.
2. `apply` is the one-time, cluster-wide step: it upgrades the control-plane static pods, etcd, certs, the `kubeadm-config`/`kubelet-config` ConfigMaps and the CoreDNS/kube-proxy addons. That's already done, and a worker doesn't have the manifests, PKI or admin credentials to do it. `upgrade node` syncs one node to the upgraded cluster: on a worker it rewrites `/var/lib/kubelet/config.yaml` from the new `kubelet-config` ConfigMap.
3. Switching the pkgs.k8s.io repo to the new minor (`/v1.34/` → `/v1.35/` in `/etc/apt/sources.list.d/kubernetes.list`) and then running `apt-get update`.
4. The [version skew policy](https://kubernetes.io/releases/version-skew-policy/): kube-apiserver must be the newest component; a kubelet may never be newer than the API server and may be up to three minors older.
5. `--force` for `bare` (no controller): it's deleted and nothing recreates it. `--delete-emptydir-data` for the emptyDir pod: it's evicted and its emptyDir data is lost (its controller recreates it elsewhere, empty).
6. Any two: scale the workload up so the budget can be met elsewhere; relax or delete the PDB; `--disable-eviction` (bypasses PDBs, only if the task allows); `--timeout` stops the hang but doesn't solve it.
7. It upgrades etcd (local), kube-apiserver, kube-controller-manager, kube-scheduler, CoreDNS and kube-proxy, renews certs, and updates the cluster config. You still: upgrade the kubelet (+ daemon-reload/restart), upgrade kubectl, and drain/uncordon. The CNI is never kubeadm's job.
8. `/etc/kubernetes/tmp/kubeadm-backup-manifests-<date>/` (and `kubeadm-backup-etcd-<date>/` for etcd). kubeadm already rolled back the failing component, so fix the reported cause and **rerun `kubeadm upgrade apply`**.

</details>

---

**Done when:** every [Day 3 acceptance criterion](../stories/day-03-kubeadm-upgrade.md#acceptance-criteria) passes, timed and without notes. Tomorrow (Day 4, RBAC and kubeconfig) builds on the PKI and `admin.conf` you've now upgraded and renewed: Roles, bindings and ServiceAccounts verified with `k auth can-i`, kubeconfig contexts, and optionally issuing a user cert through the CSR API.
