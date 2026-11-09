# Day 11 Lesson: Break and fix the cluster (nodes, kubelet, containerd and the control plane)

**Story:** [day-11-troubleshoot-cluster](../stories/day-11-troubleshoot-cluster.md) · **Read time:** ~40 min · **Then:** 2–2.5 hours in the lab

**Why this matters:** Troubleshooting is **30%** of the exam, the heaviest domain. Expect at least one "node X is NotReady, fix it" task and at least one "the cluster is broken / a component isn't working" task, and they're often worth more than average. Both are scored on one thing: is the node Ready and the component Running at the end. Partial credit is rare. These tasks are also where people burn 20 minutes, because they guess instead of following a fixed order. The rule today: **find the evidence before you touch anything, and fix the first broken layer you find.**

On EKS, "node NotReady" meant replacing the instance and "API server down" meant opening an AWS support case. Here there's no replacement node and no support. You SSH in and read logs from systemd, containerd and the kubelet. Days 1 and 2 showed you where every file lives. Today you break each one on purpose and learn what the evidence looks like, so on Friday you recognise it in seconds.

> **Version notes (checked today).**
> - The [official CKA page](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) still says the exam is based on **Kubernetes v1.37**, with Troubleshooting at **30%**. kubernetes.io/docs is at v1.37.
> - Nothing in this lesson changed between your **v1.35** lab (after Day 3's upgrade) and v1.37. File paths, `crictl`, `journalctl` and the kubelet config format are the same.
> - `kubectl debug node/<name>` is in the docs ([Debugging Kubernetes Nodes With Kubectl](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/)). It needs a **working API server** and a kubelet that can start pods, so it's no use for most of today's faults. SSH is the main tool.
> - `k top` needs metrics-server. The exam clusters usually have it. Your lab only has it if you installed it on Day 6 (see Day 6, section 6).

---

## 1. The mental model: four layers, and where each one logs

Every fault today sits in one of four layers. Each layer depends on the one below it, and each one logs somewhere different:

```
 layer 4   kubectl / your kubeconfig        ~/.kube/config           "connection refused", "x509", "Unauthorized"
              │  HTTPS :6443
 layer 3   control-plane static pods        crictl ps -a / crictl logs / /var/log/pods/
           (apiserver, etcd, scheduler, kcm)   ▲ started by the kubelet from /etc/kubernetes/manifests/
              │
 layer 2   kubelet  (systemd service)       systemctl status kubelet / journalctl -u kubelet
              │  CRI socket /run/containerd/containerd.sock
 layer 1   containerd (systemd service)     systemctl status containerd / journalctl -u containerd
```

What that means in practice:

| If this is broken… | …then you see | Logs live in |
|---|---|---|
| **containerd** | kubelet runs but can't start or list containers. Node goes NotReady. Already-running containers **keep running** (each one has its own `containerd-shim` process) | `journalctl -u containerd`, and the kubelet journal complaining about the CRI socket |
| **kubelet** | No new pods on that node, no status updates. Node goes NotReady (`Unknown`) after the grace period. Running containers **keep running** | `journalctl -u kubelet` |
| **a static pod manifest** | That component's container exits, restarts, or never appears | `crictl ps -a` + `crictl logs <id>`, or `/var/log/pods/`. If **no container exists at all**, it's the kubelet journal (YAML parse error, image pull failure) |
| **kubeconfig / client** | `kubectl` fails but the cluster is fine | `k config view`, `k cluster-info`, `curl -k https://<cp>:6443/livez` |

Two consequences people forget:

- **A dead kubelet doesn't kill pods.** If the kubelet on cp1 stops, the API server keeps serving (its container is still running). `k get nodes` works and shows cp1 `NotReady`. Don't go looking at the API server.
- **A dead scheduler or controller-manager doesn't break `kubectl`.** Everything looks fine until you create or scale something. That's why the story has you create a pod and scale a Deployment. You need a probe that exercises the component.

### What each control-plane component does when it's missing

| Down | `kubectl` works? | Symptom |
|---|---|---|
| etcd | No (apiserver can't serve, then exits) | `connection refused` or timeouts; apiserver logs `dial tcp 127.0.0.1:2379: connect: connection refused` |
| kube-apiserver | No | `The connection to the server 192.168.x.x:6443 was refused` |
| kube-scheduler | Yes | New pods stay `Pending` with **no events at all** (no `FailedScheduling`, because nothing is even trying). `NODE` is `<none>` |
| kube-controller-manager | Yes | `k scale` changes `spec.replicas` but no pods appear. Deleted pods aren't replaced. Deployments don't create ReplicaSets. Nodes that die are never marked `NotReady` (the node lifecycle controller lives here too) |
| kubelet (on a node) | Yes | That node `NotReady`, conditions `Unknown`, message `Kubelet stopped posting node status` |
| containerd (on a node) | Yes | That node `NotReady`; kubelet logs `container runtime is down` / `connect: no such file or directory` on the containerd socket |
| CNI (Calico pod on a node) | Yes | Node `NotReady` with `cni plugin not initialized`, or pods stuck `ContainerCreating` with `failed to setup network for sandbox` |

Pending with no events → scheduler. Pending **with** a `FailedScheduling` event → the scheduler is fine, read the event (taints, resources, affinity, PVC). That one distinction saves minutes.

---

## 2. The tools, and what each one is for

### systemd: the kubelet and containerd

```bash
sudo systemctl status kubelet          # Active: active (running) | activating (auto-restart) | inactive (dead)
sudo systemctl is-enabled kubelet      # "disabled" = it won't come back after a reboot; the grader may reboot or check this
sudo journalctl -u kubelet -n 50 --no-pager      # last 50 lines; start here, not with -f
sudo journalctl -u kubelet --since "5 min ago" --no-pager | grep -iE 'error|fail' | tail -20
sudo journalctl -u kubelet -f          # live, once you know what you're looking for
systemctl cat kubelet                  # unit + /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf drop-in
cat /var/lib/kubelet/kubeadm-flags.env # extra flags kubeadm passes (container-runtime-endpoint, node-ip ...)
```

`activating (auto-restart)` means the kubelet starts, fails, and systemd restarts it every 10 seconds. That's a **config** problem. Read the journal. `inactive (dead)` means someone stopped it. `systemctl enable --now kubelet` and check it stays up.

After editing a systemd unit or drop-in, run `systemctl daemon-reload` before restarting. After editing `/var/lib/kubelet/config.yaml` you only need `systemctl restart kubelet`.

### The kubelet's files

| File | What's in it | Typical fault |
|---|---|---|
| `/var/lib/kubelet/config.yaml` | KubeletConfiguration: `staticPodPath`, `authentication.x509.clientCAFile`, `cgroupDriver`, `clusterDNS`, `rotateCertificates` | Wrong path (`clientCAFile`), wrong `cgroupDriver`, typo in a field |
| `/etc/kubernetes/kubelet.conf` | kubeconfig the kubelet uses to talk to the API server | Wrong `server:` port or IP, wrong cert path |
| `/var/lib/kubelet/pki/kubelet-client-current.pem` | kubelet's client cert (rotated automatically) | Expired or deleted |
| `/var/lib/kubelet/kubeadm-flags.env` | `--container-runtime-endpoint`, `--node-ip`, … | Wrong socket path |
| `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` | Points the kubelet at the files above | Wrong `--config` or `--kubeconfig` path; wrong binary path in `ExecStart` |

The story says "wrong `--client-ca-file`". With kubeadm that setting lives in the config file as **`clientCAFile`** under `authentication.x509`, not as a flag. Grep for both.

### crictl: the container runtime's view

`crictl` talks to containerd directly, so it works when the API server is dead. kubeadm doesn't write `/etc/crictl.yaml`. If `crictl` warns about default endpoints, either ignore it or set it once:

```bash
sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock   # writes /etc/crictl.yaml
```

```bash
sudo crictl ps                              # running containers
sudo crictl ps -a                           # include Exited: the one you want
sudo crictl ps -a --name kube-apiserver     # filter by container name
sudo crictl pods                            # pod sandboxes (a sandbox with no container = image or start failure)
sudo crictl logs <container-id>             # stdout/stderr of that container, even if Exited
sudo crictl logs --tail 20 $(sudo crictl ps -a --name kube-apiserver -q | head -1)   # newest apiserver container
sudo crictl inspect <id> | grep -iA3 -E '"reason"|exitCode'
sudo crictl images | grep kube              # what images are present
```

`crictl ps -a` lists the newest container first per name, and the `ATTEMPT` column climbs while it crash-loops. An `ATTEMPT` that keeps rising is a component crashing after start (bad flag value, can't reach etcd). No container at all for that name is a manifest the kubelet couldn't parse, or an image it couldn't pull.

### Log files on disk

```
/var/log/pods/<namespace>_<pod>_<uid>/<container>/<restart>.log   # the real files, kept across restarts
/var/log/containers/<pod>_<namespace>_<container>-<id>.log        # symlinks into the above
```

```bash
sudo ls /var/log/pods/ | grep apiserver
sudo tail -20 /var/log/pods/kube-system_kube-apiserver-cp1_*/kube-apiserver/*.log
```

When crictl shows a container that already got garbage-collected, these files still hold its last output.

### When the API server is up

```bash
k cluster-info                                   # control plane + CoreDNS URLs
k get nodes -o wide
k describe node worker1 | sed -n '/Conditions/,/Addresses/p'     # Ready, MemoryPressure, DiskPressure, PIDPressure
k get pods -n kube-system -o wide                # every static pod shows as <component>-cp1
k get events -A --sort-by=.lastTimestamp | tail -20
k get events -A --field-selector type=Warning
k top nodes ; k top pods -A --sort-by=memory     # needs metrics-server
k logs -n kube-system kube-scheduler-cp1 --tail 20   # works for static pods while the apiserver is up
k get --raw='/readyz?verbose'                     # each apiserver health check: [+]etcd ok ...
```

`curl -k https://localhost:6443/livez` also works from cp1 with no credentials (kubeadm allows anonymous access to the health endpoints). `ok` means the API server process is alive, so a `kubectl` failure is a kubeconfig problem.

---

## 3. Two fixed orders to memorise

### A. "kubectl: connection refused" (acceptance criterion 2)

Say this out loud until it's automatic:

1. **Am I on the right cluster and the right machine?** `k config current-context`, and the exam's `kubectl config use-context` line for this question. Then `k config view --minify | grep server`. Is that IP and port right? (Port `6443`, not `6444`.)
2. **Is anything listening?** On cp1: `sudo ss -tlnp | grep 6443`, or `curl -k https://localhost:6443/livez`.
3. **Is the apiserver container there?** `sudo crictl ps -a --name kube-apiserver`. Exited, climbing ATTEMPT, or missing?
4. **Why did it die?** `sudo crictl logs <id>` (or `/var/log/pods/kube-system_kube-apiserver-*`). The last 5–10 lines name the problem: unknown flag, file not found, etcd unreachable.
5. **No container at all?** `sudo journalctl -u kubelet -n 50 | grep -i apiserver`: YAML parse error or image pull failure.
6. **Is the kubelet itself running?** `systemctl status kubelet`. If not, nothing in `/etc/kubernetes/manifests` gets started. Then `systemctl status containerd`.
7. **Is etcd healthy?** `sudo crictl ps -a --name etcd` + its logs. The apiserver can't run without it.
8. **Certificates?** `sudo kubeadm certs check-expiration` if the logs mention `x509: certificate has expired`.

Steps 1–4 find most faults in under two minutes.

### B. "Node X is NotReady"

1. `k describe node X | grep -A8 Conditions`. `Unknown` + `Kubelet stopped posting node status` → kubelet or its connection. `False` with a `NetworkPluginNotReady`/`cni plugin not initialized` message → CNI.
2. `ssh X`, then `sudo -i`.
3. `systemctl status kubelet`. Stopped → `systemctl enable --now kubelet`. Restart loop → step 4.
4. `journalctl -u kubelet -n 50 --no-pager`. The first `E…` line usually names the file and the field.
5. `systemctl status containerd`. If it's down, fix it **first**, then restart the kubelet.
6. Fix, `systemctl restart kubelet`, `exit` back to the base node, and wait for `k get nodes -w` to show `Ready` (20–60 s).

---

## 4. The six story faults: symptom, evidence, fix

Inject commands are in section 5. This table is what you should recognise on sight.

| # | Fault | What you see first | The evidence | Fix |
|---|---|---|---|---|
| 1 | `systemctl stop kubelet` on worker1 | worker1 `NotReady` after ~40–50 s, conditions `Unknown` | `systemctl status kubelet` → `inactive (dead)` | `systemctl enable --now kubelet` |
| 2 | Wrong `clientCAFile` in worker1's `/var/lib/kubelet/config.yaml` | worker1 `NotReady` | `status` → `activating (auto-restart)`; journal: `failed to construct kubelet dependencies: unable to load client CA file /etc/kubernetes/pki/ca.crtx: open … no such file or directory` | Correct the path to `/etc/kubernetes/pki/ca.crt`, `systemctl restart kubelet` |
| 3 | Wrong `--etcd-servers` port in `kube-apiserver.yaml` | `kubectl` connection refused (after 30–60 s) | `crictl ps -a --name kube-apiserver` → Exited, ATTEMPT climbing; `crictl logs` → `dial tcp 127.0.0.1:2380: connect: connection refused` or a TLS handshake error to 2380 | Port back to `2379` |
| 4 | Bad image tag in `kube-scheduler.yaml` | New pods `Pending`, **no events** | `k get pods -n kube-system` → `kube-scheduler-cp1 ImagePullBackOff`; `crictl ps -a --name kube-scheduler` → nothing new; journal: `Failed to pull image "registry.k8s.io/kube-scheduler:v9.9.9"` | Restore the tag (`k get pods -n kube-system -o yaml \| grep image:` shows the tag the others use) |
| 5 | Broken `kube-controller-manager.yaml` | `k scale deploy web --replicas=5` → still 1 pod | `kube-controller-manager-cp1` `CrashLoopBackOff` (bad flag value) or `RunContainerError` (bad command); `crictl logs` names it | Fix the flag, path or command |
| 6 | `systemctl stop containerd` on worker1 | worker1 `NotReady`; its pods still answer curl for a while | kubelet journal: `"Container runtime not ready"`, `connect: no such file or directory` on `/run/containerd/containerd.sock`; `systemctl status containerd` → inactive | `systemctl enable --now containerd`, then `systemctl restart kubelet` if the node doesn't recover within a minute |

Notes on a few:

- **Fault 3 variants.** A wrong flag **name** (`--etcd-server=`) gives `Error: unknown flag: --etcd-server` and an immediate exit. A wrong cert path gives `open /etc/kubernetes/pki/…: no such file or directory` (you did that on Day 2). A YAML indentation error gives **no container at all**, only a kubelet journal line like `Could not process manifest file`. Each needs a different place to look, which is why the order in section 3A goes crictl → logs → kubelet journal.
- **Fault 4.** The API server is up, so the mirror pod's status tells you everything. Don't SSH until `k describe pod -n kube-system kube-scheduler-cp1` stops being useful.
- **Fault 5.** Existing pods keep running, so the cluster looks healthy. The probe is "does a change to desired state produce a change in actual state". Scale, or delete a pod and see if it comes back.
- **Fault 6.** Running pods on worker1 keep serving because the shims survive. That's not a sign containerd is fine.

### What happens to workloads while a node is down

Pods on a `NotReady`/`Unreachable` node get the `node.kubernetes.io/not-ready` or `unreachable` taint with `NoExecute`. Every pod has a default toleration of **300 s** for those taints, so after about 5 minutes the controller-manager evicts them, and Deployments recreate them elsewhere. On a 2-node lab where cp1 is tainted, the replacements go **Pending**. Don't misread that as a scheduling fault. Fix the node and they come back.

### Editing static pod manifests safely (acceptance criterion 3)

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak   # ALWAYS first
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
watch -n2 'sudo crictl ps -a --name kube-apiserver'       # new container within ~20–60 s
```

- The backup goes in `/tmp`, **never** in `/etc/kubernetes/manifests/`. The kubelet runs every file in that directory as a pod, and a second apiserver fights the first for port 6443. (It skips files whose names start with a dot, so vim's `.swp` files are harmless.)
- If you made it worse, `sudo cp /tmp/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml` puts you back where you started, which is better than guessing.
- To force the kubelet to recreate the pod: `sudo mv` the manifest to `/tmp`, wait until `crictl ps` shows the container gone, then move it back. `systemctl restart kubelet` also works.
- `diff /tmp/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml` before you leave the node confirms you changed only what you meant to.

---

## 5. Hands-on walkthrough on cp1 + worker1

Take a VM snapshot of both nodes first (`multipass snapshot cp1 ; multipass snapshot worker1`). A broken etcd manifest you can't undo costs the evening.

**Step 0: baseline.** Record what healthy looks like, so you can compare:

```bash
k get nodes -o wide ; k get pods -n kube-system -o wide
sudo crictl ps                                   # on cp1: etcd, apiserver, kcm, scheduler, kube-proxy, calico, coredns
sudo grep -E 'clientCAFile|staticPodPath' /var/lib/kubelet/config.yaml
sudo grep -E 'image:|etcd-servers' /etc/kubernetes/manifests/kube-{apiserver,scheduler}.yaml
k create deploy web --image=nginx                # your probe workload
```

**Step 1: get someone (or something) to break it.** Doing it yourself means you already know the answer. Put this on each node as `/root/chaos.sh`. It applies one random fault, saves backups in `/root/chaos-bak/`, and writes the answer to a file you open only after fixing it:

```bash
#!/usr/bin/env bash
# usage: sudo bash /root/chaos.sh      (cp1: control-plane faults; worker1: node faults)
set -e
M=/etc/kubernetes/manifests; B=/root/chaos-bak; mkdir -p $B
cp -a $M /var/lib/kubelet/config.yaml /etc/kubernetes/kubelet.conf $B/ 2>/dev/null || true
if [ -f $M/kube-apiserver.yaml ]; then
  faults=(etcd-port sched-image kcm-kubeconfig kcm-command apiserver-yaml kubelet-stop)
else
  faults=(kubelet-stop kubelet-cafile containerd-stop kubelet-server-port kubelet-disabled)
fi
f=${faults[$RANDOM % ${#faults[@]}]}
case $f in
  etcd-port)        sed -i 's#127.0.0.1:2379#127.0.0.1:2380#' $M/kube-apiserver.yaml ;;
  sched-image)      sed -i 's#\(image: registry.k8s.io/kube-scheduler:\).*#\1v9.9.9#' $M/kube-scheduler.yaml ;;
  kcm-kubeconfig)   sed -i 's#--kubeconfig=/etc/kubernetes/controller-manager.conf#--kubeconfig=/etc/kubernetes/controller-manager.cnf#' $M/kube-controller-manager.yaml ;;
  kcm-command)      sed -i 's#- kube-controller-manager$#- kube-controller-manger#' $M/kube-controller-manager.yaml ;;
  apiserver-yaml)   sed -i '0,/    - kube-apiserver/s//   - kube-apiserver/' $M/kube-apiserver.yaml ;;
  kubelet-stop)     systemctl stop kubelet ;;
  kubelet-disabled) systemctl disable --now kubelet ;;
  kubelet-cafile)   sed -i 's#/etc/kubernetes/pki/ca.crt#/etc/kubernetes/pki/ca.crtx#' /var/lib/kubelet/config.yaml; systemctl restart kubelet ;;
  kubelet-server-port) sed -i 's#:6443#:6444#' /etc/kubernetes/kubelet.conf; systemctl restart kubelet ;;
  containerd-stop)  systemctl stop containerd ;;
esac
echo "$f  ($(date +%T))" > $B/ANSWER
echo "Fault injected. Start your timer. Answer is in $B/ANSWER (don't peek)."
```

`kcm-command` mirrors a real exam trick: one letter wrong in `command:`, which shows up as `RunContainerError` / `executable file not found in $PATH`. `apiserver-yaml` breaks indentation so you practise the "no container at all, read the kubelet journal" path. `kubelet-server-port` is a killer.sh favourite: the kubelet runs fine but logs `connection refused` to `cp1:6444`.

**Step 2: run the six story faults, timed.** ⏱ Target **< 6 min each** (acceptance criterion 1). For each one:

1. On the right node, `sudo bash /root/chaos.sh` (re-run until you get the fault you want, or do the six by hand with the `sed` lines above).
2. Wait 60 s. Start the timer.
3. From cp1, follow section 3A or 3B **in order**. Write the first evidence line you found in the story Notes.
4. Fix it. Stop the timer only when the **probe** passes:

| Fault | Probe that proves it's fixed |
|---|---|
| kubelet stopped / bad CA / containerd / kubelet.conf | `k get nodes` → worker1 `Ready`, and `systemctl is-enabled kubelet containerd` → `enabled` |
| apiserver | `k get nodes` answers, and `crictl ps --name kube-apiserver` shows ATTEMPT not climbing for 1 min |
| scheduler | `k run probe --image=nginx` → `Running` with a NODE |
| controller-manager | `k scale deploy web --replicas=3` → 3 pods; `k delete pod <one>` → replaced |

5. Compare with `/root/chaos-bak/ANSWER`, and `diff` the manifest against the backup.

**Step 3: the log tools (task 7).** Once each:

```bash
sudo crictl ps -a | head ; sudo crictl pods | head
sudo crictl logs --tail 5 $(sudo crictl ps -a --name etcd -q | head -1)
sudo journalctl -u kubelet -f          # in a second terminal while you run: sudo systemctl restart kubelet
sudo ls /var/log/pods/ ; sudo tail -5 /var/log/pods/kube-system_etcd-cp1_*/etcd/*.log
sudo journalctl -u containerd -n 20 --no-pager
```

**Step 4: cluster information (task 8).** These are also typical one-line exam questions ("write the name of the pod using the most CPU to /opt/…"):

```bash
k cluster-info
k get events -A --sort-by=.lastTimestamp | tail
k top nodes
k top pods -A --sort-by=cpu --no-headers | head -1 | awk '{print $2}' > /tmp/top-pod.txt
k logs web-xxx -c nginx --since=10m > /tmp/web.log    # "container output streams" in the curriculum
```

**Step 5: from memory.** Close this page and say the 3A order out loud, then write it into the story Notes (acceptance criterion 2).

**Cleanup:** `k delete deploy web ; k delete pod probe`. Check `k get nodes` and `k get pods -n kube-system` are all healthy before tomorrow's mock. `rm -rf /root/chaos-bak` once everything's restored.

---

## 6. Traps that cost points

- **Fixing on the wrong machine.** The exam gives you a base node and tells you to `ssh` to others. After fixing worker1, `exit` back before running the next question's `kubectl config use-context`. Check your prompt (`hostname`) before every edit.
- **Forgetting `sudo -i`.** Without root, `crictl`, the manifests and the kubelet config all fail with permission errors that look like other problems.
- **`systemctl start` without `enable`.** The node comes back now, but a disabled kubelet is still a fault if the grader checks it or the node reboots. Use `systemctl enable --now`.
- **Restarting the kubelet before fixing containerd.** The kubelet comes up into the same broken state. Bottom layer first.
- **Editing the manifest and immediately re-editing.** The kubelet needs 20–60 s to notice and recreate the pod. Each new edit restarts the wait. Watch `crictl ps -a`.
- **Backup inside `/etc/kubernetes/manifests/`.** Two apiservers, one port, endless restarts. `/tmp` only.
- **Reading `journalctl -u kubelet -f` from the top of a restart loop.** It's noise. Use `-n 50` and find the **first** error line after `Started kubelet`.
- **Blaming the scheduler for a Pending pod that has a `FailedScheduling` event.** The scheduler is working; the event tells you why it can't place the pod.
- **Assuming the kubelet config uses flags.** kubeadm puts most settings in `/var/lib/kubelet/config.yaml` (camelCase fields: `clientCAFile`, `staticPodPath`). Flags live in `kubeadm-flags.env` and the systemd drop-in.
- **Fixing the symptom.** Deleting a static pod's mirror (`k delete pod kube-scheduler-cp1`) does nothing, and adding a toleration to make a Pending pod schedule onto cp1 hides a broken worker. Fix the root cause the task describes.
- **Stopping at "Running".** A component that's Running but crash-looping every 2 minutes will be failing when the grader looks. Watch the ATTEMPT or RESTARTS count for a minute.
- **Spending 15 minutes on one broken node.** If section 3B hasn't found it in 6 minutes, flag the question and come back. The other questions are worth the same.

---

## 7. Self-check: answer before you open the lab

1. `kubectl get nodes` returns `The connection to the server 192.168.64.10:6443 was refused`. List the first four things you check, in order, and the command for each.
2. worker1 is `NotReady`. `systemctl status kubelet` says `activating (auto-restart)`. What does that tell you, and what's your next command?
3. A new pod has been `Pending` for 3 minutes and `k describe pod` shows **no events**. Which component do you suspect, and how do you confirm it without SSH?
4. You scale a Deployment from 1 to 4. `k get deploy` shows `READY 1/4` and no new pods appear, not even Pending ones. What's broken, and why does everything else look healthy?
5. You edited `kube-apiserver.yaml`, waited a minute, and `crictl ps -a --name kube-apiserver` shows no new container at all. Where do you look, and what are the two likely causes?
6. You stop containerd on worker1. Ten seconds later, `curl` to an nginx pod on worker1 still works. Why, and does that mean containerd isn't needed?
7. On which file would you fix a kubelet that logs `Get "https://192.168.64.10:6444/api/v1/nodes/worker1": dial tcp … connection refused`, and what do you run after editing it?
8. Why must the manifest backup go in `/tmp` and not next to the original as `kube-apiserver.yaml.bak`?

<details>
<summary>Answers</summary>

1. (1) Right context and server address: `k config current-context`, `k config view --minify | grep server`. (2) Anything listening on cp1: `curl -k https://localhost:6443/livez` or `sudo ss -tlnp | grep 6443`. (3) The container: `sudo crictl ps -a --name kube-apiserver` (Exited? ATTEMPT climbing? missing?). (4) Its logs: `sudo crictl logs <id>`, or `/var/log/pods/kube-system_kube-apiserver-*/`. Then the kubelet journal if there's no container, then kubelet/containerd status, then etcd, then certs.
2. The kubelet starts, fails, and systemd restarts it every ~10 s, so it's a configuration problem, not a stopped service. Next: `sudo journalctl -u kubelet -n 50 --no-pager` and find the first error after `Started kubelet`. It usually names the file and field (for example `unable to load client CA file …`). Fix `/var/lib/kubelet/config.yaml` (or `kubelet.conf` / `kubeadm-flags.env`), then `systemctl restart kubelet`.
3. kube-scheduler. A running scheduler that can't place a pod always records a `FailedScheduling` event. Confirm with `k get pods -n kube-system | grep scheduler` (ImagePullBackOff, CrashLoopBackOff, or missing), then `k describe pod -n kube-system kube-scheduler-cp1` and `k logs -n kube-system kube-scheduler-cp1`.
4. kube-controller-manager. The Deployment and ReplicaSet controllers live there, so nothing turns `replicas: 4` into new pods. Everything else looks fine because existing pods keep running, the API server keeps serving, and the scheduler has nothing new to schedule. Check `k get pods -n kube-system | grep controller-manager` and its logs or `crictl logs`.
5. The kubelet journal: `sudo journalctl -u kubelet -n 50 | grep -i apiserver`. Likely causes: a YAML syntax or indentation error (the kubelet can't parse the manifest, so it never creates a container), or an image it can't pull (wrong image name or tag; `crictl pods` may show a sandbox with no container). A bad flag would instead show an Exited container with logs.
6. Each running container is managed by its own `containerd-shim` process, which keeps running when the containerd daemon stops. Existing pods carry on, but the kubelet can't start, stop, probe-exec or list containers, the node goes `NotReady` and new pods can't start there. containerd is very much needed: `systemctl enable --now containerd`, then restart the kubelet if the node doesn't recover.
7. `/etc/kubernetes/kubelet.conf`, the kubelet's kubeconfig. Its `server:` points at the wrong port. Change `:6444` to `:6443`, then `sudo systemctl restart kubelet` and wait for `Ready`.
8. The kubelet treats every file in `/etc/kubernetes/manifests/` as a static pod, whatever its extension. A `.bak` copy becomes a second kube-apiserver pod competing for port 6443 (and, after you fix the original, the stale copy may still be broken), so you get restarts you can't explain. `/tmp` is outside `staticPodPath`.

</details>

---

**Done when:** every [Day 11 acceptance criterion](../stories/day-11-troubleshoot-cluster.md#acceptance-criteria) passes: all six faults found and fixed in under 6 minutes each, the "kubectl connection refused" order stated from memory, and every manifest edit preceded by a copy to `/tmp`. Tomorrow (Day 12) is the first killer.sh mock: it will hit you with at least one broken node or control-plane component, and today's fixed orders are what keep that question under 6 minutes.
