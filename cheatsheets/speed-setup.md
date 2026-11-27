# Speed setup

Roughly 15–20 tasks in 120 minutes, about 6–7 minutes each.

> **Current exam format.** Each task runs on its own host. You `ssh <host>` from the task's infobox, and you `exit` back to `base` after the task. Every task host already has `kubectl`, the `k` alias and bash completion. `base` has none of them. So you **don't** set aliases once at the start. Set only what you need, on the host where you need it (see [Day 12, section 1](../notes/day-12-mock-1-strategy-and-pacing.md#1-what-changes-now-that-each-task-has-its-own-host)).

## Per host, only when the task needs it

```bash
export do="--dry-run=client -o yaml"    # k run web --image=nginx $do > web.yaml
export now="--force --grace-period 0"   # k delete pod web $now
echo 'set et ts=2 sw=2' >> ~/.vimrc     # only if you will edit YAML on this host
```

`$do` is empty on a new host. If you forget to export it, `k run x $do > x.yaml` **creates** the pod. Check with `echo $do`.

In your own lab, put these in `~/.bashrc` once, because there is no per-task host:

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period 0"
source <(kubectl completion bash); complete -o default -F __start_kubectl k
```

```vim
" ~/.vimrc
set expandtab tabstop=2 shiftwidth=2
```

## Habits for every task

1. `ssh` to the host in the infobox. Run `hostname`. If the task also gives a context, run its `kubectl config use-context` line. Work on the wrong host scores zero.
2. Generate YAML instead of writing it: `k run`, `k create deploy|svc|cm|secret|role|rolebinding|job|cronjob|ingress`, `k expose`, each with `$do`.
3. Look up field names with `k explain pod.spec.affinity --recursive` before opening the docs.
4. Verify every answer: `k get`, `k describe`, or a test pod with `k run tmp --rm -it --image=busybox -- sh`.
5. Flag anything taking over 8 minutes and move on. Come back at the end.
6. `exit` back to `base` before the next task. Nested ssh is not supported.

## Imperative commands worth having in your fingers

```bash
k run web --image=nginx --port=80 --labels=app=web $do
k create deploy web --image=nginx --replicas=3 $do
k expose deploy web --port=80 --target-port=8080 --type=NodePort $do
k create cm app-cfg --from-literal=MODE=prod --from-file=app.conf $do
k create secret generic db --from-literal=PASS=s3cr3t $do
k create role pod-reader --verb=get,list,watch --resource=pods -n dev
k create rolebinding rb --role=pod-reader --serviceaccount=dev:app -n dev
k create job once --image=busybox -- sh -c 'echo hi'
k create cronjob tick --image=busybox --schedule="*/5 * * * *" -- date
k create ingress web --class=nginx --rule="app.example.com/*=web:80" $do   # trailing * = Prefix
k set image deploy/web nginx=nginx:1.27
k scale deploy web --replicas=5
k rollout undo deploy/web --to-revision=2
k auth can-i list pods --as=system:serviceaccount:dev:app -n dev
```
