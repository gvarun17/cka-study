# Day 12 Lesson: Mock exam 1, exam strategy, pacing and the highest-value tasks

**Story:** [day-12-mock-1](../stories/day-12-mock-1.md) · **Read time:** ~30 min (read it **before** you activate killer.sh) · **Then:** 2 h mock + ~1 h review + app-troubleshooting drills

**Why this matters:** Today covers every domain: Troubleshooting **30%**, Cluster Architecture **25%**, Services & Networking **20%**, Workloads & Scheduling **15%**, Storage **10%**. You already know most of the material. What decides Friday is **execution**: whether you work on the right host, whether you leave a task done instead of half-done, and whether you keep one stuck question from costing you three others. Today is the first time you test that under a clock. This lesson is about pacing and habits. The content is in Days 2–11. Section 4 is a one-page "minimum correct answer" for the tasks that carry the most points, so you can check yourself against it after the mock.

> **Version and format notes (checked today).**
> - The [CNCF curriculum repo](https://github.com/cncf/curriculum) has `CKA_Curriculum_v1.37.pdf`, and the [LF exam tips page](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad) says *"The CKA environment is currently running Kubernetes v1.37"*. Your lab is v1.35 after Day 3. Nothing you'll practise today differs between the two.
> - Exam format (same page): **15–20 performance-based tasks, 2 hours**, PSI Bridge with the PSI Secure Browser. The pass mark is 66% (see [exam-day](../cheatsheets/exam-day.md)).
> - **Every task runs on its own host.** *"You must complete each task in this exam on a designated host. An infobox at the start of each task provides you with instructions to ssh into the designated host."* Each host already has **`kubectl` with the `k` alias and bash completion**, plus `yq`, `curl`, `wget` and `man`. The **base** machine (hostname `base`) has none of these. **Nested ssh is not supported**, so `exit` back to `base` after every task.
> - Copy/paste in the exam terminal is **Ctrl+Shift+C / Ctrl+Shift+V**. Close a tab with **Ctrl+Alt+W**, not Ctrl+W, which closes the browser tab. The INSERT key is disabled, so use `i` in vim.
> - [killer.sh](https://training.linuxfoundation.org/blog/exam-simulators/): **two sessions**, each open for **36 hours from activation**, **20–25 questions**, and **the same questions in both sessions**. So Day 14's session 2 is a speed and accuracy re-run, not a new exam. Plan for that (section 6).

---

## 1. What changes now that each task has its own host

The [speed-setup](../cheatsheets/speed-setup.md) sheet was written for "one terminal, many contexts". The current exam is "one terminal, ssh to a new host per task". That changes three habits:

| Old habit | What to do now | Why |
|---|---|---|
| Set aliases once in the first 2 minutes | `k` and completion are **already there on every host**. Don't touch `~/.bashrc` | The setup is per host. Doing it 17 times wastes a minute each |
| `export do=...` once | If a task needs generated YAML, type `export do="--dry-run=client -o yaml"` as the first line **on that host** (5 seconds), or type the flags out | Each ssh is a fresh shell. `$do` is empty on the next host, and `k run x $do` with an empty `$do` **creates the pod** instead of printing YAML |
| `kubectl config use-context` per question | Run whatever the infobox says: usually `ssh <host>`, sometimes also a context. Then check with `hostname; k config current-context` | Work on the wrong host scores zero, and nothing warns you |
| `~/.vimrc` once | Only on hosts where you'll edit YAML: `echo 'set et ts=2 sw=2' >> ~/.vimrc` (or `:set et ts=2 sw=2` inside vim) | Tabs in YAML fail `kubectl apply` with `found character that cannot start any token` |

The `$do` trap is the nasty one. `k run tmp --image=nginx $do > p.yaml` on a fresh host creates a real pod called `tmp` and writes `pod/tmp created` into `p.yaml`. If a question checks "only these pods exist in namespace X", that stray pod can cost you the task. Check with `echo $do` if you're unsure.

### The per-task loop (memorise it)

```
read the whole task (twice for the numbers: names, namespaces, ports, paths)
  → ssh <host> from the infobox                 hostname
  → sudo -i only if you need node files/systemd
  → do it (imperative first, $do + vim second)
  → VERIFY with k get / describe / curl / a test pod
  → exit (back to base)                          hostname → base
  → note the task # + "done" / "flag" on scratch paper
```

Write the namespace into every command (`-n prod`) instead of relying on a default. If a task says to write output to a file (`/opt/answers/q4.txt`), write it **on the host the task names**, at exactly that path, because that's where the grader reads it.

---

## 2. Pacing: the three-pass plan

120 minutes ÷ ~17 tasks ≈ **7 minutes per task**. The weights aren't equal: each task shows its weight (e.g. 4%, 7%, 13%). A 13% "fix the broken cluster" task is worth three 4% tasks. Plan around that.

```
 0–2 min   Skim the whole list once. Mark each task: Q (quick), M (medium), H (heavy/node work).
 Pass 1    Do every Q and M task in order. Hard cap: 8 min. At 8 min, write down where you
 (~75 min) got to, flag it, exit, move on. Don't "just try one more thing".
 Pass 2    Heavy tasks: etcd restore, upgrade, broken node/control plane, anything flagged.
 (~35 min) Biggest weight first.
 Pass 3    Last 10 min: re-verify flagged and heavy tasks with k get. Fix anything half-done.
```

Why this order: Pass 1 gets you around 60% of the points with little risk. The heavy tasks are where you might get stuck, and you don't want them eating the time for six easy tasks. Exception: if a heavy task is high-weight and you know you can do it fast (say etcd backup and restore at 12%), do it in Pass 1. You've drilled it.

**Time budget per task type** (your Day 2–11 acceptance criteria were set to fit these):

| Task type | Target | Hard cap |
|---|---|---|
| Create pod/deploy/svc/cm/secret/job, scale, set image, rollout undo | 2–3 min | 5 |
| RBAC (SA + Role + Binding + `auth can-i`) | 3–4 | 6 |
| NetworkPolicy | 4–5 | 8 |
| Ingress / Gateway + HTTPRoute | 4–6 | 8 |
| PV + PVC + pod mount | 4–5 | 8 |
| Scheduling (taint/toleration, affinity, nodeSelector), sidecar, HPA | 3–5 | 7 |
| Helm install/upgrade, Kustomize, CRD/operator queries | 3–5 | 7 |
| etcd backup + restore | 6–8 | 10 |
| kubeadm upgrade (control plane + one worker) | 10–12 | 15 |
| Node NotReady / broken control-plane component | 4–6 | 10 |

If you reach the hard cap, leave the task in the **best state you can** before you flag it. Partial work sometimes scores (the PVC exists even if the pod doesn't mount it yet). A broken static pod manifest doesn't, and it might break later tasks that use the same cluster. If you edited a manifest and you're giving up, **restore the `/tmp` backup**.

---

## 3. Habits that cost the most points

Ranked by how many points they cost people on killer.sh and the real exam:

1. **Wrong host or context.** Check `hostname` before every edit.
2. **Not verifying.** "Created" doesn't mean "working". After every task, run one command that proves the result: `k get pod -o wide` shows Running, `k auth can-i ... --as` says `yes`, curl from a test pod answers, `k get pvc` shows Bound.
3. **Misreading names.** `web-svc` vs `websvc`, `ns: production` vs `prod`, port `8080` vs `80`. Copy names from the task text with Ctrl+Shift+C and paste them. Don't retype.
4. **Stopping at "Running" after a node or control-plane fix.** Watch for a minute. ATTEMPT/RESTARTS must stay flat (Day 11).
5. **Fighting vim for YAML.** If `k apply` fails on indentation, regenerate with `$do` instead of fixing it line by line. For a long paste: `:set paste`, then `i`, then Ctrl+Shift+V.
6. **Reading the docs from the top.** Search kubernetes.io for a page title you already know ("Configure Liveness", "Network Policies", "PersistentVolumes", "Upgrading kubeadm clusters", "Operating etcd clusters") and jump to the YAML block. Know your 10 pages in advance (section 5).
7. **Deleting and recreating something you were told to modify.** If the task says "update deployment X", `k edit` / `k set` / `k patch`. Recreating it changes the UID and can lose fields the grader checks (labels, annotations, revision history).

---

## 4. The highest-value tasks: minimum correct answer

These are condensed from the earlier lessons. Each one is the **minimal sequence that scores**, plus how to verify it. After the mock, compare what you typed with these.

### etcd backup and restore (Day 2) · cluster architecture

```bash
ssh <host>; sudo -i
grep -E 'cert-file|key-file|trusted-ca-file|listen-client' /etc/kubernetes/manifests/etcd.yaml
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
etcdutl snapshot status /opt/etcd-backup.db -w table           # verify: has a hash and key count

etcdutl snapshot restore /opt/etcd-backup.db --data-dir /var/lib/etcd-restore   # NEW dir
cp /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
vim /etc/kubernetes/manifests/etcd.yaml      # ONLY the etcd-data hostPath.path → /var/lib/etcd-restore
watch crictl ps --name etcd                  # new container; then k get pods -A answers
```

If the exam node only has `etcdctl` and its version is 3.5, `etcdctl snapshot restore` still works there. Check `etcdctl version` first. With 3.6 it has to be `etcdutl` (Day 2 version note).

### kubeadm upgrade, control plane then worker (Day 3)

```bash
# control plane
apt-mark unhold kubeadm && apt-get update && apt-get install -y kubeadm='1.37.x-*' && apt-mark hold kubeadm
kubeadm upgrade plan
kubeadm upgrade apply v1.37.x            # 'v' + three-part version here, package format above
k drain <cp> --ignore-daemonsets          # from a host with kubectl
apt-mark unhold kubelet kubectl && apt-get install -y kubelet='1.37.x-*' kubectl='1.37.x-*' && apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
k uncordon <cp>
# worker: same, but `kubeadm upgrade node` instead of plan/apply
```

Verify with `k get nodes` (it shows the **kubelet** version) and `kubeadm version`. If the apt repo points at the old minor, edit `/etc/apt/sources.list.d/kubernetes.list` first (`v1.36` → `v1.37`). Check the repo before you start, not after an install fails.

### Node NotReady / broken control plane (Day 11) · troubleshooting

Use the fixed orders from Day 11 §3: **3A** for `kubectl` connection refused, **3B** for NotReady. Finish every fix with `systemctl enable --now`, a `/tmp` backup before any manifest edit, and a probe that exercises the component (create a pod for the scheduler, scale a Deployment for the controller-manager).

### RBAC (Day 4)

```bash
k create sa deployer -n ci
k create role deployer --verb=create,get,list,update --resource=deployments -n ci     # or clusterrole
k create rolebinding deployer --role=deployer --serviceaccount=ci:deployer -n ci
k auth can-i create deployments --as=system:serviceaccount:ci:deployer -n ci          # yes
k auth can-i delete deployments --as=system:serviceaccount:ci:deployer -n ci          # no
```

### NetworkPolicy (Day 9)

Start from a default deny if the task asks for one, then allow exactly what the task names. With egress rules, **remember DNS** (UDP and TCP 53 to kube-system). Before and after, test with `k exec <client> -- wget -qO- -T2 <svc>.<ns>` from an allowed pod and a denied one. A policy you haven't tested is a guess.

### Services, Ingress, Gateway (Days 8–9)

```bash
k expose deploy web --port=80 --target-port=8080 -n app          # check: k get endpointslices -n app
k create ingress web -n app --class=nginx --rule="web.example.com/=web:80"
# Gateway API: k get gatewayclass first, then Gateway + HTTPRoute (Day 9 §5 YAML), parentRefs → the Gateway
curl -s -H 'Host: web.example.com' http://<node-ip>:<nodeport>/
```

### Storage (Day 10)

PV (`capacity`, `accessModes`, `storageClassName`, `hostPath`) → PVC with the **same** `storageClassName` and accessMode, size ≤ PV → pod `volumes.persistentVolumeClaim.claimName`. Verify `k get pv,pvc` shows `Bound`, then `k exec` and write a file. If it stays `Pending`, read `k describe pvc`. The usual causes are a class mismatch, `WaitForFirstConsumer`, or the PVC asking for more than the PV.

### Workloads, scheduling, Helm (Days 5–7)

Sidecar = `initContainers` with `restartPolicy: Always`. HPA = `k autoscale deploy web --min=2 --max=5 --cpu=50%` (`--cpu-percent=50` on older kubectl; check `k autoscale -h`), and the pods **must** have CPU requests. Taint → toleration with the same key/value/effect. Helm: `helm repo add`, `helm search repo`, `helm install -n X --create-namespace --set key=val`, `helm upgrade`, `helm list -A`, `helm rollback`.

---

## 5. Your 10 doc pages: know the title, jump to the YAML

Find these in kubernetes.io search with one term, before Friday. The exam only allows the docs, so practise getting there quickly:

| Search term | Page | What you copy |
|---|---|---|
| `etcd backup` | Operating etcd clusters for Kubernetes | snapshot save flags |
| `kubeadm upgrade` | Upgrading kubeadm clusters | apt sequence |
| `network policy` | Network Policies | the full example `NetworkPolicy` |
| `ingress` | Ingress | minimal Ingress + `pathType` |
| `httproute` | Gateway API (concepts) | Gateway + HTTPRoute |
| `persistent volumes` | Persistent Volumes / Configure a Pod to Use a PV | PV, PVC, pod volume |
| `liveness` | Configure Liveness, Readiness and Startup Probes | probe blocks |
| `taint` | Taints and Tolerations | toleration block |
| `affinity` | Assigning Pods to Nodes | nodeAffinity block |
| `sidecar` | Sidecar Containers | init container with `restartPolicy: Always` |

Also `kubectl cheat sheet` (Quick Reference) for jsonpath and `--sort-by` one-liners.

---

## 6. Running killer.sh today

**Before activating:** you need 3.5 hours free in a row. Close Slack, put the phone away, and use the same laptop and screen you'll use Friday. You can redo the questions in the same environment for 36 hours after activation, so you can review tonight or tomorrow morning.

**During (2 h, strict):**

- Start a real timer, separate from killer.sh's.
- Follow the three-pass plan and the per-task loop. Keep a scratch log: `Q# · start time · done/flag`.
- **No notes, no lessons, no AI.** Only kubernetes.io/docs, kubernetes.io/blog and helm.sh/docs. An honest 45% teaches you more than a 75% you got with help.
- killer.sh is harder and longer than the real exam on purpose (20–25 questions instead of 15–20). Not finishing is normal. Use the pacing anyway.

**After (score, then review every question):**

1. Write down the score before you look at any solution.
2. Go through **every** solution, including the questions you got right. killer.sh solutions often show a faster route (`k create ... $do | yq`, jsonpath, `--sort-by`) that you should copy.
3. Log every miss **and** every slow question (over 8 min) in the story's Notes table. Pick one root cause per row:

| Root cause | What it looks like | Fix it on Day 13 by… |
|---|---|---|
| **Knowledge gap** | Didn't know the field, command or concept | Re-read the lesson section, then redo the task from memory |
| **Slow** | Got it right but over the cap | Drill the imperative route 3× until it's under target |
| **Misread** | Wrong name, namespace, port or path | Habit: copy-paste names, re-read the task before `exit` |
| **Wrong host/context** | Correct work on the wrong machine | Habit: `hostname` before every edit |
| **Didn't verify** | Thought it was done; grader disagreed | Habit: one proof command per task |

Day 13 uses this table directly, so make each row specific. "NetPol egress, forgot DNS port 53, 9 min" is useful. "NetPol bad" isn't.

**Session 2 on Day 14 has the same questions.** Don't memorise answers tonight. Fix the **causes** in the table. On Day 14 the goal is all of it, correct, in under 90 minutes, with every task verified.

---

## 7. Hands-on: application troubleshooting drills (remaining time)

The story's last task. These four faults make up most of the "application" part of the Troubleshooting domain, and killer.sh will have at least one. Run them on cp1 + worker1. Have someone else apply them if you can, or apply all four at once and fix them in random order.

```bash
k create ns drill
cat <<'EOF' | k apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: crash, namespace: drill}
spec:
  replicas: 1
  selector: {matchLabels: {app: crash}}
  template:
    metadata: {labels: {app: crash}}
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo starting; cat /config/app.conf; sleep 3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: pull, namespace: drill}
spec:
  replicas: 1
  selector: {matchLabels: {app: pull}}
  template:
    metadata: {labels: {app: pull}}
    spec:
      containers:
      - {name: web, image: nginx:1.277}
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: drill}
spec:
  replicas: 2
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web, tier: front}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports: [{containerPort: 80}]
---
apiVersion: v1
kind: Service
metadata: {name: web, namespace: drill}
spec:
  selector: {app: website}
  ports: [{port: 80, targetPort: 8080}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: probe, namespace: drill}
spec:
  replicas: 1
  selector: {matchLabels: {app: probe}}
  template:
    metadata: {labels: {app: probe}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        livenessProbe:
          httpGet: {path: /healthz, port: 80}
          initialDelaySeconds: 3
          periodSeconds: 5
        readinessProbe:
          httpGet: {path: /, port: 8080}
EOF
```

⏱ Target: **< 4 min each**, from first command to proof.

| Drill | What you see | Evidence command | Root cause | Fix | Proof |
|---|---|---|---|---|---|
| `crash` | `CrashLoopBackOff`, RESTARTS climbing | `k logs -n drill deploy/crash --previous` → `cat: can't open '/config/app.conf'` | Command expects a file nobody mounted | Create a ConfigMap `app-cfg` with key `app.conf` and mount it at `/config` (`k edit deploy`) | Pod `Running`, `k logs` prints the file |
| `pull` | `ErrImagePull` → `ImagePullBackOff` | `k describe pod -n drill -l app=pull \| tail` → `manifest unknown` / `not found` for `nginx:1.277` | Tag typo | `k set image -n drill deploy/pull web=nginx:1.27` | New ReplicaSet's pod `Running` |
| `web` svc | Pods Running, `curl web.drill` times out or is refused | `k get endpointslices -n drill -l kubernetes.io/service-name=web` → no endpoints; `k get pod -n drill --show-labels` | Selector `app=website` matches nothing; `targetPort` 8080 but nginx listens on 80 | `k edit svc web -n drill`: selector `app: web`, `targetPort: 80` | `k run t --rm -it -n drill --image=busybox:1.36 -- wget -qO- -T2 web` returns HTML |
| `probe` | `RESTARTS` climbing every ~15 s, `READY 0/1` | `k describe pod -n drill -l app=probe` events: `Liveness probe failed: HTTP probe failed with statuscode: 404`, `Readiness probe failed: … connect: connection refused` | Liveness path `/healthz` doesn't exist in nginx; readiness on the wrong port | Liveness path `/`, readiness port `80` | `READY 1/1`, restarts stop increasing for a minute |

Things this drill should teach you:

- **`--previous`** is how you read a crash-looping container's logs. Without it you often get an empty log from the container that just started.
- **ImagePullBackOff never fixes itself** with a correct image somewhere else. Change the spec.
- **A Service with no endpoints isn't a networking problem.** Check the selector against `--show-labels` and the `targetPort` against the container's port **before** you look at NetworkPolicy, kube-proxy or DNS.
- **A failing liveness probe** shows up as restarts with exit code 0/137 and clean logs. Only `describe` events show the probe failure. **A failing readiness probe** never restarts anything. It only keeps the pod out of the endpoints, so a Service looks broken.

**Bonus (if time):** `k scale deploy web -n drill --replicas=20` on your small worker, then work out from `describe` why some pods are `Pending` (`Insufficient cpu`/`memory`, or cp1's taint). That's a scheduling-troubleshooting task, not a broken scheduler, because there *is* a `FailedScheduling` event (Day 11 §1).

**Cleanup:** `k delete ns drill`.

---

## 8. Traps that cost points (mock and exam)

- **Activating killer.sh without the full time block.** The 36-hour clock runs whether you use it or not.
- **Pausing the mock to look something up in your notes.** The score stops meaning anything, and Day 13's plan is then built on wrong data.
- **`$do` on a fresh host.** It's empty there, so `k run x $do > x.yaml` creates the pod. Re-export it on each host or type the flags.
- **Forgetting `exit`.** Nested ssh isn't supported, so the next `ssh <host>` fails or lands you somewhere odd. Get used to seeing `base` in the prompt between tasks.
- **Spending Pass 1 on a 4% task you're stuck on.** Flag it at 8 minutes. Every task you skip past costs at most its own weight. Being stuck costs you the tasks you never reach.
- **Leaving a broken manifest behind when you give up.** Restore the `/tmp` backup before you flag the task.
- **Not reading the whole task.** The last line often says "write the result to /opt/…" or "make sure the pod is Running". Those are the graded parts.
- **Recreating instead of modifying.** If the task says "update" or "edit", use `edit`/`set`/`patch`/`scale`.
- **Treating the killer.sh score as a prediction.** It's harder and longer on purpose. 50%+ today is on track. The log of causes matters more than the number.

---

## 9. Self-check: answer before you activate killer.sh

1. You ssh to a task's host and run `k run tmp --image=nginx $do > pod.yaml`. What could go wrong, and how do you guard against it?
2. You're 8 minutes into a 4% NetworkPolicy task and it still isn't working. What exactly do you do in the next 60 seconds?
3. A task says "update the Deployment `api` in namespace `prod` to use image `api:2.1`". Why is `k delete deploy api` plus re-creating it a bad idea, and what's the one-line answer?
4. Name the five root-cause categories you'll use for the miss log, and the habit that fixes each of the last three.
5. A pod is `Running` with `READY 0/1` and 0 restarts. The Service in front of it returns nothing. What's the most likely cause, and which two commands confirm it?
6. A pod restarts every 20 seconds, but `k logs --previous` shows nginx starting cleanly with no errors. Where do you look, and what are you expecting to see?
7. killer.sh session 2 has the same questions as session 1. How should that change what you do tonight and what you do on Day 14?
8. The exam's base machine has no `kubectl`. Where do you run `kubectl`, and how do you make sure you're in the right place before an edit?

<details>
<summary>Answers</summary>

1. Each ssh session is a fresh shell, so `$do` is probably empty on this host. The command then **creates** the pod `tmp` and writes `pod/tmp created` into `pod.yaml`. You get a stray pod and a useless file. Guard: on each host where you generate YAML, run `export do="--dry-run=client -o yaml"` first (or `echo $do` to check), or type the flags. If it already happened, `k delete pod tmp` straight away.
2. Note where you got to (for example "policy applied, egress DNS missing?") on scratch paper. If the half-done policy might break something else, delete it or leave it in its least harmful state. Flag the task, `exit` to `base`, and start the next task. Come back in Pass 2/3.
3. Recreating changes the object's UID, resets rollout history, and can drop labels, annotations, replicas or other fields the grader checks. It can also briefly leave the app with zero pods. The task says update, so: `k set image deploy/api -n prod <container>=api:2.1` (get the container name with `k get deploy api -n prod -o jsonpath='{.spec.template.spec.containers[*].name}'`), then `k rollout status deploy/api -n prod`.
4. Knowledge gap, slow, misread, wrong host/context, didn't verify. Misread → copy-paste names from the task and re-read it before `exit`. Wrong host/context → `hostname` (and `k config current-context` if the task gives one) before every edit. Didn't verify → one proof command per task (`k get`, `auth can-i`, curl from a test pod, `Bound`).
5. A failing **readiness** probe (wrong port or path). The pod isn't added to the Service's endpoints, but nothing restarts. Confirm with `k describe pod <p>` (events: `Readiness probe failed: …`) and `k get endpointslices -l kubernetes.io/service-name=<svc>` (no ready endpoints). The other common cause, a selector mismatch, shows the same empty endpoints. Compare `--show-labels` with the Service selector.
6. `k describe pod` events. You expect `Liveness probe failed: HTTP probe failed with statuscode: 404` (wrong path) or `connection refused` (wrong port), followed by `Container nginx failed liveness probe, will be restarted`. The app is fine. The probe is wrong. Fix the probe's path or port, not the app.
7. Tonight: don't memorise answers. Log the **cause** of each miss and fix that (drill the slow commands, practise the habit). Redo questions in the same environment while it's open if it helps you learn the *method*. Day 14: treat session 2 as a speed and accuracy run. The target is every task correct and verified in under ~90 minutes, using the three-pass plan, and a score that says nothing about knowledge you don't have.
8. On the task's designated host, after `ssh <host>` from the infobox. Each host has `kubectl`, the `k` alias and completion. Before any edit, run `hostname` (and `k config current-context` if the task names a context). After the task, `exit` and check the prompt says `base` again.

</details>

---

**Done when:** every [Day 12 acceptance criterion](../stories/day-12-mock-1.md#acceptance-criteria) passes: a full 2-hour run without stopping, every miss and slow question logged with one root cause, and a score you can judge against the ~50% bar. Also finish the four app drills in under 4 minutes each. Tomorrow (Day 13) uses your miss log directly: you redo every miss from memory, drill your two weakest domains, and re-time etcd restore and the kubeadm upgrade.
