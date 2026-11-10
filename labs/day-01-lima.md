# Day 1 lab: build the cluster on a Mac with Lima

**What you end up with:** a two-node kubeadm cluster, `cp1` (control plane) and `worker1`, running Kubernetes **v1.34** with Calico, on your Apple Silicon Mac. It's the lab that every later day uses. **Time:** about 1.5 hours.

**Machine labels.** Every command block says where to run it:

| Label | Where |
|---|---|
| **[Mac]** | macOS Terminal |
| **[cp1]** | Inside cp1: `limactl shell --workdir / cp1`, then `cd ~` |
| **[worker1]** | Inside worker1: `limactl shell --workdir / worker1`, then `cd ~` (or `ssh worker1` from cp1 once step 4 is done) |
| **[both]** | The same block on cp1 **and** on worker1 |

Each step has three parts: **Goal** (what you're achieving and why), **Do** (the commands), and **Check** (what you should see before moving on). Don't go to the next step until the check passes.

---

## Step 0: Check your Mac (2 min)

**Goal:** confirm the Mac can run two 4 GB VMs with Apple's virtualization framework.

**Do** **[Mac]**

```bash
uname -m                     # arm64
sw_vers -productVersion      # 13 or newer (Lima's vz driver needs macOS 13+)
sysctl -n hw.memsize | awk '{print $1/1073741824 " GB"}'   # 16 GB or more is comfortable
df -h ~ | tail -1            # at least ~60 GB free
brew --version               # Homebrew installed (https://brew.sh if not)
```

**Check:** `arm64`, macOS 13+, 16 GB+ RAM, enough disk, `brew` works.

---

## Step 1: Install Lima (2 min)

**Goal:** get `limactl`, the command-line tool that creates and manages the VMs.

**Do** **[Mac]**

```bash
brew install lima
limactl --version            # 2.x
```

**Check:** `limactl --version` prints a 2.x version.

---

## Step 2: Create the two VMs (10 min)

**Goal:** two Ubuntu 24.04 VMs with 2 CPUs, 4 GB RAM and 30 GB disk each, on a shared network so they can reach each other.

Why these flags:

- `template:ubuntu-24.04` is plain Ubuntu 24.04, the same OS as the lessons.
- `--network=lima:user-v2` puts both VMs on one network (192.168.104.0/24) where they can reach each other. Lima's default network gives every VM the same IP and no VM-to-VM path, which won't work for a cluster.
- `--containerd=none` stops Lima installing its own containerd. You install containerd yourself in step 6, as in the exam. Lima's copy would sit earlier in `PATH` and break step 6.
- `--mount-none` stops Lima sharing your Mac's home folder into the VMs. The VMs stay clean, like exam nodes.

**Do** **[Mac]**

```bash
for vm in cp1 worker1; do
  limactl create --name=$vm --cpus=2 --memory=4 --disk=30 \
    --network=lima:user-v2 --containerd=none --mount-none --tty=false \
    template:ubuntu-24.04
done
limactl start cp1
limactl start worker1
limactl list
```

The first start downloads the Ubuntu image (~600 MB), so it can take a few minutes.

**Check:** `limactl list` shows `cp1` and `worker1` with STATUS `Running`.

Now find each VM's IP:

```bash
limactl shell --workdir / cp1     ip -4 -o addr show | grep 192.168.104
limactl shell --workdir / worker1 ip -4 -o addr show | grep 192.168.104
```

Write both addresses down (e.g. `192.168.104.2` and `192.168.104.3`). The rest of this lab calls them **CP_IP** and **W1_IP**. Also note the interface name in that output (usually `eth0`).

**If it fails:** `limactl start` errors about the network → run `brew upgrade lima` and retry. Status stays `Broken` → `limactl delete -f <vm>` and rerun the create loop for it.

---

## Step 3: Set the hostnames (5 min)

**Goal:** name the VMs `cp1` and `worker1`, and make the names stick across reboots. Lima names them `lima-cp1` and `lima-worker1`. Kubernetes registers each node under its hostname, and every lesson uses `cp1` and `worker1`. If the hostname changed back after a reboot, the kubelet would try to register as a different node and the node would look broken.

**Do** **[cp1]**

```bash
sudo hostnamectl set-hostname cp1
```

**[worker1]**

```bash
sudo hostnamectl set-hostname worker1
```

**[both]** stop cloud-init resetting the hostname at boot, and add both names to `/etc/hosts` (use your real IPs):

```bash
echo 'preserve_hostname: true' | sudo tee /etc/cloud/cloud.cfg.d/99-keep-hostname.cfg

HOSTS='192.168.104.2 cp1
192.168.104.3 worker1'
echo "$HOSTS" | sudo tee -a /etc/hosts
# also add them to cloud-init's hosts template, in case it rewrites /etc/hosts at boot
[ -f /etc/cloud/templates/hosts.debian.tmpl ] && echo "$HOSTS" | sudo tee -a /etc/cloud/templates/hosts.debian.tmpl >/dev/null
```

**Check** **[both]**

```bash
hostname                     # cp1 / worker1
ping -c1 cp1 && ping -c1 worker1      # 1 received for both
```

Then reboot both from the Mac and check the names survived:

```bash
limactl stop cp1 && limactl stop worker1 && limactl start cp1 && limactl start worker1
limactl shell --workdir / cp1 hostname        # cp1
limactl shell --workdir / worker1 hostname    # worker1
limactl shell --workdir / cp1 ip -4 -o addr show | grep 192.168.104   # same CP_IP as before
```

**If it fails:** `ping` resolves but gets no reply → compare the IPs in `/etc/hosts` with step 2's output. Hostname went back to `lima-...` → check the `99-keep-hostname.cfg` file exists, set the hostname again, reboot again. **If an IP changed after the reboot**, update `/etc/hosts` on both VMs, and tell me before you go further: the cluster is tied to these IPs.

---

## Step 4: SSH from cp1 to worker1 (5 min)

**Goal:** `ssh worker1` from cp1 with no password. Days 2, 3, 9, 10 and 11 use it, and the exam has you SSH between nodes the same way.

**Do** **[cp1]**

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys        # so "ssh cp1" works too
```

**[Mac]** copy cp1's public key into worker1:

```bash
limactl shell --workdir / cp1 sh -c 'cat ~/.ssh/id_ed25519.pub' \
  | limactl shell --workdir / worker1 sh -c 'cat >> ~/.ssh/authorized_keys'
```

**Check** **[cp1]**

```bash
ssh -o StrictHostKeyChecking=accept-new worker1 hostname   # worker1, no password
ssh -o StrictHostKeyChecking=accept-new cp1 hostname       # cp1
```

**If it fails:** `Permission denied (publickey)` → on the Mac, compare `limactl shell --workdir / cp1 sh -c 'cat ~/.ssh/id_ed25519.pub'` with `limactl shell --workdir / worker1 sh -c 'tail -1 ~/.ssh/authorized_keys'`. They must match. Also make sure you're your normal user on cp1 (`whoami`), not root.

---

## Step 5: Node prerequisites (5 min)

**Goal:** prepare the Linux kernel on both nodes for Kubernetes:

- **swap off:** the kubelet refuses to start with swap on by default
- **`overlay` module:** containerd stores image layers with it
- **`br_netfilter` module + bridge sysctls:** lets iptables see pod traffic, which kube-proxy needs
- **`ip_forward=1`:** lets the node route packets between pods and nodes

**Do** **[both]**

```bash
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab

printf 'overlay\nbr_netfilter\n' | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

printf 'net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1\n' \
  | sudo tee /etc/sysctl.d/k8s.conf
sudo sysctl --system >/dev/null
```

**Check** **[both]**

```bash
swapon --show                                # no output
lsmod | grep -E 'overlay|br_netfilter'       # both listed
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables   # both = 1
```

---

## Step 6: Install containerd (5 min)

**Goal:** install the container runtime that the kubelet uses to run containers, and switch it to the **systemd cgroup driver**. The kubelet uses systemd. If containerd uses a different driver, pods restart randomly.

**Do** **[both]**

```bash
sudo apt-get update
sudo apt-get install -y containerd apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

**Check** **[both]**

```bash
which containerd                             # /usr/bin/containerd (not /usr/local/bin)
grep SystemdCgroup /etc/containerd/config.toml   # SystemdCgroup = true
systemctl is-active containerd               # active
stat -fc %T /sys/fs/cgroup                   # cgroup2fs
```

**If it fails:** `which containerd` shows `/usr/local/bin` → Lima installed its own containerd (the `--containerd=none` flag was missed). Delete and recreate that VM from step 2.

---

## Step 7: Install kubeadm, kubelet and kubectl v1.34 (5 min)

**Goal:** install the three Kubernetes packages from the official repo, pinned to v1.34. The exam is on v1.37. Starting lower gives Day 3 a real upgrade to practise.

- **kubeadm** builds and upgrades the cluster.
- **kubelet** is the node agent that starts pods.
- **kubectl** is the CLI.

`apt-mark hold` stops a routine `apt upgrade` from upgrading them by accident.

**Do** **[both]**

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock
```

**Check** **[both]**

```bash
kubeadm version -o short                     # v1.34.x
apt-mark showhold                            # kubeadm kubectl kubelet
systemctl status kubelet --no-pager | head -3    # "activating (auto-restart)": EXPECTED for now
```

The kubelet restarts every few seconds because it has no config yet. `kubeadm init` and `kubeadm join` give it one. Seeing this "broken before init" state once helps you recognise it later.

---

## Step 8: Create the control plane on cp1 (5 min)

**Goal:** run `kubeadm init`. It creates the certificates, writes the static pod manifests for etcd, kube-apiserver, kube-controller-manager and kube-scheduler into `/etc/kubernetes/manifests/`, and starts them through the kubelet. Then copy the admin kubeconfig so `kubectl` works as your user.

`--pod-network-cidr=10.244.0.0/16` sets the IP range for pods. It must not overlap the VM network (192.168.104.0/24). That's why this lab doesn't use Calico's default `192.168.0.0/16`, which you'll see in some lessons.

**Do** **[cp1]** (use your CP_IP)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.104.2
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

The output ends with a `kubeadm join ...` command. **Copy it somewhere.** If you lose it: `kubeadm token create --print-join-command`.

**Check** **[cp1]**

```bash
kubectl get nodes                            # cp1  NotReady  control-plane  v1.34.x  (NotReady is expected: no CNI yet)
kubectl get pods -n kube-system              # etcd, apiserver, controller-manager, scheduler Running; coredns Pending
sudo ls /etc/kubernetes/manifests/           # etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
```

**If it fails:** preflight errors name the problem (swap, ip_forward, containerd). Fix it, run `sudo kubeadm reset -f`, then run init again.

---

## Step 9: Install the Calico network plugin (10 min)

**Goal:** give pods an IP address and a network that spans both nodes. Until a CNI plugin is installed, the node stays `NotReady` and CoreDNS stays `Pending`. Calico also enforces NetworkPolicy, which Day 9 needs.

You set two values in Calico's config file:

- **The pool CIDR**, to `10.244.0.0/16`, matching step 8.
- **The encapsulation mode**, to `VXLAN`. Pod traffic between nodes is wrapped in ordinary UDP packets, which pass reliably through Lima's virtual network.

These commands follow Calico's [on-premises install page](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises) for v3.33.0 on Kubernetes 1.34/1.35.

**Do** **[cp1]**

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org-v1beta1.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml
sed -i 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#; s#encapsulation: VXLANCrossSubnet#encapsulation: VXLAN#' custom-resources.yaml
grep -nE 'cidr|encapsulation' custom-resources.yaml     # must show 10.244.0.0/16 and VXLAN
kubectl create -f custom-resources.yaml
```

If the `grep` doesn't show both values, open the file with `vim custom-resources.yaml` and set `spec.calicoNetwork.ipPools[0].cidr` and `encapsulation` by hand **before** the last command.

**Check** **[cp1]** (takes 3–5 minutes)

```bash
watch kubectl get tigerastatus               # every row AVAILABLE True, then Ctrl-C
kubectl get nodes                            # cp1 Ready
kubectl get pods -n kube-system -l k8s-app=kube-dns   # coredns Running
```

**If it fails:** `kubectl get pods -n calico-system` shows a pod stuck in `ImagePullBackOff` → check the VM's internet (`curl -I https://quay.io`). A `CrashLoopBackOff` → `kubectl -n calico-system logs <pod>` and send me the last lines.

---

## Step 10: Join worker1 (5 min)

**Goal:** add worker1 to the cluster. `kubeadm join` uses the token to authenticate once and checks the CA hash to make sure it's talking to your cluster. The kubelet then requests its own client certificate (TLS bootstrapping).

**Do** **[worker1]** paste the join command from step 8, with `sudo`:

```bash
sudo kubeadm join 192.168.104.2:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Check** **[cp1]** (1–2 minutes for Calico to start on worker1)

```bash
kubectl get nodes -o wide                    # cp1 and worker1 both Ready, v1.34.x, INTERNAL-IP = your two IPs
kubectl get pods -A -o wide | grep -v Running   # only the header line
```

Prove pods on different nodes can talk:

```bash
kubectl create deploy nettest --image=nginx:1.27 --replicas=2
kubectl get pods -o wide -l app=nettest      # both Running on worker1 (cp1 is tainted)
kubectl run t --image=busybox:1.36 --rm -it --restart=Never --overrides='{"spec":{"nodeName":"cp1"}}' \
  -- wget -qO- -T 3 $(kubectl get pod -l app=nettest -o jsonpath='{.items[0].status.podIP}') | grep -o '<title>.*</title>'
# <title>Welcome to nginx!</title>  = a pod on cp1 reached a pod on worker1
kubectl run t2 --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default   # DNS answers
kubectl delete deploy nettest
```

**If it fails:** the join hangs at `[preflight]` → from worker1, run `curl -k https://cp1:6443/livez`. If that fails, it's a network or `/etc/hosts` problem (steps 2–3). The token expired (24 h) → on cp1, run `kubeadm token create --print-join-command` and join again. Pods can't reach each other across nodes → send me `kubectl get pods -n calico-system -o wide` and `kubectl get installation default -o yaml | grep -A6 ipPools`.

---

## Step 11: Speed setup (5 min)

**Goal:** set up the same shortcuts you'll type in the first 2 minutes of the exam, and use them every day from now on.

**Do** **[cp1]**

```bash
cat >> ~/.bashrc <<'EOF'
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period 0"
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
EOF
echo 'set expandtab tabstop=2 shiftwidth=2' >> ~/.vimrc
source ~/.bashrc
```

**Check** **[cp1]**

```bash
k get no                                     # works
k run x --image=nginx:1.27 $do | head -5     # prints YAML
k get po -n kube<TAB>                        # completes to kube-system
```

---

## Step 12: Back up the clean cluster (5 min)

**Goal:** save a copy of both VMs, so on Days 2, 3 and 11 you can undo a broken cluster in about 2 minutes instead of rebuilding for an hour. Lima's own `limactl snapshot` is experimental, so this copies the stopped VM folders instead. On macOS, `cp -c` makes a copy-on-write clone, so it's instant and takes almost no extra disk space.

**Do** **[Mac]**

```bash
limactl stop cp1 && limactl stop worker1
mkdir -p ~/lima-backups/day1
cp -c -R ~/.lima/cp1 ~/.lima/worker1 ~/lima-backups/day1/
limactl start cp1 && limactl start worker1
```

**Check** **[Mac]**

```bash
ls ~/lima-backups/day1                       # cp1 worker1
limactl shell --workdir / cp1 kubectl get nodes   # both Ready again after ~1 min
```

**To restore later** (both VMs together, always):

```bash
limactl stop -f cp1 ; limactl stop -f worker1
rm -rf ~/.lima/cp1 ~/.lima/worker1
cp -c -R ~/lima-backups/day1/cp1 ~/lima-backups/day1/worker1 ~/.lima/
limactl start cp1 && limactl start worker1
```

Take a new backup the same way at the end of any day you want to keep (`~/lima-backups/day3`, etc.).

---

## Day 1 done when

These are the [Day 1 story's acceptance criteria](../stories/day-01-lab-setup.md#acceptance-criteria):

- [ ] `k get nodes` shows **2 nodes Ready**.
- [ ] Without notes, generate YAML with `$do` for a pod, deployment, service, configmap, secret, job and cronjob in **under 5 minutes total**:
  ```bash
  k run web --image=nginx:1.27 $do
  k create deploy web --image=nginx:1.27 --replicas=3 $do
  k create svc clusterip web --tcp=80:80 $do        # or: k expose deploy web --port=80 $do (needs the deploy to exist)
  k create cm cfg --from-literal=MODE=prod $do
  k create secret generic db --from-literal=PASS=s3cr3t $do
  k create job once --image=busybox:1.36 $do -- echo hi
  k create cronjob tick --image=busybox:1.36 --schedule="*/5 * * * *" $do -- date
  ```
- [ ] On kubernetes.io, find the **kubeadm install page** and the **etcd backup page** in under 30 seconds each, using the docs search.

## How this differs from the lessons

- Machine commands use `limactl` instead of `multipass` (`limactl shell --workdir / cp1` = `multipass shell cp1`).
- Your Linux user is your Mac username, not `ubuntu`. Commands that use `~` or `$USER` work the same.
- The pod network is `10.244.0.0/16`. Where a lesson shows `podSubnet: 192.168.0.0/16`, yours says `10.244.0.0/16`.
- Snapshots are the folder copies from step 12.

Tomorrow (Day 2) you take this cluster apart: static pods, certificates, and an etcd backup and restore. Make sure step 12's backup exists before you start.
