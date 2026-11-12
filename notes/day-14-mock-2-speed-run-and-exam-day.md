# Day 14 Lesson: Mock exam 2, the speed run, and exam-day execution

**Story:** [day-14-mock-2](../stories/day-14-mock-2.md) · **Read time:** ~25 min (read sections 1–3 **before** you activate killer.sh session 2) · **Then:** 2 h mock + ~45 min slow-task redo + ~30 min exam-day setup, then **stop**

**Why this matters:** Tomorrow's paper covers every domain: Troubleshooting **30%**, Cluster Architecture **25%**, Services & Networking **20%**, Workloads & Scheduling **15%**, Storage **10%**. Day 12 taught the pacing plan and Day 13 closed your knowledge gaps. Today is different: session 2 has the **same questions** as session 1, so the score tells you almost nothing new. What it can show is whether your **method** is fast and reliable enough to leave 30 minutes spare. Those 30 minutes are your margin on Friday for one bad task, one slow docs search, or one misread. The second half of today is about removing every non-Kubernetes risk from tomorrow: the machine, the room, the ID, the check-in clock.

> **Version and format notes (checked today).**
> - The [LF exam tips page](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad) still says *"The CKA environment is currently running Kubernetes v1.37"*. Format: **15–20 performance-based tasks, 2 hours**, one designated host per task (`ssh <host>`, `exit` back to `base`, `sudo -i` for root, **no nested ssh**). Hosts have `kubectl` with the `k` alias and completion, plus `yq`, `curl`, `wget`, `man`. `base` has none of them. Don't reboot `base`.
> - **Allowed docs** ([Resources Allowed](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed)): kubernetes.io/docs, kubernetes.io/blog, helm.sh/docs, **and, for the CKA only, the Gateway API docs at gateway-api.sigs.k8s.io**, plus any task-specific links in the task's Quick Reference box. The search box on kubernetes.io is fine, but you may not open external results. Earlier lessons listed only the first three. The Gateway API site is the better place for HTTPRoute matches, filters and `parentRefs`.
> - **Pass mark 66%**. Scored automatically, results emailed within 24 hours ([CKA/CKAD/CKS FAQ](https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks)). A retake is included when eligible.
> - **Check-in** ([Taking the exam](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/taking-the-exam)): you can click *Take Exam* up to **30 minutes early**. You **must start no later than 30 minutes after** your slot, or you're a no-show with no refund. Steps: PSI Secure Browser checks hardware and software → agreement → photo of a **government ID with a signature**, name matching your LF profile exactly → selfie → live room scan (wait should be under 15 min) → proctor releases the exam. The timer **does not pause** for disconnects.
> - **Machine** ([Candidate requirements](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/candidate-requirements)): **one** active monitor (1920×1080, 15"+ recommended), a webcam you can move for the room scan, a working mic, wired network if possible, a **personal** device (work-managed laptops with MDM, VPN or endpoint agents often fail), no VM, reboot beforehand, laptop plugged in. On a Mac, the Secure Browser needs camera, microphone, automation and input-monitoring permission.
> - [killer.sh](https://training.linuxfoundation.org/blog/exam-simulators/): 2 sessions, 36 hours each from activation, 20–25 questions, **identical in both sessions**.

---

## 1. What session 2 is for (and what it isn't)

| Session 2 **is** | Session 2 **isn't** |
|---|---|
| A pacing test: can you clear 20–25 harder-than-real tasks in ≤ 90 min? | A knowledge test. You've seen the answers |
| A habit audit: `hostname` before edits, `-n` on every command, one proof per task | A score to feel good or bad about. 90% today is expected |
| The last time you see the exam-style terminal before tomorrow | A reason to stay up late redoing questions |

Because you know the questions, **use the scratch log to time yourself per task**, not just in total. A task you "know" that still takes 9 minutes is a habit or typing problem, and those are exactly what carry over to new questions tomorrow.

Target splits for a 2-hour session 2:

```
 0–2    Skim, mark Q/M/H (as Day 12)
 2–60   Pass 1: every Q and M task. Target ≤ 4 min each
60–85   Pass 2: heavy tasks (etcd, upgrade, broken node/control plane)
85–90   Pass 3: quick re-verify sweep
90–120  Spare. This is the acceptance criterion. Use it to re-verify everything once more,
        then stop the clock and note the time you finished
```

If you finish Pass 1 after the 60-minute mark, the fix isn't "know more". It's one of the speed routes in section 2.

---

## 2. Speed routes: the moves that buy the 30 minutes

These are the moves that separate "correct in 7 min" from "correct in 3 min". Each one replaces a slow habit. Pick the 3–4 that hit your slow rows from Day 12 and use them deliberately in session 2.

| Slow habit | Fast route | Saves |
|---|---|---|
| Writing YAML in vim from scratch | `k create deploy web --image=nginx:1.27 --replicas=3 $do > web.yaml`, edit only the extra fields | 2–3 min |
| Opening docs to find a field name | `k explain pod.spec.containers.livenessProbe --recursive \| less` (or without `--recursive` for one level) | 1–2 min |
| `k edit` on a field you can't change (pod `containers`, `volumes`) and fighting the rejection | Let `k edit` fail. It saves a copy to `/tmp/kubectl-edit-*.yaml`. Then `k replace --force -f /tmp/kubectl-edit-XXXX.yaml` | 2 min |
| Deleting a pod and waiting 30 s | `k delete pod x --force --grace-period=0` (type it; `$now` isn't set on exam hosts) | 30 s per delete |
| Reading `describe` top to bottom | `k describe pod x \| tail -20` (events are at the end), or `k events -n ns --for pod/x` | 1 min |
| Eyeballing a list to answer "which pod uses most CPU" or "sorted by age" | `k top pod -A --sort-by=cpu`, `k get pods -A --sort-by=.metadata.creationTimestamp` | 1–2 min |
| Copying a value out of `-o yaml` by hand | `k get secret db -n app -o jsonpath='{.data.PASS}' \| base64 -d` / `-o jsonpath='{.items[*].metadata.name}'` | 1 min |
| Typing a long YAML into a test pod | `k run t --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T2 svc.ns` | 1 min |
| Scrolling the docs page | kubernetes.io search for the page title (Day 12 §5 list), then Ctrl+F for `kind: NetworkPolicy` | 1–2 min |
| Patching one field with an editor | `k patch deploy web -n app -p '{"spec":{"replicas":3}}'`, `k set resources deploy web --requests=cpu=100m,memory=64Mi`, `k label`, `k annotate`, `k taint` | 1 min |
| Re-checking with `get` after each tiny change | `k get pod -n app -w` in the same terminal while you work, Ctrl+C when done | 30 s |
| Writing an answer file by copying terminal output | Redirect directly: `k get nodes -o name > /opt/answers/q3.txt`, then `cat` it once | 30 s |

Two more habits for tasks that write to node files:

- `sudo -i` **once** at the start of a node task, not `sudo` on every line. When you're done, `exit` twice (root, then ssh) and check the prompt shows `base`.
- Before editing anything under `/etc/kubernetes/manifests/`: `cp <file> /tmp/<file>.bak`. Never leave a backup **in** the manifests directory. The kubelet tries to run every file there (Day 2).

---

## 3. Highest-value tasks: 60-second recall cards

You've drilled the full sequences on Days 12–13. Before you activate session 2, go through these cards out loud. If you can't say the whole card in about a minute, that task is your first redo after the mock.

**etcd backup + restore (often 7–13%).** `grep` cert paths from `etcd.yaml` → `etcdctl snapshot save` with `--endpoints --cacert --cert --key` → `etcdutl snapshot status -w table` → `etcdutl snapshot restore --data-dir /var/lib/etcd-restore` (new dir) → back up the manifest → change **only** the `etcd-data` hostPath → `watch crictl ps --name etcd` → `k get pods -A`. Check `etcdctl version` first: 3.5 can still restore with `etcdctl`, 3.6 needs `etcdutl`.

**kubeadm upgrade (often 10%+).** Repo file `/etc/apt/sources.list.d/kubernetes.list` to the new minor → `apt-get update` → unhold/install/hold kubeadm → `kubeadm upgrade plan` → `kubeadm upgrade apply v1.X.Y` → drain → kubelet + kubectl → `daemon-reload`, restart kubelet → uncordon. Worker: `kubeadm upgrade node`, the rest is the same. Proof: `k get nodes` (kubelet version), no `SchedulingDisabled`.

**Broken node / control plane.** NotReady: `systemctl status kubelet` → `journalctl -u kubelet -n 50` → fix config/flag/cert path → `systemctl enable --now kubelet`. API down: `crictl ps -a --name kube-apiserver` → `crictl logs` or the kubelet journal → fix the manifest (backup first). Proof: a probe that uses the component, then watch restarts for a minute.

**RBAC.** `create sa` → `create role|clusterrole --verb --resource` → `create rolebinding|clusterrolebinding --serviceaccount=ns:name` → `auth can-i ... --as=system:serviceaccount:ns:name` for one `yes` **and** one `no`.

**NetworkPolicy.** `podSelector` = who it protects. `from`/`to` = peers. `namespaceSelector` + `podSelector` in **one** list item = AND, two items = OR. Egress needs DNS (UDP+TCP 53). Proof: `wget -T2` from an allowed and a denied pod.

**Service / Ingress / Gateway.** `k expose` → `k get endpointslices -l kubernetes.io/service-name=<svc>` (empty = selector or `targetPort`) → `k create ingress --class --rule` or Gateway + HTTPRoute (`parentRefs` to the Gateway, `backendRefs` to the Service:port) → curl with a `Host:` header.

**Storage.** PV and PVC with the same `storageClassName` and access mode, PVC size ≤ PV → `claimName` in the pod → `Bound` → write a file through `k exec`. Pending: `k describe pvc` (class, `WaitForFirstConsumer`, size).

**Workloads / scheduling / Helm.** Sidecar = `initContainers` + `restartPolicy: Always`. HPA needs CPU requests. Toleration matches key/value/effect. `helm upgrade --install -n ns --create-namespace --set k=v`, `helm list -A`, `helm rollback`.

---

## 4. Hands-on: session 2, then the slow-task redo

### 4a. killer.sh session 2 (2 h)

Same rules as Day 12: one block of time, a separate timer, **docs only**, no notes or lessons. Activate it on the **machine and network you'll use tomorrow**, so you also test the setup.

On your scratch log, add a column to the Day 12 one:

```
Q# | start | end | min | done/flag | if > 4 min: why (typing / docs / misread / verify)
```

When you finish, note the clock time, then score it.

### 4b. Slow-task redo (≤ 45 min, on cp1 + worker1)

Take every row from 4a that went over 4 minutes (or over 8 for heavy tasks). For each one, rebuild the same **type** of task on your lab and do it twice, using the speed route from section 2 that fits. Don't redo the killer.sh question itself a third time. A different example of the same task type tests the method.

To practise the per-host loop on your own lab (the part that differs most from your EKS day job), run every redo as if cp1 were a task host:

```bash
# from your Mac (playing "base")
ssh cp1                        # or: multipass shell cp1 / limactl shell cp1
hostname                       # cp1
export do="--dry-run=client -o yaml"
# ... do the task, prove it ...
exit                           # back on the Mac, prompt changes
```

Quick lab drills that match the killer.sh task types you're most likely to have been slow on. Time each one:

| Drill | Task | Target |
|---|---|---|
| D1 | In a new namespace `q1`, a Deployment `api` (nginx:1.27, 2 replicas) with CPU request 50m, exposed as ClusterIP `api:80`. Prove with a busybox `wget` | 3 min |
| D2 | Change pod `cfg` (create it first with `k run cfg --image=busybox:1.36 -- sleep 3600`) to mount ConfigMap `app` at `/etc/app`, using `k edit` → fail → `k replace --force -f /tmp/kubectl-edit-*.yaml` | 3 min |
| D3 | Write the names of all pods in `kube-system` sorted by creation time to `/tmp/q3.txt` on cp1, one per line | 1 min |
| D4 | SA `ci` in `q1` that can `get,list,watch` pods and `create` deployments in `q1` only. Show one `yes` and one `no` | 3 min |
| D5 | etcd snapshot to `/opt/snap.db` and its status as a table, saved to `/opt/snap-status.txt` | 3 min |

<details>
<summary>Reference answers for D1–D5</summary>

```bash
# D1
k create ns q1
k create deploy api -n q1 --image=nginx:1.27 --replicas=2
k set resources deploy api -n q1 --requests=cpu=50m
k expose deploy api -n q1 --port=80
k run t -n q1 --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T2 api

# D2
k create cm app --from-literal=MODE=prod
k edit pod cfg        # add volumes + volumeMounts → save → "pods cfg is invalid" → copy saved to /tmp/kubectl-edit-XXXX.yaml
k replace --force -f /tmp/kubectl-edit-XXXX.yaml
k exec cfg -- cat /etc/app/MODE

# D3
k get pods -n kube-system --sort-by=.metadata.creationTimestamp -o custom-columns=NAME:.metadata.name --no-headers > /tmp/q3.txt

# D4
k create sa ci -n q1
k create role ci -n q1 --verb=get,list,watch --resource=pods
k create role ci-deploy -n q1 --verb=create --resource=deployments
k create rolebinding ci -n q1 --role=ci --serviceaccount=q1:ci
k create rolebinding ci-deploy -n q1 --role=ci-deploy --serviceaccount=q1:ci
k auth can-i create deployments -n q1 --as=system:serviceaccount:q1:ci      # yes
k auth can-i create deployments -n default --as=system:serviceaccount:q1:ci # no

# D5 (sudo -i on cp1)
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key snapshot save /opt/snap.db
etcdutl snapshot status /opt/snap.db -w table > /opt/snap-status.txt; cat /opt/snap-status.txt
```

D4 could be one Role with two rules (write it with `$do` and add the second rule in vim). Two Roles is faster to type and scores the same.

</details>

**Cleanup:** `k delete ns q1; k delete pod cfg --force --grace-period=0; k delete cm app`. Leave the lab alone after this. No upgrades, no rebuilds.

### 4c. Exam-day setup (30 min, early evening)

Do this on the **exact** laptop, webcam and network for tomorrow:

1. Run the PSI system check / install the PSI Secure Browser from your LF portal (story AC 2). On a Mac, grant camera, microphone, automation and input monitoring when asked. If it flags software, quit or uninstall it now, not at 30 minutes before the exam.
2. If the laptop is work-managed (MDM, corporate VPN, endpoint agent), switch to a personal machine **today**, or test that the check passes with the VPN off.
3. Check that the name on your LF profile matches your government ID **exactly**, and that the ID has a signature.
4. Room: clear desk, nothing written on walls or whiteboards in view, second monitor unplugged (not just off), phone out of reach. Confirm what else is allowed on the desk in the check-in instructions.
5. Write down your slot in IST and the "click Take Exam" time (30 min earlier). Set two alarms.
6. Walk the [exam-day checklist](../cheatsheets/exam-day.md) and tick it (story AC 3).

### 4d. Tomorrow's timeline (fill in your slot)

```
T-3 h     Light breakfast. No new material. Optionally the section 3 cards once, out loud
T-60 min  Reboot the laptop. Quit everything. Ethernet or sit next to the router. Laptop plugged in
T-40 min  Desk and room as they'll be scanned. ID on the desk
T-30 min  Click "Take Exam". Secure Browser → agreement → ID → selfie → room scan
T-0       Exam released. First 2 min: skim all tasks, mark Q/M/H and the weights
          Then the per-task loop (Day 12 §1) and the three-pass plan
T+105     Stop new work. Re-verify flagged and heavy tasks
T+120     Done. Results by email within 24 h
```

If the check-in or exam environment breaks: use PSI's **live chat** right away. Don't troubleshoot it alone, because the clock runs.

---

## 5. Traps that cost points (today and tomorrow)

- **Celebrating a session 2 score.** You knew the answers. The useful numbers are your finish time and how many tasks took more than 4 minutes.
- **Re-activating your study tonight.** A late session costs more in focus tomorrow than one more drill gains. Stop by early evening (story task 4).
- **Running the PSI check on a different network or laptop.** It proves nothing about tomorrow. Same machine, same room, same Wi-Fi or cable.
- **A work laptop with a corporate VPN or MDM.** The Secure Browser often refuses to start or gets killed mid-exam. Test today or switch.
- **Skipping the docs you're allowed.** The Gateway API site is allowed for the CKA. If an HTTPRoute task needs a header match or a redirect filter, that site has the exact YAML.
- **Using `$now`, `$do` or aliases you set up in your lab.** Exam hosts have only `k` and completion. Type `--force --grace-period=0`, export `do` per host.
- **Leaving the backup inside `/etc/kubernetes/manifests/`.** The kubelet tries to run it as a second static pod.
- **Ending a heavy task at "Running".** Watch for a minute, and check RESTARTS stays flat.
- **Logging in late.** More than 30 minutes after your slot and you're a no-show. Aim to click *Take Exam* at T-30.

---

## 6. Self-check: answer before you activate session 2

1. Session 2 has the same questions as session 1. What two numbers from today actually tell you something, and what's the target for each?
2. You `k edit` a running pod to add a volume and the save is rejected. What's the fastest route from there to a working pod with the volume?
3. Which documentation sites can you open during the CKA, and which one did the earlier lessons leave out?
4. A task asks for "the names of all nodes, one per line, in `/opt/answers/nodes.txt`". Write the command, and say on which machine you run it.
5. You need to restart a static pod after editing its manifest, and you saved `kube-apiserver.yaml.bak` next to the original. What goes wrong?
6. Your slot is 10:00 IST. What's the earliest you can start check-in, and the latest you can start before you're a no-show?
7. What do you do if the exam terminal freezes 40 minutes in?
8. You finish session 2 at 85 minutes, but D2 and the NetworkPolicy task each took over 6 minutes. What do you redo in 4b, and how?

<details>
<summary>Answers</summary>

1. **Finish time** (target ≤ 90 min, so at least 30 min spare, story AC 1) and **the number of tasks over 4 min** (8 for heavy tasks), with their causes. The score is expected to be high and doesn't predict much.
2. Exit the editor. kubectl prints that a copy was saved to `/tmp/kubectl-edit-XXXX.yaml`. Run `k replace --force -f /tmp/kubectl-edit-XXXX.yaml`, which deletes and recreates the pod with your change. Then verify with `k exec ... -- ls <mountPath>`.
3. kubernetes.io/docs, kubernetes.io/blog, helm.sh/docs, **gateway-api.sigs.k8s.io (CKA only)**, and any links in the task's Quick Reference box. Earlier lessons left out the Gateway API site. Search on kubernetes.io is fine, but you can't open external results.
4. On the **task's designated host** (after `ssh <host>`), because that's where the grader looks: `k get nodes -o name > /opt/answers/nodes.txt` gives `node/<name>`. If they want bare names: `k get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' > /opt/answers/nodes.txt` (or `-o custom-columns=N:.metadata.name --no-headers`). `cat` it once to check.
5. The kubelet loads every file in `/etc/kubernetes/manifests/`, so it tries to run the `.bak` too. You get two apiserver definitions fighting over port 6443 and a confusing failure. Keep backups in `/tmp`.
6. Earliest: 09:30 IST (*Take Exam* opens 30 min before). Latest start: 10:30 IST, after which you're a no-show. Aim for 09:30, because check-in plus the room scan can take 15+ minutes.
7. Open PSI's live chat with the proctor straight away. The timer doesn't pause. Don't reboot `base`, which doesn't reset the environment. If you get disconnected, relaunch from the *Access* button or the Secure Browser and repeat check-in.
8. Not the killer.sh questions again. Build a **new** example of each type on your lab: a pod edit that needs `replace --force` (D2 again, then with a different field, e.g. adding a sidecar), and a fresh NetworkPolicy with egress + DNS tested from an allowed and a denied pod. Do each twice with the timer. If the slow part was the docs search, practise jumping to the page by title.

</details>

---

**Done when:** every [Day 14 acceptance criterion](../stories/day-14-mock-2.md#acceptance-criteria) passes: session 2 finished with at least 30 minutes left, the PSI system check passed on tomorrow's exact machine and network, and the exam-day checklist ticked. Then close the laptop. Tomorrow builds on all fourteen days at once: skim, three passes, `hostname` before every edit, one proof per task, `exit` back to `base`. Good luck, Varun.
