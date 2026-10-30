# Day 2 Lesson: kubeadm internals, static pods, certificates and etcd backup/restore

**Story:** [day-02-etcd-static-pods](../stories/day-02-etcd-static-pods.md) · **Read time:** ~40 min · **Then:** 2+ hours in the lab

**Why this matters:** Cluster Architecture, Installation & Configuration is 25% of the exam, and an etcd backup/restore task appears on almost every CKA. It's worth a lot of points, it's mechanical once practiced, and it's easy to wreck the cluster if you rush it. Everything in this lesson also feeds Troubleshooting (30%): when the API server is down, the files in this lesson are the only things you can still fix.

On EKS, etcd is behind an AWS API and you never back it up yourself. Here you'll take a snapshot with your own certs, wipe state, and bring the cluster back from a file.

> **Version note.** The [official CKA page](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) now says the exam is based on **Kubernetes v1.37** (Day 1 said v1.35; that has since moved). Your v1.34 lab is still fine for everything in this lesson. Recent Kubernetes releases ship **etcd 3.6.x**, and in etcd 3.6 `etcdctl` **no longer supports `snapshot status` or `snapshot restore`**. Those two commands moved to `etcdutl` ([etcd v3.6 announcement](https://etcd.io/blog/2025/announcing-etcd-3.6/)). `snapshot save` stays in `etcdctl`. Check your own version with `sudo crictl images | grep etcd`.

---

## 1. Static pods: how the control plane starts with no API server

The kubelet has two sources of pods:

| Source | Who creates it | How to delete it |
|---|---|---|
| **API server** (normal pods) | Scheduler binds it; kubelet watches the API | `kubectl delete pod` |
| **Local directory** (static pods) | The kubelet reads YAML files from `staticPodPath` | **Remove the file.** `kubectl delete` only deletes the mirror; it comes straight back. |

For each static pod the kubelet also creates a read-only **mirror pod** in the API so you can see it. The mirror's name is `<pod-name>-<node-name>`, which is why you see `kube-apiserver-cp1`.

```
/var/lib/kubelet/config.yaml
  staticPodPath: /etc/kubernetes/manifests      ← the kubelet polls this directory (~20s)
                 │
                 ├─ etcd.yaml                     → etcd-cp1
                 ├─ kube-apiserver.yaml           → kube-apiserver-cp1
                 ├─ kube-controller-manager.yaml  → kube-controller-manager-cp1
                 └─ kube-scheduler.yaml           → kube-scheduler-cp1
```

What the kubelet does when a file changes:

- **File added:** start the pod.
- **File edited:** kill the pod and start a new one with the new spec. A YAML typo means the pod just doesn't come back, and there's no error in `kubectl` (because the API server may be the pod that's broken). Look at `journalctl -u kubelet` instead.
- **File removed:** stop the pod.

Exam favorite: *"Create a static pod named X on node Y."* The trick is that the path might not be the default, so read it:

```bash
ssh worker1
sudo grep staticPodPath /var/lib/kubelet/config.yaml
# staticPodPath: /etc/kubernetes/manifests  (on workers this dir exists but is empty)
```

If the kubelet config has no `staticPodPath`, add it and `sudo systemctl restart kubelet`.

Also check how the kubelet was started, because a `--config` or `--pod-manifest-path` flag could point elsewhere:

```bash
systemctl cat kubelet                     # unit + drop-in 10-kubeadm.conf
cat /var/lib/kubelet/kubeadm-flags.env     # extra kubelet flags kubeadm wrote
```

---

## 2. The PKI: who uses which cert

kubeadm creates **three CAs** plus a key pair for service accounts. Learn the pattern rather than every file: each component has a cert it *serves* with, and a cert it uses as a *client* to talk to something else.

```
/etc/kubernetes/pki/
├── ca.crt / ca.key                        Cluster CA (10 years). Signs everything below except etcd and front-proxy.
├── apiserver.crt / .key                   API server's serving cert (SANs: IPs, kubernetes.default.svc, …)
├── apiserver-kubelet-client.crt / .key    API server → kubelet (logs, exec)
├── apiserver-etcd-client.crt / .key       API server → etcd  (signed by etcd CA!)
├── front-proxy-ca.crt / .key              CA for the aggregation layer
├── front-proxy-client.crt / .key          API server → extension API servers (metrics-server)
├── sa.key / sa.pub                        Signs/verifies ServiceAccount tokens (not a cert)
└── etcd/
    ├── ca.crt / ca.key                    etcd's own CA
    ├── server.crt / .key                  etcd serving cert on :2379
    ├── peer.crt / .key                    etcd ↔ etcd on :2380
    └── healthcheck-client.crt / .key      liveness probe client
```

The kubeconfigs in `/etc/kubernetes/` embed client certs signed by the cluster CA:

| File | Used by | Identity |
|---|---|---|
| `admin.conf` | you (`kubectl`) | group `kubeadm:cluster-admins` (bound to cluster-admin) |
| `super-admin.conf` | break-glass | group `system:masters` (bypasses RBAC; keep it off-node in real life) |
| `controller-manager.conf` | kube-controller-manager | `system:kube-controller-manager` |
| `scheduler.conf` | kube-scheduler | `system:kube-scheduler` |
| `kubelet.conf` | kubelet | points to `/var/lib/kubelet/pki/kubelet-client-current.pem`, auto-rotated |

### The kube-apiserver flags you must be able to explain

```bash
sudo grep -E -- '--(client-ca|tls-|etcd-|kubelet-client|proxy-client|requestheader-client|service-account-(key|signing))' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
```

| Flag | Points at | Purpose |
|---|---|---|
| `--client-ca-file` | `pki/ca.crt` | Trust anchor for client certs (kubectl, kubelets, controllers) |
| `--tls-cert-file` / `--tls-private-key-file` | `pki/apiserver.crt/.key` | What the API server presents on :6443 |
| `--etcd-servers` | `https://127.0.0.1:2379` | Where etcd is |
| `--etcd-cafile` | `pki/etcd/ca.crt` | Verify etcd's server cert |
| `--etcd-certfile` / `--etcd-keyfile` | `pki/apiserver-etcd-client.*` | Authenticate *to* etcd |
| `--kubelet-client-certificate` / `--kubelet-client-key` | `pki/apiserver-kubelet-client.*` | Authenticate to kubelets for `logs`/`exec` |
| `--proxy-client-cert-file` / `--proxy-client-key-file` | `pki/front-proxy-client.*` | Aggregated APIs |
| `--requestheader-client-ca-file` | `pki/front-proxy-ca.crt` | Trust headers from the front proxy |
| `--service-account-key-file` | `pki/sa.pub` | Verify SA tokens |
| `--service-account-signing-key-file` | `pki/sa.key` | Sign SA tokens |

Classic broken-cluster fault: one of these paths has a typo (`apiserver-etcd-client.crt` → `apiserver-etcd-clinet.crt`). The API server container exits immediately, `kubectl` gets "connection refused", and the answer is in the container log (section 5).

### Cert expiry

```bash
sudo kubeadm certs check-expiration     # leaf certs: 1 year; CAs: 10 years
sudo kubeadm certs renew all            # renews leaf certs, not CAs
# Renewed certs are only read at startup, so restart the 4 static pods:
cd /etc/kubernetes/manifests && sudo mkdir -p /tmp/m && sudo mv *.yaml /tmp/m/ && sleep 20 && sudo mv /tmp/m/*.yaml .
cp /etc/kubernetes/admin.conf ~/.kube/config   # admin.conf was renewed too
```

`kubeadm upgrade apply` renews all certs automatically, which is why clusters upgraded at least once a year rarely hit expiry. To read one cert directly:

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -issuer -dates -ext subjectAltName
```

---

## 3. etcd: what you're backing up

etcd is a Raft-replicated key-value store. Every Kubernetes object lives under `/registry/...` (for example `/registry/pods/default/web-abc`). On kubeadm it's a single-member cluster in a static pod:

```bash
sudo grep -E -- '--(data-dir|listen-client-urls|advertise-client-urls|cert-file|key-file|trusted-ca-file|name|initial-cluster)' \
  /etc/kubernetes/manifests/etcd.yaml
```

| Flag | Default on kubeadm | Why you care |
|---|---|---|
| `--listen-client-urls` | `https://127.0.0.1:2379,https://<ip>:2379` | Your `--endpoints` |
| `--trusted-ca-file` | `/etc/kubernetes/pki/etcd/ca.crt` | `etcdctl --cacert` |
| `--cert-file` / `--key-file` | `/etc/kubernetes/pki/etcd/server.crt/.key` | `etcdctl --cert/--key` |
| `--data-dir` | `/var/lib/etcd` | Inside the container; matched by a hostPath volume |
| `--name` / `--initial-cluster` | `cp1` / `cp1=https://<ip>:2380` | Only matter if restoring a multi-member cluster |

**The key mental model:** the container sees `--data-dir=/var/lib/etcd`. That path is a `volumeMount` named `etcd-data`, backed by a **hostPath volume** on the node. For a restore you change the **hostPath**. The container keeps seeing `/var/lib/etcd`, but now it's your restored data.

```yaml
# /etc/kubernetes/manifests/etcd.yaml (relevant parts)
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/var/lib/etcd          # container path: leave it
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
  volumes:
  - hostPath:
      path: /var/lib/etcd               # ← change this to /var/lib/etcd-restore
      type: DirectoryOrCreate
    name: etcd-data
```

You can change both `--data-dir` and the hostPath to the new path, and that works too, as long as the `mountPath` also matches. Changing only one of the three is what breaks restores. Changing only the hostPath is the smallest edit.

### etcdctl vs etcdutl

| Tool | Talks to | Use for |
|---|---|---|
| `etcdctl` | A **running** etcd over the network (needs endpoint + certs) | `snapshot save`, `endpoint health/status`, `member list`, `get` |
| `etcdutl` | **Files** on disk (no network, no certs) | `snapshot status`, `snapshot restore`, `defrag --data-dir` |

So: save with `etcdctl` (certs needed), check and restore with `etcdutl` (no certs). `ETCDCTL_API=3` is the default since etcd 3.4; adding it is harmless, and old exam guides still show it.

---

## 4. The exam procedure: backup

Timed target: **under 4 minutes**. Read the paths; never type them from memory, because the exam may use non-default certs or a non-default endpoint.

```bash
# 0. Right host? The task usually says "on cp1" or gives an ssh target.
hostname

# 1. Read paths from the manifest (the source of truth)
sudo grep -E 'trusted-ca-file|cert-file|key-file|listen-client-urls' /etc/kubernetes/manifests/etcd.yaml
# Pick the non-peer ones: --cert-file, --key-file, --trusted-ca-file.

# 2. Save
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db

# 3. Verify
sudo etcdutl snapshot status /opt/etcd-backup.db -w table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
```

If `etcdctl` says `context deadline exceeded`, it's nearly always a wrong cert, wrong endpoint, or a missing `https://`. Test with `... endpoint health` before blaming the snapshot.

`--cert/--key` can also be `apiserver-etcd-client.crt/.key` from `/etc/kubernetes/pki/`. Both are signed by the etcd CA and both work. Use what the task gives you.

---

## 5. The exam procedure: restore

Timed target: **under 8 minutes**.

```bash
# 1. Restore to a NEW directory (it must not exist yet)
sudo etcdutl snapshot restore /opt/etcd-backup.db --data-dir /var/lib/etcd-restore

# 2. Point etcd at it: edit the etcd-data hostPath
sudo vim /etc/kubernetes/manifests/etcd.yaml
#   volumes:
#   - hostPath:
#       path: /var/lib/etcd-restore
#     name: etcd-data

# 3. Wait. The kubelet recreates etcd, the API server reconnects.
watch sudo crictl ps          # etcd, then kube-apiserver, come back in 1–2 min
k get pods -A                 # the objects you created after the snapshot are gone
```

Optional, more careful version (what the [kubernetes.io page](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) recommends): stop the API server (and scheduler and controller manager) before restoring, so nothing writes to the old etcd mid-switch. On kubeadm that means moving their manifests out of `/etc/kubernetes/manifests/`, restoring, then moving them back. In a single-node exam task the hostPath edit alone is normally enough. Use the careful version if the task says so.

If the API server doesn't come back after a couple of minutes, kick the kubelet once with `sudo systemctl restart kubelet`. Then go straight to logs.

### Where the logs are when kubectl is down

| What | Command |
|---|---|
| Static pod containers (incl. exited) | `sudo crictl ps -a \| grep -E 'etcd\|apiserver'` |
| A container's log | `sudo crictl logs <container-id>` |
| Same logs as files | `/var/log/pods/kube-system_<pod>_<uid>/<container>/0.log` (symlinked from `/var/log/containers/`) |
| Why the kubelet didn't start a pod (YAML errors) | `sudo journalctl -u kubelet -n 50 --no-pager` |
| containerd itself | `sudo journalctl -u containerd -n 50` |

The kubelet and containerd are systemd services, so their logs are in journald. The four control-plane components are containers, so their logs are in `crictl` and `/var/log/pods`. That split is acceptance criterion 3.

---

## 6. Hands-on walkthrough on cp1 + worker1

Take a VM snapshot first (`multipass snapshot cp1`). Restores go wrong while you learn them.

**Step 0: get etcdutl/etcdctl on cp1.** On the exam they're preinstalled. In your lab, Ubuntu's `etcd-client` package is too old (no `etcdutl`), so install binaries that match your etcd image:

```bash
sudo crictl images | grep etcd                   # e.g. registry.k8s.io/etcd  3.6.x-0
ETCD_VER=v3.6.x                                  # use the version from the line above
curl -fsSL https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  | tar xz -C /tmp
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcd{ctl,utl} /usr/local/bin/
etcdctl version && etcdutl version
```

**Step 1: read the manifests (task 1).**

```bash
sudo ls -l /etc/kubernetes/manifests/
sudo grep -E -- '--(client-ca|tls-|etcd-|kubelet-client|service-account)' /etc/kubernetes/manifests/kube-apiserver.yaml
```

Write one line per flag in your story Notes, using the table in section 2.

**Step 2: map PKI and kubeconfigs (task 2).**

```bash
sudo ls /etc/kubernetes/pki /etc/kubernetes/pki/etcd /etc/kubernetes/*.conf
sudo grep -h -E 'client-certificate|server:' /etc/kubernetes/{scheduler,controller-manager,kubelet}.conf
sudo grep -E 'kubeconfig|authentication|authorization' /etc/kubernetes/manifests/kube-scheduler.yaml
```

**Step 3: kubelet config (task 3).**

```bash
sudo grep -E 'staticPodPath|cgroupDriver|clientCAFile|rotateCertificates' /var/lib/kubelet/config.yaml
```

**Step 4: static pod on worker1 (task 4).** Generate on cp1, create on worker1:

```bash
k run static-web --image=nginx --port 80 $do      # copy the output
ssh worker1
sudo grep staticPodPath /var/lib/kubelet/config.yaml
sudo vim /etc/kubernetes/manifests/static-web.yaml   # paste
exit
k get pod -o wide | grep static-web               # static-web-worker1, Running on worker1
k delete pod static-web-worker1                   # watch: it comes back
ssh worker1 sudo rm /etc/kubernetes/manifests/static-web.yaml
k get pod -w                                      # gone within ~20s
```

**Step 5: cert expiry (task 5).**

```bash
sudo kubeadm certs check-expiration
```

Note the two sections: leaf certs (~1 year from Day 1) and CAs (~10 years). `RESIDUAL TIME` is what an exam question would ask for.

**Step 6: backup, change, restore (task 6, the acceptance drill).** Time yourself.

```bash
k create ns before-backup
# ⏱ start: snapshot + status (section 4) → stop. Target < 4 min.
k create ns after-backup
k create deploy after --image=nginx -n after-backup
# ⏱ start: restore + hostPath edit + wait (section 5) → stop when `k get ns` works. Target < 8 min.
k get ns            # before-backup present, after-backup gone
```

Then do it again from scratch: restore your VM snapshot, and redo steps 6's two timed sections without looking at this page. Twice is what makes it stick.

**Step 7: a static pod failure.** Break the API server on purpose so you've seen the symptom:

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/apiserver.bak
sudo sed -i 's#apiserver-etcd-client.crt#apiserver-etcd-clinet.crt#' /etc/kubernetes/manifests/kube-apiserver.yaml
k get nodes                                  # connection refused (after ~30s)
sudo crictl ps -a | grep apiserver           # Exited
sudo crictl logs $(sudo crictl ps -a --name kube-apiserver -q | head -1)   # "no such file or directory"
sudo cp /root/apiserver.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

Never leave a backup copy *inside* `/etc/kubernetes/manifests/`. The kubelet would try to run it as a second pod.

---

## 7. Traps that cost points

- **Restoring into an existing directory.** `etcdutl snapshot restore` refuses if `--data-dir` exists. Always use a new path.
- **Editing only one of `--data-dir`, `mountPath`, `hostPath`.** The safe single edit is the hostPath.
- **Using `--peer-cert-file` paths for `etcdctl`.** You want the client-facing `--cert-file`/`--key-file`/`--trusted-ca-file`.
- **Forgetting `https://` in `--endpoints`.** The result is a hang until `context deadline exceeded`.
- **Running `etcdctl snapshot restore` on etcd 3.6.** It's gone; use `etcdutl`. Read the error if one tool fails, and switch.
- **Wrong host.** The snapshot path the task names is on a specific machine. Check `hostname` before and after `ssh`.
- **Backup file left inside the manifests directory**, or a swap file from vim (`.etcd.yaml.swp`). Edit elsewhere, or quit vim cleanly.
- **Panic-waiting.** After editing a static pod manifest, give it 60–90 seconds and watch `crictl ps` rather than re-editing. Each edit restarts the countdown.
- **`kubectl delete` on a static pod.** It only deletes the mirror pod. The task's verifier will still see it running.

---

## 8. Self-check: answer before you open the lab

1. You need `--cacert`, `--cert` and `--key` for `etcdctl snapshot save`. Which three flags in `etcd.yaml` do you copy them from?
2. After restoring to `/var/lib/etcd-restore`, which single field in `etcd.yaml` do you change, and why not `--data-dir` alone?
3. Which of `snapshot save`, `snapshot status` and `snapshot restore` need certs, and which tool runs each on etcd 3.6?
4. You `kubectl delete pod kube-scheduler-cp1`. What happens, and how do you actually stop it?
5. The API server is down and `kubectl` gives "connection refused". Name two places you'd read its logs.
6. Which cert does the API server use to authenticate to etcd, and which CA signed it?
7. You ran `kubeadm certs renew all`. Why is the cluster still using the old certs, and what do you do?
8. A task says "create a static pod on node02". What's the first command you run after `ssh node02`?

<details>
<summary>Answers</summary>

1. `--trusted-ca-file` → `--cacert`, `--cert-file` → `--cert`, `--key-file` → `--key`. Not the `--peer-*` ones.
2. The `hostPath.path` of the `etcd-data` volume. `--data-dir` is a path *inside the container*, and it's backed by a mount of that hostPath. Changing only `--data-dir` points etcd at a container path that isn't mounted, so etcd starts with an empty data dir: a "fresh" cluster with nothing in it.
3. Only `snapshot save` talks to a live etcd, so only it needs endpoint and certs; it runs with `etcdctl`. `snapshot status` and `snapshot restore` work on the file, need no certs, and run with `etcdutl` (removed from `etcdctl` in 3.6).
4. The mirror pod is deleted and immediately recreated, because the kubelet still has `/etc/kubernetes/manifests/kube-scheduler.yaml`. To stop it, move that file out of the manifests directory.
5. Any two of: `crictl ps -a` + `crictl logs <id>`; `/var/log/pods/kube-system_kube-apiserver-cp1_*/kube-apiserver/0.log`; `/var/log/containers/`. If the container never even started, `journalctl -u kubelet`.
6. `/etc/kubernetes/pki/apiserver-etcd-client.crt` (flags `--etcd-certfile`/`--etcd-keyfile`), signed by the **etcd CA** (`pki/etcd/ca.crt`), not the cluster CA.
7. Components read certs only at startup. Restart the four static pods (move the manifests out and back), and copy the renewed `admin.conf` to `~/.kube/config`.
8. `sudo grep staticPodPath /var/lib/kubelet/config.yaml`, because the path may not be the default.

</details>

---

**Done when:** every [Day 2 acceptance criterion](../stories/day-02-etcd-static-pods.md#acceptance-criteria) passes, timed and without notes. Tomorrow (Day 3) builds on this: `kubeadm upgrade` rewrites these same static pod manifests and renews these certs, and you'll take an etcd snapshot before you start, as you would in production.
