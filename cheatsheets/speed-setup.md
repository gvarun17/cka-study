# Speed setup

Roughly 15–20 tasks in 120 minutes, about 6–7 minutes each. Set this up in the first 2 minutes of the exam, and use it daily from Day 1 so it's habit.

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"    # k run web --image=nginx $do > web.yaml
export now="--force --grace-period 0"   # k delete pod web $now
source <(kubectl completion bash); complete -o default -F __start_kubectl k
```

```vim
" ~/.vimrc
set expandtab tabstop=2 shiftwidth=2
```

## Habits for every question

1. Run the `kubectl config use-context ...` line the question gives you, **every time**. Work done in the wrong cluster scores zero.
2. Generate YAML instead of writing it: `k run`, `k create deploy|svc|cm|secret|role|rolebinding|job|cronjob|ingress`, `k expose`, each with `$do`.
3. Look up field names with `k explain pod.spec.affinity --recursive` before opening the docs.
4. Verify every answer: `k get`, `k describe`, or a test pod with `k run tmp --rm -it --image=busybox -- sh`.
5. Flag anything taking over 8 minutes and move on. Come back at the end.
6. After any `ssh`, **`exit`** before the next question.

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
k create ingress web --rule="app.example.com/=web:80" $do
k set image deploy/web nginx=nginx:1.27
k scale deploy web --replicas=5
k rollout undo deploy/web --to-revision=2
k auth can-i list pods --as=system:serviceaccount:dev:app -n dev
```
