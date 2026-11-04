# Day 6 Lesson: Deployments, config and resources

**Story:** [day-06-workloads](../stories/day-06-workloads.md) · **Read time:** ~35 min · **Then:** 2 hours in the lab

**Why this matters:** Workloads & Scheduling is **15%** of the exam, and today's material shows up in other domains too. A Troubleshooting task (30%) is often "the Deployment isn't rolling out" or "pods aren't being created in namespace X", and the cause is a bad image, a missing ConfigMap key or a ResourceQuota. None of these tasks are hard. They're **speed** tasks: a rollback, a ConfigMap mounted as a volume, a quota or an HPA should each take 2–4 minutes. If you write YAML from scratch, they take 10. The goal today is to get every one of them down to a `k create ... $do`, one short edit in vim, and a check.

On EKS you've seen all of these objects, usually rendered by Helm or Argo. The difference today is driving them by hand and knowing **which controller rejected what**: a quota error lands on the **ReplicaSet**, not the pod, and that one fact saves a lot of time on troubleshooting tasks.

> **Version note (checked today).** The exam is on **Kubernetes v1.37** (see Day 2). Your lab is on **v1.35** after Day 3. Three things changed recently and matter today:
> - **`kubectl autoscale --cpu`**. Since kubectl v1.34 the man page examples use `--cpu=50%` (utilization) or `--cpu=500m` (average value), plus a new `--memory=70%` / `--memory=200Mi`. The old `--cpu-percent=50` still works in v1.37. Use whichever your `k autoscale --help` shows.
> - **In-place pod resize is GA in v1.35** ([blog](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga), [task page](https://www.kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)). A pod's CPU/memory can now be changed without recreating it, through the `resize` subresource. Section 4 covers the short version.
> - `--record` on `kubectl` commands is deprecated. To fill the `CHANGE-CAUSE` column, set the annotation yourself (section 2).

---

## 1. The objects and who owns them

```
 Deployment web  ──(deployment controller, in kube-controller-manager)──┐
   spec.template (pod template) ── hash ──► pod-template-hash label     │
                                                                        ▼
   ReplicaSet web-7d9f8c6b5   (revision 2, current, replicas 3)  ◄── scales up
   ReplicaSet web-5c4b8d7f9   (revision 1, old,     replicas 0)  ◄── scales down, KEPT for undo
         │ (replicaset controller)
         ▼
   Pods web-7d9f8c6b5-xxxxx ×3 ──► scheduler ──► kubelet on worker1 ──► containerd
        ▲
        │ admission at CREATE time (in kube-apiserver):
        │   LimitRanger      → fills default requests/limits, enforces min/max
        │   ResourceQuota    → rejects if the namespace's sum would exceed hard limits
        │   (rejection is an event on the ReplicaSet: "FailedCreate ... exceeded quota")
```

| Object | Controller | Key facts for the exam |
|---|---|---|
| Deployment | deployment controller | Every change to `spec.template` = new ReplicaSet = new **revision**. Scaling does **not** create a revision. |
| ReplicaSet | replicaset controller | Old RSs are kept at 0 replicas, up to `revisionHistoryLimit` (default 10). That's what `rollout undo` uses. |
| ConfigMap / Secret | none (data only) | The **kubelet** reads them when it starts a pod. Mounted volumes refresh (~1 min). Env vars **never** refresh. |
| LimitRange | admission plugin | Mutates and validates **new** pods only. Existing pods are untouched. |
| ResourceQuota | admission + quota controller | Counts usage per namespace. If it limits `requests.cpu`, every new pod **must** set cpu requests (or get them from a LimitRange). |
| HPA | HPA controller (every 15 s) | Reads `metrics.k8s.io` (metrics-server). Utilization = usage ÷ **request**. No requests → `<unknown>`. |
| Job / CronJob | job / cronjob controllers | CronJob creates a Job on schedule, and the Job creates pods. |

---

## 2. Deployments and rollouts

### Fastest route

```bash
k create deploy web --image=nginx:1.27 --replicas=3 --port=80 $do > web.yaml   # edit, then apply
k create deploy web --image=nginx:1.27 --replicas=3                              # or straight in

k scale deploy web --replicas=5
k set image deploy/web nginx=nginx:1.28        # <container-name>=<image>. `k create deploy` names the container after the image (nginx)
k annotate deploy web kubernetes.io/change-cause="bump to 1.28" --overwrite   # fills CHANGE-CAUSE
k rollout status deploy/web --timeout=60s      # blocks until done; non-zero exit if it times out
k rollout history deploy/web
k rollout history deploy/web --revision=2      # shows that revision's pod template (image!)
k rollout undo deploy/web                      # to previous revision
k rollout undo deploy/web --to-revision=1      # to a named revision
k rollout pause deploy/web                     # batch several changes ...
k set image deploy/web nginx=nginx:1.28 ; k set resources deploy web -c nginx --requests=cpu=100m,memory=64Mi
k rollout resume deploy/web                    # ... into ONE new revision
k rollout restart deploy/web                   # new revision with the same spec (adds a restartedAt annotation)
```

**Revisions renumber on undo.** If revisions are 1 (1.27), 2 (1.28), 3 (bad), `undo --to-revision=1` makes the 1.27 template the new **revision 4**, and revision 1 disappears from history (a template appears only once). Don't go looking for "revision 1" afterwards.

### Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600     # after this, Progressing=False reason ProgressDeadlineExceeded (no auto-rollback!)
  minReadySeconds: 0
  strategy:
    type: RollingUpdate            # or Recreate: kill all old pods, then start new (downtime, no surge)
    rollingUpdate:
      maxSurge: 1                  # extra pods above replicas during the rollout (number or %; default 25%)
      maxUnavailable: 0            # pods below replicas allowed (default 25%). 0 + surge 1 = one-at-a-time, zero downtime
  selector:
    matchLabels: {app: web}        # immutable after creation
  template:
    metadata:
      labels: {app: web}           # must match the selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports: [{containerPort: 80}]
```

Quick strategy edit without vim:

```bash
k patch deploy web -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
k patch deploy web -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'   # Recreate rejects a rollingUpdate block
```

**How a bad rollout looks:** with `maxUnavailable: 25%` on 4 replicas, the controller makes 1–2 new pods that go `ErrImagePull` → `ImagePullBackOff`, and the old pods stay up. The Deployment never "fails" by itself. It just stops progressing. `k rollout status` hangs, `k get rs` shows two RSs with pods, and after 10 min the condition becomes `ProgressDeadlineExceeded`. **Kubernetes does not roll back for you.** You do.

---

## 3. ConfigMaps and Secrets

### Create

```bash
k create cm app-cfg --from-literal=MODE=prod --from-literal=LOG_LEVEL=debug
k create cm app-files --from-file=app.properties              # key = file name
k create cm app-files --from-file=config.ini=./my-app.ini      # key = config.ini
k create cm app-env --from-env-file=app.env                    # each KEY=VAL line becomes a key
k create secret generic db --from-literal=username=app --from-literal=password='S3cr3t!'
k create secret generic tls-files --from-file=./tls.crt --from-file=./tls.key
k create secret tls web-tls --cert=tls.crt --key=tls.key       # type kubernetes.io/tls, keys tls.crt / tls.key
k create secret docker-registry regcred --docker-server=r.io --docker-username=u --docker-password=p

k get secret db -o jsonpath='{.data.password}' | base64 -d; echo   # decode (single quotes in the literal above stop bash expanding the !)
```

`data:` values are base64. `stringData:` takes plain text and the API server encodes it. Use `stringData` when you write a Secret by hand. Base64 is **encoding, not encryption**. Encryption at rest is an API server flag (`--encryption-provider-config`), not a Secret field.

### Consume: all four ways in one pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: APP_MODE                 # 1) one key, renamed
      valueFrom:
        configMapKeyRef:
          name: app-cfg
          key: MODE
    - name: DB_USER                  # 2) one Secret key
      valueFrom:
        secretKeyRef:
          name: db
          key: username
          optional: false            # default: pod won't start (CreateContainerConfigError) if missing
    envFrom:                         # 3) all keys as env vars
    - configMapRef:
        name: app-cfg
      prefix: CFG_                   # optional: CFG_MODE, CFG_LOG_LEVEL
    - secretRef:
        name: db
    volumeMounts:
    - name: db-secret
      mountPath: /etc/db             # 4) files: /etc/db/username, /etc/db/password
      readOnly: true
    - name: cfg
      mountPath: /etc/app/app.properties
      subPath: app.properties        # one file into an existing dir. subPath mounts NEVER refresh
  volumes:
  - name: db-secret
    secret:
      secretName: db                 # note: secretName, not name
      defaultMode: 0400
  - name: cfg
    configMap:
      name: app-files                # note: name
      items:                         # optional: only these keys, and choose the file name
      - key: app.properties
        path: app.properties
```

Imperative shortcuts on a Deployment (no vim):

```bash
k set env deploy/web --from=configmap/app-cfg                   # envFrom-like: copies every key as an env var
k set env deploy/web --from=configmap/app-cfg --keys=MODE       # one key
k set env deploy/web --from=secret/db --prefix=DB_
k set env deploy/web --list                                     # show
# there is no `k set volume`: volumes are always YAML (k edit, or k get -o yaml > f.yaml and apply)
```

**Update behaviour:**

| Consumed as | Updates when the CM/Secret changes? |
|---|---|
| env / envFrom | **No.** Restart: `k rollout restart deploy/web` |
| volume (whole dir) | Yes, within ~1 min (kubelet sync + cache TTL), through an atomic `..data` symlink swap |
| volume with `subPath` | **No** |
| `immutable: true` CM/Secret | Can't change at all. Delete and recreate |

---

## 4. Requests, limits and QoS

- **Request** = what the **scheduler** reserves on a node. A pod fits only if the node's allocatable minus the sum of existing **requests** is at least the pod's request. Actual usage doesn't matter to the scheduler.
- **Limit** = what the **kubelet/kernel** enforces. Above the CPU limit, the container is **throttled** (CFS quota). Above the memory limit, it's **OOMKilled** (exit 137, `Last State: Terminated, Reason: OOMKilled`).
- Only a limit set → request defaults to the limit.

```bash
k set resources deploy web -c nginx --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi
k run qos --image=nginx --restart=Never $do > qos.yaml     # then add a resources block in vim
```

```yaml
    resources:
      requests: {cpu: 100m, memory: 128Mi}   # 100m = 0.1 core. Mi = 2^20 bytes (M = 10^6: 128M ≠ 128Mi)
      limits:   {cpu: 200m, memory: 256Mi}
```

### QoS classes (acceptance criterion 3: say this without notes)

| Class | Rule (checked across **every** container in the pod) | Eviction order under node pressure | `oom_score_adj` |
|---|---|---|---|
| **Guaranteed** | Every container has cpu **and** memory limits, and requests = limits (or requests unset, so they default to limits) | Last | -997 |
| **Burstable** | Not Guaranteed, but at least one container has a cpu or memory request or limit | Middle (by how far usage is above request) | 2–999 |
| **BestEffort** | No container has any request or limit | First | 1000 |

```bash
k get pod qos -o jsonpath='{.status.qosClass}{"\n"}'
k describe pod qos | grep -i qos
```

One sidecar without resources next to a fully specified main container makes the pod **Burstable**, not Guaranteed. QoS is assigned at creation and can't change.

### In-place resize (GA in v1.35): short version

```bash
k patch pod qos --subresource resize -p \
  '{"spec":{"containers":[{"name":"qos","resources":{"requests":{"cpu":"150m"},"limits":{"cpu":"200m"}}}]}}'
k get pod qos -o jsonpath='{.status.containerStatuses[0].resources}{"\n"}'   # what's actually applied
```

The container's `resizePolicy` (`restartPolicy: NotRequired` by default, or `RestartContainer`) decides whether a restart is needed, separately for cpu and memory. A resize that would change the QoS class is rejected. If the node can't fit it, the pod shows the condition `PodResizePending` (reason `Deferred` or `Infeasible`). For a **Deployment**, change the template instead (`k set resources`). That's a normal rollout and is what most exam tasks expect.

---

## 5. LimitRange and ResourceQuota

```bash
k create ns team-a
k create quota team-a-q -n team-a --hard=pods=4,requests.cpu=1,requests.memory=1Gi,limits.cpu=2,limits.memory=2Gi
k create quota objs -n team-a --hard=configmaps=5,secrets=5,services=3,count/deployments.apps=2
# LimitRange has no `k create` command: YAML only
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: team-a
spec:
  limits:
  - type: Container
    defaultRequest: {cpu: 100m, memory: 128Mi}   # injected as requests if the container sets none
    default:        {cpu: 200m, memory: 256Mi}   # injected as LIMITS (the field name is "default")
    min:            {cpu: 50m,  memory: 64Mi}
    max:            {cpu: "1",  memory: 1Gi}
  # - type: Pod             max: sum over containers
  # - type: PersistentVolumeClaim   min/max storage
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-q
  namespace: team-a
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

```bash
k describe quota -n team-a          # Used vs Hard
k describe limitrange -n team-a
```

**How rejection looks:**

| Situation | Where you see the error |
|---|---|
| `k run` / `k apply` a bare pod over quota | Right in your terminal: `forbidden: exceeded quota: team-a-q, requested: requests.cpu=500m, used: ..., limited: ...` |
| Quota limits `requests.cpu`, pod sets none, and there's no LimitRange | `forbidden: failed quota: team-a-q: must specify requests.cpu` |
| Pod above LimitRange `max` | `forbidden: maximum cpu usage per Container is 1, but limit is 2` |
| **Deployment** over quota | `k apply` **succeeds**. The Deployment shows 2/5 READY. **`k describe rs <new-rs>`** shows `FailedCreate ... exceeded quota`. The pod doesn't exist, so `k describe pod` has nothing. |

The last row is the troubleshooting trap: "deployment has fewer pods than replicas, no pods Pending" → look at the ReplicaSet events (or `k get events -n team-a --sort-by=.lastTimestamp`).

---

## 6. HPA and metrics-server

```bash
# metrics-server (the exam cluster usually has it; your lab doesn't)
k apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
k -n kube-system patch deploy metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
# kubeadm kubelets serve self-signed certs, so without that flag: "x509: cannot validate certificate"
k -n kube-system rollout status deploy/metrics-server
k top nodes ; k top pods -A --sort-by=cpu ; k top pod web-xxx --containers

k autoscale deploy web --min=2 --max=10 --cpu=50%          # kubectl ≥1.34. Older: --cpu-percent=50
k autoscale deploy web --min=2 --max=10 --cpu=50% $do > hpa.yaml   # to add memory or behavior
k get hpa web -w                                            # TARGETS cpu: 12%/50%
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization          # % of REQUEST. Or type: AverageValue + averageValue: 200m
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target: {type: Utilization, averageUtilization: 70}
  behavior:                        # optional; exam tasks sometimes ask for it
    scaleDown:
      stabilizationWindowSeconds: 300   # default 300 for scale-down, 0 for scale-up
```

Formula: `desired = ceil(current × currentMetric / target)`. Four pods at 100% with a 50% target → 8. `TARGETS <unknown>/50%` means one of three things: metrics-server is missing or broken (`k top pods` fails), the pods have **no cpu request**, or the pods are only just starting.

---

## 7. Jobs and CronJobs

```bash
k create job pi --image=perl:5.40 $do -- perl -Mbignum=bpi -wle 'print bpi(200)' > job.yaml
k create cronjob tick --image=busybox:1.36 --schedule="*/1 * * * *" $do -- sh -c 'date; echo hi' > cj.yaml
k create job tick-now --from=cronjob/tick          # run a CronJob once, right now (great for testing)
k get jobs,pods -l job-name=pi                     # pods carry job-name=<job>
k logs job/pi
k wait --for=condition=complete job/pi --timeout=120s
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch
spec:
  completions: 6                 # total successful pods needed (default 1)
  parallelism: 2                 # at most 2 at a time (default 1)
  backoffLimit: 4                # failed pod retries before the Job is Failed (default 6)
  activeDeadlineSeconds: 120     # hard wall-clock limit for the whole Job (beats backoffLimit)
  ttlSecondsAfterFinished: 300   # auto-delete the Job and its pods 5 min after it finishes
  # completionMode: Indexed      # pods get JOB_COMPLETION_INDEX 0..5
  template:
    spec:
      restartPolicy: Never       # Never or OnFailure. Always is rejected for Jobs
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh", "-c", "echo working; sleep 5"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: tick
spec:
  schedule: "*/5 * * * *"        # min hour day-of-month month day-of-week
  timeZone: "Asia/Kolkata"       # optional (GA since 1.27); default is kube-controller-manager's time zone (UTC on kubeadm)
  concurrencyPolicy: Forbid      # Allow (default) | Forbid (skip if still running) | Replace
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 3  # default 3
  failedJobsHistoryLimit: 1      # default 1
  suspend: false
  jobTemplate:                   # a whole Job spec goes in here, so there are two "spec:" levels
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: tick
            image: busybox:1.36
            command: ["sh", "-c", "date"]
```

`restartPolicy: OnFailure` restarts the container **in the same pod** (backoff counts restarts). `Never` makes a **new pod** for each failure, so failed pods pile up where you can read their logs. Pick `Never` when you need to debug.

---

## 8. Hands-on walkthrough on cp1 + worker1

From cp1. `k config current-context` first. Work in a namespace: `k create ns d6 && k config set-context --current --namespace=d6`.

**Step 1: rollout and rollback (task 1, acceptance criterion 1).** ⏱ Target < 3 min from the bad image to rolled back.

```bash
k create deploy web --image=nginx:1.27 --replicas=4
k annotate deploy web kubernetes.io/change-cause="1.27 initial"
k set image deploy/web nginx=nginx:1.28 && k annotate deploy web kubernetes.io/change-cause="1.28" --overwrite
k rollout status deploy/web
# --- start the timer ---
k set image deploy/web nginx=nginx:1.99-doesnotexist && k annotate deploy web kubernetes.io/change-cause="bad" --overwrite
k rollout status deploy/web --timeout=20s          # times out: that's how you "notice"
k get rs ; k get pods | grep -v Running            # ImagePullBackOff on the new RS's pods
k rollout history deploy/web                       # 1 "1.27 initial", 2 "1.28", 3 "bad"
k rollout history deploy/web --revision=1 | grep Image
k rollout undo deploy/web --to-revision=1
k rollout status deploy/web && k get deploy web -o jsonpath='{..image}{"\n"}'   # nginx:1.27
# --- stop the timer ---
k rollout history deploy/web                       # 2, 3, 4 (1 was renumbered to 4)
```

Then the strategy: patch `maxSurge: 1, maxUnavailable: 0`, run `k set image deploy/web nginx=nginx:1.28` and watch `k get rs -w` in a second terminal (tmux). Old RS steps down one at a time. Do the same with `Recreate` and see all pods terminate first. Finally try `pause` → two changes → `resume` and confirm `rollout history` grew by **one**.

**Step 2: ConfigMap and Secret (task 2, acceptance criterion 2).** ⏱ Target < 5 min.

```bash
k create cm app-cfg --from-literal=MODE=prod --from-literal=LOG_LEVEL=debug
k create secret generic db --from-literal=username=app --from-literal=password=S3cr3t
k run cfg --image=busybox:1.36 $do -- sh -c 'sleep 3600' > cfg.yaml
vim cfg.yaml          # add env APP_MODE from app-cfg/MODE, and the db Secret as a volume at /etc/db (section 3)
k apply -f cfg.yaml
k exec cfg -- sh -c 'echo $APP_MODE; ls /etc/db; cat /etc/db/password; echo'   # prod / password username / S3cr3t
```

Then see the update behaviour yourself: `k edit cm app-cfg` (MODE=staging) and `k create secret generic db --from-literal=username=app --from-literal=password=N3w $do | k apply -f -`. After ~60 s `/etc/db/password` shows `N3w`, but `$APP_MODE` is still `prod`. Finally reference a key that doesn't exist (`key: NOPE`) in a new pod and read the `CreateContainerConfigError` in `k describe pod`.

**Step 3: requests, limits, QoS (task 3, acceptance criterion 3).**

```bash
k run be  --image=nginx                                                          # BestEffort
k run bu  --image=nginx $do > bu.yaml   # add requests cpu 100m/mem 64Mi only    → Burstable
k run gu  --image=nginx $do > gu.yaml   # add limits cpu 200m/mem 128Mi only     → Guaranteed (requests copied)
k apply -f bu.yaml -f gu.yaml
k get pod be bu gu -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
k get pod gu -o jsonpath='{.spec.containers[0].resources}{"\n"}'                # requests filled in = limits
```

Then OOMKill on purpose: run `k run oom --image=polinux/stress --restart=Never $do -- stress --vm 1 --vm-bytes 200M --vm-hang 1`, add `limits: {memory: 100Mi}`, apply, and run `k describe pod oom | grep -A3 'Last State\|State:'` → `OOMKilled`, exit code 137. Then resize `gu` in place (section 4) and check that `.status.containerStatuses[0].restartCount` stayed 0.

Say the QoS table out loud, without looking. That's the criterion.

**Step 4: LimitRange and ResourceQuota (task 4).**

```bash
k create ns team-a
k create quota team-a-q -n team-a --hard=pods=3,requests.cpu=500m,requests.memory=512Mi
k run nores -n team-a --image=nginx                   # REJECTED: must specify requests.cpu,requests.memory
k apply -f limitrange.yaml                            # the LimitRange from section 5, in team-a
k run nores -n team-a --image=nginx                   # now admitted
k get pod nores -n team-a -o jsonpath='{.spec.containers[0].resources}{"\n"}'   # defaults injected
k create deploy big -n team-a --image=nginx --replicas=5
k get deploy big -n team-a                            # 2/5 (pods quota 3, nores uses 1)
k describe rs -n team-a -l app=big | grep -A2 FailedCreate    # exceeded quota: team-a-q, requested: pods=1 ...
k describe quota -n team-a
```

**Step 5: HPA (task 5).** Install metrics-server as in section 6. Wait about 60 s, then:

```bash
k top nodes
k create deploy php --image=registry.k8s.io/hpa-example --port=80
k set resources deploy php --requests=cpu=200m --limits=cpu=500m
k expose deploy php --port=80
k autoscale deploy php --min=1 --max=5 --cpu=50%
k run load --image=busybox:1.36 --restart=Never -- sh -c 'while true; do wget -q -O- http://php; done'
k get hpa php -w                                      # climbs past 50%, REPLICAS grows within ~1 min
k delete pod load                                     # scale-down waits 5 min (stabilization window)
```

Break it: create a second Deployment with no requests (`k create deploy nreq --image=nginx`), run `k autoscale deploy nreq --max=3 --cpu=50%`, and see `<unknown>/50%` in `k get hpa` plus the `FailedGetResourceMetric` event ("missing request for cpu") in `k describe hpa nreq`.

**Step 6: Jobs and CronJobs (task 6).**

```bash
k apply -f job.yaml                 # the completions:6 / parallelism:2 Job from section 7
k get pods -l job-name=batch -w     # 2 at a time, 6 total
k create job fail --image=busybox:1.36 $do -- sh -c 'exit 1' > fail.yaml   # set backoffLimit: 2, restartPolicy: Never
k apply -f fail.yaml ; k get pods -l job-name=fail -w                      # 3 failed pods, then Job Failed
k describe job fail | grep -i -A2 'BackoffLimitExceeded\|Conditions'
k create cronjob tick --image=busybox:1.36 --schedule="*/1 * * * *" -- sh -c date
k create job tick-now --from=cronjob/tick && k logs job/tick-now
k get cj,jobs -w                    # a new Job each minute; only 3 successful kept
```

**Cleanup:** `k delete ns d6 team-a` and `k config set-context --current --namespace=default`. Keep metrics-server installed; Day 11 uses `k top`.

---

## 9. Traps that cost points

- **`k set image deploy/web web=...` when the container is called `nginx`.** You get `unable to find container named "web"`. `k create deploy` names the container after the image's base name. Check with `k get deploy web -o jsonpath='{..containers[*].name}'`.
- **Thinking scaling creates a revision.** It doesn't. Only template changes do.
- **Expecting an automatic rollback** after `progressDeadlineSeconds`. Kubernetes only sets a condition.
- **Rolling back to a revision number you read before an earlier undo.** Revisions renumber. Re-run `rollout history` right before `undo --to-revision`.
- **Changing `spec.selector`** of a Deployment. It's immutable. Delete and recreate (`k replace --force -f`).
- **`configMap.name` vs `secret.secretName`** in volumes. Mixing them up gives a validation error, or a pod stuck in `ContainerCreating` with `FailedMount`.
- **Expecting env vars to update** after editing a ConfigMap. They don't, so restart the rollout. `subPath` mounts don't update either.
- **Forgetting `-n`** when creating the ConfigMap/Secret. The pod in another namespace can't see it: `CreateContainerConfigError: configmap "x" not found`.
- **`128M` vs `128Mi`**, and `cpu: 1` vs `cpu: 1m` (one core vs one-thousandth).
- **Guaranteed needs cpu AND memory**, in **every** container. A sidecar without limits makes the pod Burstable.
- **Quota on `requests.cpu` without a LimitRange**: every pod without requests is rejected. Read the error, then add a LimitRange or set resources.
- **Debugging a short Deployment with `k describe pod`.** Rejected pods don't exist. Use `k describe rs` or `k get events`.
- **LimitRange "isn't working"** on pods that already exist. It only applies at admission. Recreate them (`k rollout restart`).
- **HPA `<unknown>`**: no metrics-server, or no cpu **request** on the target pods.
- **`restartPolicy: Always` in a Job**: rejected. Use `Never` or `OnFailure`.
- **CronJob YAML nesting**: `spec.jobTemplate.spec.template.spec.containers`. Generate it with `k create cronjob ... $do`. Don't type the nesting by hand.
- **Cron schedule in UTC.** "Run at 09:00 IST" is `30 3 * * *` unless you set `timeZone`.

---

## 10. Self-check: answer before you open the lab

1. You run `k scale deploy web --replicas=6`, then `k set image deploy/web nginx=nginx:1.28`. How many new revisions, and why?
2. A rollout of a bad image has been running for 15 minutes. What state is the Deployment in, are old pods still serving, and what exactly do you run to get back to the image from revision 2?
3. A pod has two containers. Container A: requests = limits = 500m CPU / 256Mi. Container B: no resources. What is the pod's QoS class? What single change makes it Guaranteed?
4. You edited a ConfigMap. The pod mounts it as a volume at `/etc/app` and also reads key `MODE` through `valueFrom`. What changes in the running pod, and when?
5. A namespace has a ResourceQuota with `requests.memory: 1Gi` and no LimitRange. `k create deploy api --image=nginx --replicas=2` returns "created", but `k get pods` shows nothing. Where do you find the reason, and what does it say?
6. `k get hpa` shows `TARGETS <unknown>/60%`. Give two possible causes and the command that tells them apart.
7. Write the kubectl command that creates an HPA for deployment `api`: 2–8 replicas, target 60% CPU.
8. A Job must run 10 pods in total, 3 at a time, give up after 3 failed pods, and delete itself 10 minutes after finishing. Which four fields?

<details>
<summary>Answers</summary>

1. One. Scaling changes `spec.replicas`, which isn't part of the pod template, so it creates no ReplicaSet and no revision. `set image` changes the template, so it creates a new ReplicaSet and a new revision.
2. It's stuck mid-rollout: the new RS has a few pods in `ImagePullBackOff`, and the old RS still has `replicas − maxUnavailable` pods running and serving. After `progressDeadlineSeconds` (600 s) the `Progressing` condition is `False` with reason `ProgressDeadlineExceeded`, but nothing rolls back automatically. Run `k rollout history deploy/web` (and `--revision=2` to confirm the image), then `k rollout undo deploy/web --to-revision=2` and `k rollout status deploy/web`.
3. **Burstable**: at least one container has resources, but not every container has cpu and memory limits equal to requests. Give container B cpu and memory limits (requests default to them, or set them equal).
4. The volume files update within about a minute (kubelet sync plus cache, through the `..data` symlink swap), as long as the mount isn't a `subPath`. The env var `MODE` never changes in the running container. It needs a new pod: `k rollout restart deploy/<name>`.
5. In the ReplicaSet's events: `k describe rs -l app=api` (or `k get events --sort-by=.lastTimestamp`). It shows `FailedCreate ... forbidden: failed quota: <name>: must specify requests.memory`. The Deployment was created, but the ReplicaSet's pod creates were rejected at admission. Fix: set memory requests, or add a LimitRange with `defaultRequest`.
6. (a) metrics-server is missing or broken, (b) the target pods have no CPU request, so utilization can't be computed (also possible: the pods only just started). `k top pods` fails in case (a) and works in case (b). `k describe hpa` shows `FailedGetResourceMetric` with the specific message.
7. `k autoscale deploy api --min=2 --max=8 --cpu=60%` (on an older kubectl: `--cpu-percent=60`).
8. `completions: 10`, `parallelism: 3`, `backoffLimit: 3`, `ttlSecondsAfterFinished: 600`. Plus `restartPolicy: Never` (or `OnFailure`) in the pod template, which is required.

</details>

---

**Done when:** every [Day 6 acceptance criterion](../stories/day-06-workloads.md#acceptance-criteria) passes: bad image rolled out, noticed and rolled back to a named revision in < 3 min; a pod with one ConfigMap key as env and a Secret as a volume, checked with `exec`, in < 5 min; the Guaranteed / Burstable / BestEffort rules explained without notes. Tomorrow (Day 7, scheduling) builds on today's **requests**: the scheduler places pods by requests, so "Pending: Insufficient cpu" is today's numbers meeting tomorrow's taints, affinity and priority. Today's single-container pods also become init containers and sidecars.
