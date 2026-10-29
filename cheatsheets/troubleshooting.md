# Troubleshooting playbook

Troubleshooting is 30% of the score. Work from the symptom down one layer at a time, and fix the first broken thing you find before looking further.

| Symptom | Check first | Usual cause |
|---|---|---|
| Node NotReady | SSH in: `systemctl status kubelet`, `journalctl -u kubelet -n 50` | kubelet stopped or disabled, wrong path in `/var/lib/kubelet/config.yaml`, bad cert path, container runtime down |
| `kubectl` hangs or connection refused | On the control plane: `crictl ps -a \| grep apiserver`, `crictl logs <id>`, `/var/log/pods/` | Typo or wrong flag in `kube-apiserver.yaml`, wrong etcd endpoint or cert |
| Pods stay Pending | `k describe pod`, Events section | Taint without toleration, nodeSelector or affinity matches nothing, not enough CPU/memory, unbound PVC, scheduler down |
| Deployment doesn't scale or roll out | `k get pods -n kube-system` | kube-controller-manager broken (check its static pod manifest) |
| CrashLoopBackOff | `k logs <pod> --previous`, `k describe pod` | Bad command or args, missing ConfigMap or Secret, failing liveness probe |
| ImagePullBackOff | `k describe pod` | Image name or tag typo, missing pull secret |
| Service unreachable | `k get endpoints <svc>`, then curl from a test pod | Selector doesn't match pod labels, wrong targetPort, NetworkPolicy blocking |
| DNS fails | `nslookup <svc>.<ns>.svc.cluster.local` from busybox; `k get pods -n kube-system -l k8s-app=kube-dns` | CoreDNS down or misconfigured, NetworkPolicy blocking port 53 |
| PVC Pending | `k describe pvc` | No matching PV (size, accessMode, storageClassName), no default StorageClass, `WaitForFirstConsumer` with no pod yet |

## When kubectl is down

```bash
sudo crictl ps -a                       # which control-plane containers are exiting
sudo crictl logs <container-id>         # why
ls /var/log/pods/                       # logs survive container restarts
sudo journalctl -u kubelet -f           # kubelet complaints about manifests
```

## Rules

- Copy a static pod manifest to `/tmp` before editing it.
- After editing a static pod manifest, the kubelet recreates the pod on its own. Give it 30–60 seconds before deciding the fix failed.
- Use `k get events -A --sort-by=.lastTimestamp` for a fast overview of what's failing.
