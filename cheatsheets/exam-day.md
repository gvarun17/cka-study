# Exam-day checklist

The exam is 2 hours, about 15–20 tasks, 66% to pass, with one free retake. Allowed references: kubernetes.io/docs, kubernetes.io/blog and helm.sh/docs. Confirm the current rules on the [official CKA page](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/).

## Before the day

- [ ] Book the slot for Fri Nov 13 (today; slots fill up)
- [ ] Activate killer.sh (included with registration) for Days 12 and 14
- [ ] Run the PSI system check on the exact laptop, webcam and network you'll use
- [ ] Clear desk, nothing on the walls behind you, government ID ready

## In the exam

- [ ] Check in 30 minutes early
- [ ] First 2 minutes: aliases and vimrc from [speed-setup](speed-setup.md)
- [ ] Every question: run its `use-context` command first
- [ ] Easy and high-weight tasks first; flag anything over 8 minutes
- [ ] After any `ssh`: `exit` before the next question
- [ ] Last 15 minutes: revisit flagged questions, verify answers with `k get`

## Practice resources

- **killer.sh**: harder than the real exam on purpose. 50%+ there is a good sign.
- **[Killercoda CKA scenarios](https://killercoda.com/killer-shell-cka)**: free browser labs.
- **Your own kubeadm cluster**: the only place to break and fix kubelet and static pods freely.
