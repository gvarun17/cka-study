# Day 1 Lesson: How a cluster actually works, and building one with kubeadm

**Story:** [day-01-lab-setup](../stories/day-01-lab-setup.md) · **Read time:** ~35 min · **Then:** 2+ hours in the lab

On EKS, AWS runs the API server, etcd, scheduler and controller manager for you, and you never see them. On the CKA they are ordinary files and processes on a Linux box, and about half the score depends on finding and fixing them. Today's goal is to build that box yourself, so every later day has a real cluster to break.

---

## 1. The map: what runs where

```
                 CONTROL PLANE NODE (cp1)
 ┌──────────────────────────────────────────────────────────┐
 │  kubelet (systemd service)  ── reads ──►  /etc/kubernetes/manifests/
 │      │                                     ├─ etcd.yaml
 │      │ starts as static pods               ├─ kube-apiserver.yaml
 │      ▼                                     ├─ kube-controller-manager.yaml
 │  ┌────────┐   ┌──────────────┐             └─ kube-scheduler.yaml
 │  │  etcd  │◄──│ kube-apiserver│◄──── kubectl (you), every other component
 │  └────────┘   └──────────────┘
 │                 ▲          ▲
 │   ┌─────────────┴──┐  ┌────┴───────────┐
 │   │ controller-mgr │  │   scheduler    │
 │   └────────────────┘  └────────────────┘
 │  containerd (systemd service) ◄── kubelet talks CRI to it
 └──────────────────────────────────────────────────────────┘

                 WORKER NODE (worker1)
 ┌──────────────────────────────────────────────────────────┐
 │  kubelet (systemd) ──► containerd (systemd) ──► your pods
 │  kube-proxy (DaemonSet pod)    CNI agent (DaemonSet pod)
 └──────────────────────────────────────────────────────────┘
```

Two facts carry most of the troubleshooting domain:

1. **kubelet and containerd are systemd services.** If either stops, nothing on that node works, and you fix it with `systemctl` and `journalctl`. You can't fix it with `kubectl`.
2. **The four control-plane components are static pods.** The kubelet starts them straight from YAML files in `/etc/kubernetes/manifests/`. No API server is involved, which is how the API server can start at all. To fix one, you edit its file, and the kubelet recreates it automatically.

### What each component does

| Component | Job | When it breaks |
|---|---|---|
| **etcd** | Key-value store; the only place cluster state lives | Whole cluster stops; restore from snapshot (Day 2) |
| **kube-apiserver** | The front door. Authenticates, authorizes (RBAC), validates and stores objects in etcd. **The only component that talks to etcd.** | `kubectl` gets "connection refused" |
| **kube-scheduler** | Watches for pods with no `nodeName` and picks a node | New pods stay `Pending` with no scheduling events |
| **kube-controller-manager** | Runs the control loops: Deployment → ReplicaSet → Pods, node health, endpoints, ServiceAccount tokens… | Deployments don't create pods, scaling does nothing |
| **kubelet** | On every node: watches the API for pods bound to its node, tells the runtime to run them, reports status | Node goes `NotReady` |
| **containerd** | Container runtime; pulls images and runs containers. Speaks **CRI**. | Node `NotReady`, kubelet logs show runtime errors |
| **kube-proxy** | Programs iptables/IPVS so Service IPs route to pod IPs | Services don't route even though pods work |
| **CNI plugin** (Calico, Cilium…) | Gives each pod an IP and connects pods across nodes | Nodes stay `NotReady` after install; pods can't reach each other |
| **CoreDNS** | Cluster DNS (`svc.cluster.local`) | Names don't resolve; IPs still work |

### Follow one request end to end

`kubectl create deploy web --image=nginx --replicas=2`:

1. kubectl reads `~/.kube/config` and sends an HTTPS POST to the API server with your client cert.
2. The API server authenticates the cert, checks RBAC, runs admission, and writes the Deployment to **etcd**.
3. The **controller manager's** Deployment controller sees the new Deployment and creates a ReplicaSet. The ReplicaSet controller creates 2 Pods with no `nodeName`.
4. The **scheduler** sees unscheduled pods, filters nodes (taints, resources, affinity), scores them, and writes `nodeName` on each pod.
5. The **kubelet** on that node sees a pod bound to it, asks **containerd** to pull and run the image, and calls the **CNI** plugin to assign an IP.
6. The kubelet reports the pod as `Running` back to the API server, and the API server stores the status in etcd.

Every component talks only to the API server; none of them talk to each other directly. Learn this chain, because troubleshooting means finding which link broke. Pod `Pending` means step 4 failed. `ContainerCreating` for a long time means step 5 failed (image or CNI). A Deployment with 0 ReplicaSets means step 3 failed.

---

## 2. What `kubeadm init` actually does

kubeadm is a bootstrapper, not a process that keeps running. It writes files, starts things, and exits. It runs in **phases**, and knowing them tells you where every file came from:

| Phase | What it creates |
|---|---|
| `preflight` | Checks swap is off, ports are free, runtime is reachable, kernel modules loaded |
| `certs` | A CA plus every cert, in `/etc/kubernetes/pki/` (etcd has its own CA in `pki/etcd/`) |
| `kubeconfig` | `admin.conf`, `super-admin.conf`, `kubelet.conf`, `controller-manager.conf`, `scheduler.conf` in `/etc/kubernetes/` |
| `kubelet-start` | `/var/lib/kubelet/config.yaml`, then starts the kubelet |
| `control-plane` + `etcd` | The four static pod manifests in `/etc/kubernetes/manifests/` |
| `upload-config` | Stores the cluster config in the `kubeadm-config` ConfigMap in `kube-system` |
| `mark-control-plane` | Adds the `node-role.kubernetes.io/control-plane:NoSchedule` taint, which is why your pods don't land on cp1 |
| `bootstrap-token` | A token that workers use to join |
| `addon` | CoreDNS (Deployment) and kube-proxy (DaemonSet) |

Each phase can also be run alone (`kubeadm init phase certs all`, for example). That's useful later; you don't need it today.

### The file map: memorize this

```
/etc/kubernetes/
├── manifests/            ← static pods: edit here to fix control-plane components
├── pki/                  ← certs (apiserver.crt, ca.crt, …)
│   └── etcd/             ← etcd's own CA + server/peer certs (Day 2)
├── admin.conf            ← kubeconfig for cluster-admin
└── kubelet.conf          ← kubeconfig the kubelet uses
/var/lib/kubelet/config.yaml   ← kubelet settings (staticPodPath, cgroupDriver, clientCAFile)
/var/lib/etcd/                 ← etcd data
/etc/cni/net.d/                ← CNI config
/etc/containerd/config.toml    ← runtime config
```

---

## 3. Node prerequisites, and why each exists

You run these on **both** nodes. Know the reason for each, because a broken prerequisite is a classic exam fault.

| Prerequisite | Why |
|---|---|
| Swap off | The kubelet's memory accounting assumes no swap; preflight fails otherwise (unless explicitly configured) |
| `overlay` module | containerd's overlayfs snapshotter (image layers) |
| `br_netfilter` module + `net.bridge.bridge-nf-call-iptables=1` | Lets iptables see bridged pod traffic, which kube-proxy depends on |
| `net.ipv4.ip_forward=1` | The node must route packets between pods and nodes. If this is missing, preflight fails, and on the exam it's a common "fix this node" fault. |
| **Matching cgroup driver** (`systemd`) in both containerd and kubelet | If they disagree, pods restart randomly or the kubelet fails. kubeadm sets the kubelet to `systemd`, so containerd must say `SystemdCgroup = true`. |

---

## 4. The build, step by step

**Lab:** 2 Ubuntu 24.04 VMs, each with 2 vCPU and 4 GB RAM.
- **Multipass (local):** `multipass launch 24.04 -n cp1 -c 2 -m 4G -d 20G` and the same for `worker1`.
- **EC2:** t3.medium, both in one security group that allows **all traffic from itself**, plus SSH from your IP. Turn off source/dest check if you use Calico without encapsulation.

Pick a lab version **one minor below the exam's** (exam is listed as v1.35, so build v1.34). Day 3's upgrade then has a real target.

### 4a. On both nodes

```bash
# Swap off (now and after reboot)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

# Sysctls
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# containerd with the systemd cgroup driver
sudo apt-get update && sudo apt-get install -y containerd apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

# Kubernetes packages (v1.34 repo; Day 3 upgrades to v1.35)
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

The kubelet will now crash-loop every few seconds. **That's expected:** it has no config until `kubeadm init` or `join` writes one. Check with `journalctl -u kubelet -n 20`, and learn what the "normal before init" error looks like so it doesn't confuse you later.

### 4b. On cp1 only

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<cp1-private-ip>

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes         # cp1 NotReady: no CNI yet. That's expected.
```

Save the `kubeadm join ...` line it prints. If you lose it, run:

```bash
kubeadm token create --print-join-command
```

### 4c. Install a CNI (on cp1)

Use Calico, following its "Install Calico networking for on-premises deployments" page. The pattern is the Tigera operator plus a custom resource whose pod CIDR matches the `--pod-network-cidr` you used (`192.168.0.0/16` is Calico's default, which is why we chose it). Check that page for the current release number:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/<VERSION>/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/<VERSION>/manifests/custom-resources.yaml
watch kubectl get pods -A      # wait for calico-* and coredns to be Running
```

**Why the node was NotReady:** the kubelet reports `NetworkReady=false` until a CNI config appears in `/etc/cni/net.d/`. CoreDNS also stays `Pending` until then. This is a classic exam scenario: "the node is NotReady, fix it." It has three usual causes: CNI missing, kubelet down, or runtime down.

### 4d. On worker1

```bash
sudo kubeadm join <cp1-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

What happens: the worker uses the token to authenticate once, checks the CA hash so it knows it's talking to the real cluster, then uses **TLS bootstrapping**. The kubelet submits a CSR, which is auto-approved, and gets its own client cert. After that the token isn't needed.

Back on cp1:

```bash
kubectl get nodes -o wide      # both Ready
kubectl get pods -A -o wide    # see what runs where
```

### 4e. Snapshot now

Take a VM snapshot (`multipass snapshot cp1` / `worker1`, or AMIs on EC2). You'll break this cluster on Days 2, 3 and 11, and a 2-minute restore beats a 30-minute rebuild.

---

## 5. Look around: build the mental model

Run each of these and connect the output to section 1:

```bash
sudo ls /etc/kubernetes/manifests/
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E -- '--(etcd|client-ca|tls|service-cluster)'
kubectl get pods -n kube-system -o wide       # static pods are named <component>-cp1
sudo crictl ps                                # same containers, seen from the runtime
sudo grep -E 'staticPodPath|cgroupDriver|clientCAFile' /var/lib/kubelet/config.yaml
kubectl describe node cp1 | grep -A3 Taints
systemctl status kubelet containerd
```

Then try this experiment:

```bash
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
kubectl get pods -n kube-system | grep scheduler    # gone within ~20s
kubectl run test --image=nginx
kubectl get pod test                               # Pending forever: no scheduler
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
kubectl get pod test -w                            # scheduled and Running
kubectl delete pod test
```

You just ran the whole static-pod lifecycle and a scheduler outage. Day 11 is this, ten times harder.

---

## 6. Speed: the other half of Day 1

The exam doesn't fail most people on knowledge. It fails them on time. Set up [speed-setup](../cheatsheets/speed-setup.md) on cp1 now, then drill:

**Generate, don't write.** Every object `kubectl create` supports should start from `$do`:

```bash
k run web --image=nginx $do > pod.yaml           # then vim to add what's missing
k create deploy web --image=nginx -r 3 $do
k create cm cfg --from-literal=a=1 $do
k create secret generic s --from-literal=p=x $do
k create job j --image=busybox $do -- echo hi
k create cronjob c --image=busybox --schedule="*/1 * * * *" $do -- date
k expose deploy web --port 80 $do
```

**`kubectl explain` is faster than the docs** for field names:

```bash
k explain pod.spec.containers.livenessProbe
k explain deploy.spec.strategy --recursive
```

**Docs navigation.** In the exam you get one browser tab of kubernetes.io/docs. Use the docs search, and practice reaching these pages fast: kubeadm install, kubeadm upgrade, etcd backup, NetworkPolicy, PV/PVC, Ingress, Gateway API, RBAC, and the kubectl quick reference.

---

## 7. Traps that cost points

- **Leaving `kubectl delete` to wait.** Use `$now` (`--force --grace-period 0`) for pods; it saves 30 seconds each time.
- **Editing a running pod's immutable fields.** `k edit pod` refuses most spec changes. Instead, `k get pod x -o yaml > x.yaml`, edit it, then `k replace --force -f x.yaml`.
- **Tabs in YAML.** That's why you set `expandtab` in vimrc.
- **Forgetting the namespace.** If the question says `-n project-x`, every command needs it, including `get` when you verify.
- **Ignoring "on node X."** If the task is on a node, `ssh` there first, and `exit` afterward.

---

## 8. Self-check: answer before you open the lab

Write the answers in the story's Notes section, then check them below.

1. Which component is the only one that talks to etcd?
2. A new pod stays `Pending` and `describe` shows **no events at all**. Which component is likely down, and where do you look?
3. `kubectl get nodes` shows worker1 `NotReady`. What are the first two commands you run, and on which machine?
4. Why does the API server run as a static pod rather than a normal Deployment?
5. You deleted the `kube-apiserver` pod with `kubectl delete pod`. What happens?
6. Containerd uses the `cgroupfs` driver and the kubelet uses `systemd`. What symptoms would you expect?
7. Why don't your normal pods get scheduled on cp1?

<details>
<summary>Answers</summary>

1. **kube-apiserver.** Everything else goes through it.
2. **kube-scheduler.** With no scheduler, nothing even tries to place the pod, so there are no `FailedScheduling` events. Check `k get pods -n kube-system`, then `/etc/kubernetes/manifests/kube-scheduler.yaml`, and `crictl ps -a | grep scheduler` plus its logs.
3. `ssh worker1`, then `systemctl status kubelet` and `journalctl -u kubelet -n 50`. Also check `systemctl status containerd`. If kubelet and containerd are both healthy, check whether the CNI pod on that node is running.
4. Because it must run **before** any API exists. The kubelet can start static pods from local files without asking an API server. A Deployment would need the API server to already be running.
5. It comes straight back. That object is only a "mirror pod" the kubelet reports. The real source is the manifest file, and the kubelet recreates the pod from it. To stop a static pod, you move its file out of the manifests directory.
6. Unstable nodes: pods restart repeatedly, the kubelet may fail to start or log cgroup errors, and control-plane pods crash-loop. The fix is `SystemdCgroup = true` in `config.toml`, then restart containerd and the kubelet.
7. kubeadm's `mark-control-plane` phase added the `node-role.kubernetes.io/control-plane:NoSchedule` taint, and your pods have no matching toleration.

</details>

---

**Done when:** every [Day 1 acceptance criterion](../stories/day-01-lab-setup.md#acceptance-criteria) passes, and you can answer the 7 questions without peeking. Tomorrow (Day 2) builds directly on section 2's file map: certificates, static pods in depth, and etcd backup/restore.
