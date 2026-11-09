# Lab runbook, Days 1–11, on an Apple Silicon Mac

Every hands-on step from the daily lessons in one place, in order, from an empty Mac to the end of Day 11. Each day lists **where** to run each block, what you should see, the timed drill that matches the story's acceptance criteria, and how to clean up.

The lessons in [`notes/`](../notes/) explain *why*. This file is the *what to type*. When a step says "Day N lesson §X", the full explanation and the long YAML are there.

## Before you start

**What you need:** an Apple Silicon Mac (M1–M4) with 16 GB RAM and ~60 GB free disk, macOS admin rights, Homebrew, and internet for Day 1 (and for the downloads in [offline-prep](../cheatsheets/offline-prep.md)).

**Where to run things.** Every block is labelled:

| Label | Meaning |
|---|---|
| **[Mac]** | The macOS Terminal |
| **[cp1]** | Inside the control-plane VM: `multipass shell cp1` |
| **[worker1]** | Inside the worker VM: `multipass shell worker1`, or `ssh worker1` from cp1 |
| **[both]** | Run the same block on cp1 **and** on worker1 |

Run `kubectl` / `k` on **cp1** unless a step says otherwise.

**Two Mac-specific choices this runbook makes**, which differ from the Day 1 lesson:

1. **Pod CIDR `10.244.0.0/16`, not `192.168.0.0/16`.** Multipass VMs on macOS get addresses in `192.168.64.0/24`. Calico's default pool `192.168.0.0/16` would overlap the node network and break routing. Wherever a later lesson shows `192.168.0.0/16` as the pod subnet, yours is `10.244.0.0/16`.
2. **Multi-arch images only.** The VMs are arm64. Two images used in the Day 6 lesson (`registry.k8s.io/hpa-example`, `polinux/stress`) may not have arm64 builds, so Day 6 below uses busybox and python instead.

**Already have a working cluster from Day 1?** Skip Day 1 and start at Day 2.

---

## Day 1: Build the cluster (≈ 1.5 h)

### 1.1 Install Multipass and create the VMs

**[Mac]**

```bash
brew install --cask multipass
multipass version                                   # 1.13 or newer (snapshots need it)

multipass launch 24.04 --name cp1     --cpus 2 --memory 4G --disk 30G
multipass launch 24.04 --name worker1 --cpus 2 --memory 4G --disk 30G
multipass list                                      # both Running, note the two IPv4 addresses (192.168.64.x)
```

Write the two IPs down. The rest of the runbook calls them `CP_IP` and `W1_IP`.

### 1.2 Names and SSH between the nodes

The lessons use `ssh worker1` from cp1 (Days 2, 3, 9, 10, 11).

**[both]** add both names to `/etc/hosts` (replace the IPs with yours):

```bash
echo "192.168.64.2 cp1
192.168.64.3 worker1" | sudo tee -a /etc/hosts
ping -c1 cp1 && ping -c1 worker1
```

If the entries are missing after a VM restart, add them again. Also check that `multipass list` still shows the same IPs after your Mac reboots: the cluster is tied to them.

**[cp1]** create a key:

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys          # so "ssh cp1" works too
```

**[Mac]** copy cp1's public key to worker1:

```bash
multipass exec cp1 -- cat /home/ubuntu/.ssh/id_ed25519.pub \
  | multipass exec worker1 -- sh -c 'cat >> /home/ubuntu/.ssh/authorized_keys'
```

**[cp1]** test: `ssh -o StrictHostKeyChecking=accept-new worker1 hostname` → `worker1`.

### 1.3 Node prerequisites

**[both]**

```bash
# swap off (Multipass images usually have none; harmless if so)
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab

# kernel modules
printf 'overlay\nbr_netfilter\n' | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

# sysctls
printf 'net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1\n' \
  | sudo tee /etc/sysctl.d/k8s.conf
sudo sysctl --system | grep -E 'ip_forward|bridge-nf-call-iptables'   # both = 1
```

### 1.4 containerd

**[both]**

```bash
sudo apt-get update
sudo apt-get install -y containerd apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml          # SystemdCgroup = true
sudo systemctl restart containerd && sudo systemctl enable containerd
stat -fc %T /sys/fs/cgroup                              # cgroup2fs
```

### 1.5 kubeadm, kubelet, kubectl at v1.34

v1.34 is two minors below the exam's v1.37, so Day 3 has a real upgrade to do.

**[both]**

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

The kubelet now restarts every few seconds. That's expected until `kubeadm init`/`join` gives it a config.

### 1.6 Initialise the control plane

**[cp1]** (use your cp1 IP)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.64.2
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes                                       # cp1 NotReady (no CNI yet): expected
```

Copy the `kubeadm join ...` line it prints. Lost it? `kubeadm token create --print-join-command`.

### 1.7 Calico (v3.33.0)

**[cp1]** These are the steps from Calico's [on-premises install page](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises) for Kubernetes 1.34/1.35, with the pool changed to 10.244.0.0/16:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org-v1beta1.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml
sed -i 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml
grep -n cidr custom-resources.yaml                     # must show 10.244.0.0/16. If not, edit spec.calicoNetwork.ipPools[0].cidr by hand
kubectl create -f custom-resources.yaml
watch kubectl get tigerastatus                          # wait until AVAILABLE True (3–5 min), Ctrl-C
kubectl get nodes                                       # cp1 Ready
```

### 1.8 Join the worker

**[worker1]** paste the join line from 1.6 with `sudo`:

```bash
sudo kubeadm join 192.168.64.2:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**[cp1]**

```bash
kubectl get nodes -o wide                               # cp1 and worker1 Ready, v1.34.x
kubectl get pods -A -o wide                             # calico-*, coredns, kube-proxy all Running
```

### 1.9 Speed setup

**[cp1]**

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
k get nodes
```

`tmux` is preinstalled. Use it for a second pane when a step says "watch in a second terminal" (`Ctrl-b %` splits, `Ctrl-b o` switches).

### 1.10 Snapshot

**[Mac]**

```bash
multipass stop cp1 worker1
multipass snapshot cp1 --name day1 && multipass snapshot worker1 --name day1
multipass start cp1 worker1
# restore later:  multipass stop cp1 worker1 && multipass restore cp1.day1 --destructive && multipass restore worker1.day1 --destructive
```

### Day 1 done when

- `k get nodes` shows 2 nodes Ready.
- ⏱ < 5 min total, no notes: `k run`, `k create deploy`, `k expose`, `k create cm`, `k create secret generic`, `k create job`, `k create cronjob`, each with `$do`.
- You can find the kubeadm install page and the etcd backup page on kubernetes.io in < 30 s each.

---

## Day 2: kubeadm internals and etcd backup/restore (≈ 2 h)

**2.1 etcdctl + etcdutl (arm64).** **[cp1]**

```bash
sudo crictl images | grep etcd                           # e.g. registry.k8s.io/etcd 3.6.x-0
ETCD_VER=v3.6.5                                          # <- set to the version shown above
ARCH=$(dpkg --print-architecture)                        # arm64
curl -fsSL https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz | tar xz -C /tmp
sudo mv /tmp/etcd-${ETCD_VER}-linux-${ARCH}/etcd{ctl,utl} /usr/local/bin/
etcdctl version && etcdutl version
```

**2.2 Read the manifests, PKI and kubelet config.** **[cp1]**

```bash
sudo ls -l /etc/kubernetes/manifests/
sudo grep -E -- '--(client-ca|tls-|etcd-|kubelet-client|service-account)' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo ls /etc/kubernetes/pki /etc/kubernetes/pki/etcd /etc/kubernetes/*.conf
sudo grep -E 'staticPodPath|cgroupDriver|clientCAFile|rotateCertificates' /var/lib/kubelet/config.yaml
sudo kubeadm certs check-expiration                       # note RESIDUAL TIME for leaf certs vs CAs
```

Write one line per apiserver flag in the story Notes (Day 2 lesson §2 has the table).

**2.3 A static pod on worker1.** **[cp1]** `k run static-web --image=nginx:1.27 --port 80 $do` and copy the output. **[worker1]**:

```bash
sudo vim /etc/kubernetes/manifests/static-web.yaml        # paste
```

**[cp1]**

```bash
k get pod -o wide | grep static-web                       # static-web-worker1 Running
k delete pod static-web-worker1 ; k get pod -w            # comes straight back (Ctrl-C)
ssh worker1 sudo rm /etc/kubernetes/manifests/static-web.yaml
k get pod                                                 # gone within ~20 s
```

**2.4 Backup (⏱ < 4 min).** **[cp1]**

```bash
k create ns before-backup
sudo grep -E 'trusted-ca-file|cert-file|key-file|listen-client-urls' /etc/kubernetes/manifests/etcd.yaml
sudo mkdir -p /opt
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
sudo etcdutl snapshot status /opt/etcd-backup.db -w table
```

**2.5 Restore (⏱ < 8 min).** **[cp1]**

```bash
k create ns after-backup
sudo etcdutl snapshot restore /opt/etcd-backup.db --data-dir /var/lib/etcd-restore
sudo cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
sudo vim /etc/kubernetes/manifests/etcd.yaml              # volumes: the etcd-data hostPath → path: /var/lib/etcd-restore
watch sudo crictl ps                                      # etcd, then kube-apiserver come back (1–2 min)
k get ns                                                  # before-backup present, after-backup gone
```

If the API doesn't return in ~2 min: `sudo systemctl restart kubelet`, then `sudo crictl ps -a | grep -E 'etcd|apiserver'` and `sudo crictl logs <id>`.

**2.6 Break the API server once.** **[cp1]**

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/apiserver.bak
sudo sed -i 's#apiserver-etcd-client.crt#apiserver-etcd-clinet.crt#' /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 40; k get nodes                                     # connection refused
sudo crictl ps -a | grep apiserver                        # Exited
sudo crictl logs $(sudo crictl ps -a --name kube-apiserver -q | head -1)   # "no such file or directory"
sudo cp /tmp/apiserver.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Done when** the Day 2 story's criteria pass. Do 2.4 + 2.5 a second time without looking. Snapshot again (`--name day2`).

---

## Day 3: Upgrade v1.34 → v1.35 (≈ 2 h)

**3.1 Snapshot first.** **[Mac]** `multipass stop cp1 worker1 && multipass snapshot cp1 --name v134 && multipass snapshot worker1 --name v134 && multipass start cp1 worker1`

**3.2 etcd drill from memory** (⏱ < 10 min): repeat Day 2.4 + 2.5 with a namespace `pre-upgrade` created before the snapshot and `throwaway` after.

**3.3 Give drain something to do.** **[cp1]**

```bash
k create deploy web --image=nginx:1.27 --replicas=2
k run bare --image=nginx:1.27
k create deploy scratch --image=nginx:1.27 $do > scratch.yaml
vim scratch.yaml        # add under template.spec:  volumes: [{name: tmp, emptyDir: {}}]  and in the container: volumeMounts: [{name: tmp, mountPath: /tmp}]
k apply -f scratch.yaml
k create pdb web-pdb --selector=app=web --min-available=2
```

**3.4 Upgrade cp1 (⏱ < 10 min).** **[cp1]**

```bash
sudo -i
sed -i 's#/v1.34/#/v1.35/#' /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-cache madison kubeadm | head -3       # pick the newest 1.35.x
V=1.35.3                                                    # <- set it
apt-get install -y --allow-change-held-packages kubeadm="$V-*"
kubeadm version -o short
kubeadm upgrade plan
kubeadm upgrade apply v$V                                   # y; ends with [upgrade] SUCCESS!
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes                                           # cp1 still shows v1.34: kubelet not upgraded yet
kubectl drain cp1 --ignore-daemonsets
apt-get install -y --allow-change-held-packages kubelet="$V-*" kubectl="$V-*"
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon cp1 && kubectl get nodes                   # cp1 v1.35.x Ready
exit
```

**3.5 Drain worker1, hitting each flag.** **[cp1]**

```bash
k drain worker1                                             # lists blockers
k drain worker1 --ignore-daemonsets                         # still refuses: bare, scratch
k drain worker1 --ignore-daemonsets --force --delete-emptydir-data --timeout=60s   # blocked by the PDB
k delete pdb web-pdb
k drain worker1 --ignore-daemonsets --force --delete-emptydir-data
```

**3.6 Upgrade worker1 (⏱ < 6 min).** **[cp1]** `ssh worker1`, then:

```bash
sudo -i
sed -i 's#/v1.34/#/v1.35/#' /etc/apt/sources.list.d/kubernetes.list
apt-get update
V=1.35.3                                                    # same as cp1
apt-get install -y --allow-change-held-packages kubeadm="$V-*"
kubeadm upgrade node
apt-get install -y --allow-change-held-packages kubelet="$V-*" kubectl="$V-*"
systemctl daemon-reload && systemctl restart kubelet
exit; exit
```

**[cp1]** `k uncordon worker1 && k get nodes` → both v1.35.x Ready.

**3.7 Repeat:** restore `v134` on both VMs and redo 3.4–3.6 without looking. Then take a `v135` snapshot. Every later day runs on v1.35.

---

## Day 4: RBAC and kubeconfig (≈ 2 h)

**[cp1]** throughout. Run `k config current-context` before every drill.

**4.1 Roles and bindings.**

```bash
k create ns dev
k create role pod-reader -n dev --verb=get,list,watch --resource=pods,pods/log
k create rolebinding pod-reader-jane -n dev --role=pod-reader --user=jane
k auth can-i list pods -n dev --as=jane                   # yes
k auth can-i list pods -n default --as=jane               # no

k create sa ci -n dev
k create role deploy-mgr -n dev --verb=create,get,list,update,patch,delete --resource=deployments
k create rolebinding deploy-mgr-ci -n dev --role=deploy-mgr --serviceaccount=dev:ci
k auth can-i create deployments.apps -n dev --as=system:serviceaccount:dev:ci   # yes
k auth can-i create deployments -n dev --as=ci                                  # no: wrong identity string

k create clusterrole infra-reader --verb=get,list,watch --resource=nodes,persistentvolumes,pods
k create clusterrolebinding infra-reader-jane --clusterrole=infra-reader --user=jane
k create rolebinding infra-reader-bob -n dev --clusterrole=infra-reader --user=bob
k auth can-i list nodes --as=jane ; k auth can-i list pods -n dev --as=bob ; k auth can-i list nodes --as=bob   # yes yes no
```

**4.2 ServiceAccount inside a pod.**

```bash
k create deploy web -n dev --image=curlimages/curl -- sleep infinity
k set sa deploy/web ci -n dev
k -n dev exec -it deploy/web -- sh
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" https://kubernetes.default.svc/apis/apps/v1/namespaces/dev/deployments | head
  curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" https://kubernetes.default.svc/api/v1/namespaces/dev/secrets     # 403
  exit
k auth can-i --list -n dev --as=system:serviceaccount:dev:ci
```

**4.3 A real user with the CSR API.**

```bash
cd ~ && openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=developers"
cat <<EOF | k apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(base64 -w0 < jane.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages: [client auth]
EOF
k certificate approve jane
k get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
k config set-credentials jane --client-certificate=jane.crt --client-key=jane.key --embed-certs=true
k config set-context jane --cluster=kubernetes --user=jane --namespace=dev
k config use-context jane
k auth whoami ; k get pods ; k get deploy                  # works, works, Forbidden
k config use-context kubernetes-admin@kubernetes && k config current-context
k create rolebinding dev-view -n dev --clusterrole=view --group=developers
k --context=jane get deploy                                # now works through the group
```

**4.4 Five-task drill (⏱ < 3 min each, verify with `can-i`).** Clean up first: `k delete ns dev; k delete clusterrole infra-reader; k delete clusterrolebinding infra-reader-jane`.

1. SA `monitor` in ns `ops` may get/list/watch pods and services in `ops` only.
2. User `alice` may create secrets in ns `app`, and delete only the secret `app-tls` (two rules → YAML with `resourceNames`).
3. Group `auditors` may use the built-in `view` in all namespaces.
4. SA `scaler` in `app` may patch/update `deployments/scale` in `app` and nothing else.
5. User `bob` may list nodes and get the non-resource URL `/healthz`.

---

## Day 5: Helm, Kustomize, CRDs (≈ 2 h)

**5.1 Helm on cp1.** **[cp1]**

```bash
HV=$(curl -s https://api.github.com/repos/helm/helm/releases/latest | grep -oP '"tag_name": "\K[^"]+')
ARCH=$(dpkg --print-architecture)
curl -sL https://get.helm.sh/helm-${HV}-linux-${ARCH}.tar.gz | tar xz
sudo mv linux-${ARCH}/helm /usr/local/bin/ && helm version
echo 'source <(helm completion bash)' >> ~/.bashrc
```

**5.2 Helm lifecycle (⏱ < 5 min).**

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo && helm repo update
helm install web podinfo/podinfo -n apps --create-namespace --set replicaCount=2 --set ui.message="rev1"
helm upgrade web podinfo/podinfo -n apps --reuse-values --set ui.message="rev2"
helm history web -n apps
helm rollback web 1 -n apps && helm get values web -n apps        # rev1 again
helm show values podinfo/podinfo > /tmp/pi-values.yaml            # edit replicaCount: 3, service.type: NodePort
helm template web podinfo/podinfo -f /tmp/pi-values.yaml | grep -E 'kind:|replicas:|type:'
```

**5.3 Kustomize overlay (⏱ < 5 min).** Type the five files from Day 5 lesson §3 into `~/app/base/{deployment,service,kustomization}.yaml` and `~/app/overlays/prod/kustomization.yaml` (namespace prod, namePrefix prod-, label env=prod with includeSelectors, image nginx → 1.28, replicas 3). Then:

```bash
k create ns prod
k kustomize ~/app/overlays/prod | less
k apply -k ~/app/overlays/prod
k -n prod get deploy prod-web -o wide                              # 3/3, nginx:1.28
```

**5.4 CRDs and an operator.**

```bash
k get crd | head ; k api-resources | grep -i calico
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager -n cert-manager --create-namespace --set crds.enabled=true
k -n cert-manager rollout status deploy/cert-manager-webhook
```

⏱ < 4 min, using only `k explain`: create this and check `k -n prod get certificate web-tls` → READY True.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: selfsigned}
spec: {selfSigned: {}}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: web-tls, namespace: prod}
spec:
  secretName: web-tls
  dnsNames: [web.prod.svc]
  issuerRef: {name: selfsigned, kind: ClusterIssuer}
```

**Cleanup:** `helm uninstall web -n apps; k delete -k ~/app/overlays/prod`. Keep cert-manager.

---

## Day 6: Deployments, config, resources (≈ 2 h)

**[cp1]** `k create ns d6 && k config set-context --current --namespace=d6`

**6.1 Rollout and rollback (⏱ < 3 min from the bad image).**

```bash
k create deploy web --image=nginx:1.27 --replicas=4
k set image deploy/web nginx=nginx:1.28 && k rollout status deploy/web
k set image deploy/web nginx=nginx:1.99-doesnotexist
k rollout status deploy/web --timeout=20s                          # times out
k rollout history deploy/web
k rollout undo deploy/web --to-revision=1
k get deploy web -o jsonpath='{..image}{"\n"}'                     # nginx:1.27
```

**6.2 ConfigMap + Secret in one pod (⏱ < 5 min).**

```bash
k create cm app-cfg --from-literal=MODE=prod --from-literal=LOG_LEVEL=debug
k create secret generic db --from-literal=username=app --from-literal=password=S3cr3t
```

```yaml
apiVersion: v1
kind: Pod
metadata: {name: cfg}
spec:
  volumes:
  - name: db
    secret: {secretName: db}
  containers:
  - name: cfg
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: APP_MODE
      valueFrom: {configMapKeyRef: {name: app-cfg, key: MODE}}
    volumeMounts:
    - {name: db, mountPath: /etc/db, readOnly: true}
```

```bash
k exec cfg -- sh -c 'echo $APP_MODE; ls /etc/db; cat /etc/db/password; echo'   # prod / password username / S3cr3t
```

**6.3 QoS and OOMKill.**

```bash
k run be --image=nginx:1.27
k run bu --image=nginx:1.27 $do > bu.yaml        # add resources.requests: {cpu: 100m, memory: 64Mi}
k run gu --image=nginx:1.27 $do > gu.yaml        # add resources.limits:   {cpu: 200m, memory: 128Mi}
k apply -f bu.yaml -f gu.yaml
k get pod be bu gu -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass   # BestEffort Burstable Guaranteed

k run oom --image=python:3.12-alpine --restart=Never $do -- python -c "a=' '*300_000_000; import time; time.sleep(60)" > oom.yaml
vim oom.yaml                                     # add resources.limits: {memory: 100Mi}
k apply -f oom.yaml ; sleep 15
k describe pod oom | grep -A3 'Last State\|State:'   # OOMKilled, exit code 137
```

**6.4 Quota and LimitRange.**

```bash
k create ns team-a
k create quota team-a-q -n team-a --hard=pods=3,requests.cpu=500m,requests.memory=512Mi
k run nores -n team-a --image=nginx:1.27                           # rejected: must specify requests
```

```yaml
apiVersion: v1
kind: LimitRange
metadata: {name: defaults, namespace: team-a}
spec:
  limits:
  - type: Container
    defaultRequest: {cpu: 100m, memory: 64Mi}
    default:        {cpu: 200m, memory: 128Mi}
```

```bash
k apply -f limitrange.yaml && k run nores -n team-a --image=nginx:1.27   # admitted, defaults injected
k create deploy big -n team-a --image=nginx:1.27 --replicas=5
k get deploy big -n team-a ; k describe rs -n team-a -l app=big | grep -A2 FailedCreate   # exceeded quota
```

**6.5 metrics-server and HPA.**

```bash
k apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
k -n kube-system patch deploy metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
k -n kube-system rollout status deploy/metrics-server ; sleep 60 ; k top nodes

k create deploy burn --image=busybox:1.36 -- sh -c 'while true; do :; done'   # busy loop = CPU load (multi-arch)
k set resources deploy burn --requests=cpu=100m --limits=cpu=200m
k autoscale deploy burn --min=1 --max=4 --cpu=50%
k get hpa burn -w                                                  # ~200%/50% → scales to 4 (Ctrl-C)
k delete deploy burn ; k delete hpa burn
```

**6.6 Jobs and CronJobs.**

```yaml
apiVersion: batch/v1
kind: Job
metadata: {name: batch}
spec:
  completions: 6
  parallelism: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - {name: w, image: busybox:1.36, command: ["sh", "-c", "echo $HOSTNAME; sleep 5"]}
```

```bash
k apply -f job.yaml ; k get pods -l job-name=batch -w             # 2 at a time, 6 total
k create cronjob tick --image=busybox:1.36 --schedule="*/1 * * * *" -- sh -c date
k create job tick-now --from=cronjob/tick && k logs job/tick-now
```

**Cleanup:** `k delete ns d6 team-a && k config set-context --current --namespace=default`. Keep metrics-server.

---

## Day 7: Scheduling (≈ 2 h + 1 h mini mock)

**[cp1]** `k create ns d7 && k config set-context --current --namespace=d7`

**7.1 Taints.**

```bash
k taint nodes worker1 dedicated=gpu:NoSchedule
k create deploy t --image=nginx:1.27 --replicas=2 ; k describe pod -l app=t | grep -A3 Events   # untolerated taint
k taint nodes worker1 dedicated=gpu:NoSchedule-                    # pods start
k taint nodes worker1 evict=now:NoExecute ; sleep 10 ; k get pods -o wide   # evicted, Pending
k taint nodes worker1 evict-                                       # remove quickly (CoreDNS was evicted too)
```

**7.2 nodeSelector, affinity, nodeName.**

```bash
k label node worker1 disktype=ssd
k run sel --image=nginx:1.27 $do > sel.yaml        # add spec.nodeSelector: {disktype: ssd} → apply → on worker1
k run byname --image=nginx:1.27 $do > byname.yaml  # add spec.nodeName: cp1 → Running on cp1 despite the taint
k apply -f sel.yaml -f byname.yaml && k get pods -o wide
```

**7.3 Spread 3 replicas with anti-affinity (⏱ < 5 min).** `k create deploy spread --image=nginx:1.27 --replicas=3 $do > spread.yaml`, then add under `spec.template.spec`:

```yaml
      tolerations:
      - {key: node-role.kubernetes.io/control-plane, operator: Exists, effect: NoSchedule}
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels: {app: spread}
            topologyKey: kubernetes.io/hostname
```

`k apply -f spread.yaml && k get pods -l app=spread -o wide` → one on cp1, one on worker1, one **Pending** (`didn't match pod anti-affinity rules`). On 2 nodes, that Pending pod is the proof.

**7.4 PriorityClass and preemption.**

```bash
k create pc low --value=10 ; k create pc high --value=100000
k create deploy filler --image=nginx:1.27 --replicas=4 $do > filler.yaml   # add priorityClassName: low, requests.cpu: 400m
k apply -f filler.yaml
k run vip --image=nginx:1.27 $do > vip.yaml                               # add priorityClassName: high, requests.cpu: 800m
k apply -f vip.yaml ; k get pods -o wide                                  # some filler pods preempted, vip Running
```

**7.5 Init container + native sidecar.** Apply the `app` pod from Day 7 lesson §6, see `Init:0/2`, then `k create svc clusterip db --tcp=80` and watch it reach `2/2`. `k logs app -c log-shipper`.

**7.6 Mini mock (60 min, no notes):** the 8 tasks in Day 7 lesson §8. Pass = 6/8. Log misses in Day 8 Notes.

**Cleanup:** `k delete ns d7; k label node worker1 disktype-; k delete pc low high`; check `k describe node worker1 | grep Taints` → `<none>`.

---

## Day 8: Services and DNS (≈ 2 h)

**[cp1]** `k create ns d8 && k config set-context --current --namespace=d8`

**8.1 Service types.**

```bash
k create deploy web --image=nginx:1.27 --replicas=2 --port=80
k expose deploy web --port=80
k expose deploy web --port=8080 --target-port=80 --name=web-8080
k create service clusterip web-hl --clusterip=None --tcp=80:80
k set selector svc web-hl app=web && k get ep web-hl                # 2 pod IPs
```

**8.2 NodePort 30080 (⏱ < 3 min).**

```bash
k create deploy shop --image=nginx:1.27 --replicas=2
k expose deploy shop --port=80 --type=NodePort $do > shop-svc.yaml   # add nodePort: 30080 under ports
k apply -f shop-svc.yaml
curl -s localhost:30080 | grep -i title
curl -s worker1:30080 | grep -i title
```

**[Mac]** `curl -s http://<W1_IP>:30080 | grep -i title` works from the Mac too.

**8.3 Break and fix a Service (⏱ < 4 min).**

```bash
k create deploy api --image=nginx:1.27 --replicas=2 && k expose deploy api --port=80
k patch svc api -p '{"spec":{"selector":{"app":"apii"}}}'
k patch svc api --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":8080}]'
# now fix it: wget from a test pod → k get ep api → compare labels → k set selector svc api app=api → fix targetPort to 80
k run t --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- -T 2 api | grep title
```

**8.4 DNS names.**

```bash
k create ns other && k -n other create deploy backend --image=nginx:1.27 && k -n other expose deploy backend --port=80
k run t --image=busybox:1.28 --rm -it --restart=Never -- sh
  cat /etc/resolv.conf
  nslookup backend.other.svc.cluster.local
  nslookup backend                     # fails: wrong namespace
  nslookup web-hl                      # 2 pod IPs
  exit
```

**8.5 CoreDNS.**

```bash
k -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'
k -n kube-system get cm kubeadm-config -o yaml | grep -i subnet      # podSubnet 10.244.0.0/16 on this lab
k -n kube-system scale deploy coredns --replicas=0
k run t --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default   # times out
k -n kube-system get ep kube-dns                                     # <none>
k -n kube-system scale deploy coredns --replicas=2
```

**Cleanup:** `k delete ns d8 other && k config set-context --current --namespace=default`

---

## Day 9: NetworkPolicy, Ingress, Gateway API (≈ 2.5 h)

**[cp1]**

```bash
k create ns d9 && k config set-context --current --namespace=d9
k create ns frontend ; k create ns backend
k -n backend create deploy db --image=python:3.12-alpine --replicas=2 -- python -m http.server 5432
k -n backend expose deploy db --port=5432
k -n frontend run api --image=busybox:1.36 -l role=api -- sleep 3600
k -n frontend run web --image=busybox:1.36 -l role=web -- sleep 3600
k -n d9       run spy --image=busybox:1.36 -l role=api -- sleep 3600
```

**9.1 The db policy (⏱ < 6 min).** `db-np.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-from-api, namespace: backend}
spec:
  podSelector: {matchLabels: {app: db}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: frontend}}
      podSelector: {matchLabels: {role: api}}
    ports:
    - {protocol: TCP, port: 5432}
```

```bash
k apply -f db-np.yaml
k -n frontend exec api -- wget -qO- -T 2 db.backend:5432 | head -1    # allowed
k -n frontend exec web -- wget -qO- -T 2 db.backend:5432              # timed out
k -n d9       exec spy -- wget -qO- -T 2 db.backend:5432              # timed out
```

Then put a `-` before `podSelector:` (OR), re-apply, and see `web` get through. Put it back.

**9.2 Default deny + DNS.**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: frontend}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-dns, namespace: frontend}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}
      podSelector: {matchLabels: {k8s-app: kube-dns}}
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
```

Apply only the first: `wget` from api says `bad address`. Apply the second: name resolves, connection times out (egress to db not allowed yet). Add an egress rule from `role=api` to `app=db` in backend on 5432 (Day 9 lesson §7 step 2) and api works again. `k -n frontend delete netpol --all` afterwards.

**9.3 Ingress.**

```bash
k apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
k -n ingress-nginx wait --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=180s
NP=$(k -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
k create deploy web-v1 --image=nginx:1.27 ; k create deploy web-v2 --image=httpd:2.4
k expose deploy web-v1 --port=80 ; k expose deploy web-v2 --port=80
k create ingress hosts --class=nginx --rule="v1.example.com/*=web-v1:80" --rule="v2.example.com/*=web-v2:80"
curl -s -H 'Host: v1.example.com' worker1:$NP/ | grep -o '<title>.*</title>'   # nginx
curl -s -H 'Host: v2.example.com' worker1:$NP/                                  # It works!
```

**9.4 Gateway API, 80/20 split (⏱ < 6 min).**

```bash
k kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.8.1" | k apply -f -
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 2.8.1 --create-namespace -n nginx-gateway --set nginx.service.type=NodePort
k get gatewayclass                                                  # nginx ACCEPTED True
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: web-gw, namespace: d9}
spec:
  gatewayClassName: nginx
  listeners:
  - {name: http, protocol: HTTP, port: 80}
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: split, namespace: d9}
spec:
  parentRefs: [{name: web-gw}]
  hostnames: ["app.example.com"]
  rules:
  - backendRefs:
    - {name: web-v1, port: 80, weight: 80}
    - {name: web-v2, port: 80, weight: 20}
```

```bash
k apply -f gw.yaml ; k get gateway web-gw                           # PROGRAMMED True
k get svc                                                           # the gateway's NodePort Service: note its 80:3xxxx
GNP=<that nodePort>
for i in $(seq 1 50); do curl -s -H 'Host: app.example.com' worker1:$GNP/ | grep -o -E 'Welcome to nginx|It works'; done | sort | uniq -c   # ~40 / ~10
```

**9.5 CNI files.** **[both]** `ls -l /etc/cni/net.d/ /opt/cni/bin/`. **[cp1]** `k get installation default -o jsonpath='{.spec.calicoNetwork.ipPools}{"\n"}'` → cidr 10.244.0.0/16.

**Cleanup:** `k delete ns d9 frontend backend && k config set-context --current --namespace=default`. Keep ingress-nginx and NGF.

---

## Day 10: Storage (≈ 1.5 h)

**[cp1]** `k create ns d10 && k config set-context --current --namespace=d10`

**10.1 PV + PVC + pod (⏱ < 6 min).** Type from the docs page *Configure a Pod to Use a PersistentVolume*:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: {name: pv-data}
spec:
  storageClassName: manual
  capacity: {storage: 1Gi}
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  hostPath: {path: /mnt/data, type: DirectoryOrCreate}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: pvc-data, namespace: d10}
spec:
  storageClassName: manual
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
---
apiVersion: v1
kind: Pod
metadata: {name: writer, namespace: d10}
spec:
  volumes:
  - name: data
    persistentVolumeClaim: {claimName: pvc-data}
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "date >> /data/log.txt; sleep 3600"]
    volumeMounts:
    - {name: data, mountPath: /data}
```

```bash
k apply -f storage.yaml ; k get pv,pvc ; k exec writer -- cat /data/log.txt
k delete pod writer --force ; k apply -f storage.yaml ; k exec writer -- cat /data/log.txt   # 2 lines: data survived
```

**10.2 Released PV.**

```bash
k delete pod writer ; k delete pvc pvc-data ; k get pv pv-data            # Released
k apply -f storage.yaml ; k get pvc                                       # Pending
k patch pv pv-data --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]' ; k get pv,pvc   # Bound
```

**10.3 Pending-PVC drills (⏱ < 1 min each).** Recreate the PVC with one wrong field at a time (`2Gi`, `ReadWriteMany`, `storageClassName: manul`) and name the mismatch from `k describe pvc` + `k get pv`.

**10.4 Dynamic provisioning + default class + expansion.**

```bash
k apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
k get sc local-path
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations: {storageclass.kubernetes.io/is-default-class: "true"}
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
k apply -f sc-fast.yaml && k get sc                                       # fast (default)
# PVC pvc-dyn: 500Mi RWO, no storageClassName → STORAGECLASS fast, Pending (waiting for first consumer)
# a pod that mounts pvc-dyn → PVC Bound, a pvc-<uid> PV appears
k patch pvc pvc-dyn -p '{"spec":{"resources":{"requests":{"storage":"1Gi"}}}}'
k get pvc pvc-dyn -o jsonpath='{.spec.resources.requests.storage} {.status.capacity.storage}{"\n"}'   # 1Gi 500Mi (local-path can't resize)
```

**Cleanup:** `k delete ns d10`, delete leftover PVs (`k get pv`), unmark/delete `fast`, and make `local-path` the default.

---

## Day 11: Break and fix the cluster (≈ 2.5 h)

**11.1 Snapshot both VMs first** (**[Mac]**, as in 1.10, `--name day11`).

**11.2 Baseline.** **[cp1]** `k get nodes -o wide; k get pods -n kube-system -o wide; k create deploy web --image=nginx:1.27`

**11.3 The six faults (⏱ < 6 min each).** Either let `chaos.sh` (Day 11 lesson §5, saved as `/root/chaos.sh` on each node) pick one at random, or inject them by hand:

| # | Where | Inject |
|---|---|---|
| 1 | worker1 | `sudo systemctl stop kubelet` |
| 2 | worker1 | `sudo sed -i 's#/etc/kubernetes/pki/ca.crt#/etc/kubernetes/pki/ca.crtx#' /var/lib/kubelet/config.yaml && sudo systemctl restart kubelet` |
| 3 | cp1 | `sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/ && sudo sed -i 's#127.0.0.1:2379#127.0.0.1:2380#' /etc/kubernetes/manifests/kube-apiserver.yaml` |
| 4 | cp1 | `sudo cp /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/ && sudo sed -i 's#\(image: registry.k8s.io/kube-scheduler:\).*#\1v9.9.9#' /etc/kubernetes/manifests/kube-scheduler.yaml` |
| 5 | cp1 | `sudo cp /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/ && sudo sed -i 's#controller-manager.conf#controller-manager.cnf#' /etc/kubernetes/manifests/kube-controller-manager.yaml` |
| 6 | worker1 | `sudo systemctl stop containerd` |

Wait 60 s, start the timer, and follow the fixed orders (Day 11 lesson §3): **kubectl refused** → context/server → `curl -k https://localhost:6443/livez` → `crictl ps -a --name kube-apiserver` → `crictl logs` → `journalctl -u kubelet` → kubelet/containerd status → etcd → certs. **Node NotReady** → `k describe node` conditions → `ssh` → `systemctl status kubelet` → `journalctl -u kubelet -n 50` → `systemctl status containerd`.

Stop the timer only when the probe passes: node `Ready` (faults 1, 2, 6), `k get nodes` answers steadily (3), `k run probe --image=nginx:1.27` Running (4), `k scale deploy web --replicas=3` gives 3 pods (5).

**11.4 Log tools.** **[cp1]** `sudo crictl ps -a | head`, `sudo crictl pods | head`, `sudo journalctl -u kubelet -n 20 --no-pager`, `sudo ls /var/log/pods/`, `k get events -A --sort-by=.lastTimestamp | tail`, `k top nodes`.

**11.5 From memory:** say the "kubectl connection refused" order out loud, then write it in the story Notes.

**Cleanup:** `k delete deploy web; k delete pod probe`, all nodes Ready, all kube-system pods Running. Then snapshot (`--name day11-done`).

---

## After Day 11

- Before going offline: [offline-prep](../cheatsheets/offline-prep.md).
- Days 12 and 14 are killer.sh sessions (online). Day 13 is review: redo every missed or slow step from this runbook without notes.
