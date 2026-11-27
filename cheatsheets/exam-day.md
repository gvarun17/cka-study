# Exam-day checklist

The exam is 2 hours, about 15–20 tasks, 66% to pass, with one free retake. Allowed references: kubernetes.io/docs, kubernetes.io/blog and helm.sh/docs. Confirm the current rules on the [official CKA page](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/).

## Before the day

- [ ] Book the exam slot early (slots fill up)
- [ ] Activate killer.sh (included with registration) for Days 12 and 14
- [ ] Run the PSI system check on the exact laptop, webcam and network you'll use
- [ ] Clear desk, nothing on the walls behind you, government ID ready

## In the exam

- [ ] Check in 30 minutes early
- [ ] First 2 minutes: skim all tasks and mark Q / M / H ([Day 12 pacing](../notes/day-12-mock-1-strategy-and-pacing.md#2-pacing-the-three-pass-plan))
- [ ] Every task: `ssh` to the host in its infobox, then `hostname` (and its `use-context` line if it gives one). `k` and completion are already there
- [ ] On a host where you generate YAML: `export do="--dry-run=client -o yaml"` first ([speed-setup](speed-setup.md))
- [ ] Easy and high-weight tasks first; flag anything over 8 minutes
- [ ] After every task: `exit` back to `base` (nested ssh is not supported)
- [ ] Last 15 minutes: revisit flagged questions, verify answers with `k get`

## Practice resources

- **killer.sh**: harder than the real exam on purpose. 50%+ there is a good sign.
- **[Killercoda CKA scenarios](https://killercoda.com/killer-shell-cka)**: free browser labs.
- **Your own kubeadm cluster**: the only place to break and fix kubelet and static pods freely.
