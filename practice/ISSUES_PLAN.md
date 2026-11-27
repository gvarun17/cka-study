# Practice issue plan

The full list of practice issues, written before they were created. One row per issue. Topics follow the lessons. The count per domain follows the exam weights, with Troubleshooting highest. Every topic has 3–5 issues. See [README.md](README.md) for the issue numbers and a suggested order.

## Day 1: How a cluster works, and kubeadm

Lesson: [`notes/day-01-architecture-and-kubeadm.md`](../notes/day-01-architecture-and-kubeadm.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] Map every control-plane file to the kubeadm phase that made it | Cluster Arch | easy |  |
| [Cluster Arch] Remove worker1 and join it again with a new token | Cluster Arch | medium |  |
| [Troubleshooting] kubeadm join fails preflight on worker1 | Troubleshooting | medium | yes |

## Day 2: static pods, certificates, etcd

Lesson: [`notes/day-02-etcd-and-static-pods.md`](../notes/day-02-etcd-and-static-pods.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] Take an etcd snapshot and record its status | Cluster Arch | easy |  |
| [Cluster Arch] Create a static pod on worker1 with a non-default manifest path | Cluster Arch | medium |  |
| [Cluster Arch] Check certificate expiry and renew only the API server certificate | Cluster Arch | medium |  |
| [Cluster Arch] Restore etcd from a snapshot and prove the state went back | Cluster Arch | hard |  |

## Day 3: kubeadm upgrade

Lesson: [`notes/day-03-kubeadm-upgrade.md`](../notes/day-03-kubeadm-upgrade.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] Drain a node that has every kind of drain blocker | Cluster Arch | medium |  |
| [Cluster Arch] Upgrade the control plane by one minor version | Cluster Arch | hard |  |
| [Cluster Arch] Upgrade worker1 to match the control plane | Cluster Arch | hard |  |

## Day 4: RBAC and kubeconfig

Lesson: [`notes/day-04-rbac-and-kubeconfig.md`](../notes/day-04-rbac-and-kubeconfig.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] ServiceAccount that can only read pods and their logs in one namespace | Cluster Arch | easy |  |
| [Troubleshooting] kubectl fails with a broken kubeconfig | Troubleshooting | easy | yes |
| [Cluster Arch] Reuse one ClusterRole in two namespaces, and cluster-wide for nodes | Cluster Arch | medium |  |
| [Cluster Arch] Create user dana with the CSR API and a kubeconfig file for her | Cluster Arch | hard |  |

## Day 5: Helm, Kustomize, CRDs

Lesson: [`notes/day-05-helm-kustomize-crds.md`](../notes/day-05-helm-kustomize-crds.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] Install, upgrade and roll back a Helm release | Cluster Arch | easy |  |
| [Cluster Arch] Build a Kustomize overlay that changes image, replicas and a label | Cluster Arch | medium |  |
| [Cluster Arch] Install a CRD and create a valid custom resource using only kubectl explain | Cluster Arch | medium |  |

## Extra 1: HA control plane

Lesson: [`notes/extra-01-ha-control-plane.md`](../notes/extra-01-ha-control-plane.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] Explain quorum and inspect the etcd members and leaders | Cluster Arch | medium |  |
| [Cluster Arch] Make a fresh control-plane join command after the certificate key expired | Cluster Arch | medium |  |
| [Cluster Arch] Build a 3-node HA control plane in a separate lab | Cluster Arch | hard |  |

## Extra 2: CRI, CNI, CSI

Lesson: [`notes/extra-02-extension-interfaces.md`](../notes/extra-02-extension-interfaces.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Cluster Arch] Report the runtime, CRI socket, CNI and CSI drivers of the cluster | Cluster Arch | easy |  |
| [Troubleshooting] kubelet on worker1 can't talk to the container runtime | Troubleshooting | medium | yes |
| [Cluster Arch] Move both nodes from containerd 1.7 to containerd 2.x | Cluster Arch | hard |  |

## Day 6: workloads, config, resources

Lesson: [`notes/day-06-workloads-config-resources.md`](../notes/day-06-workloads-config-resources.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Workloads] Roll out a bad image and roll back to a named revision | Workloads | easy |  |
| [Workloads] Configure a pod from a ConfigMap key and a Secret volume | Workloads | easy |  |
| [Workloads] Autoscale a Deployment on CPU and make the HPA report a value | Workloads | medium |  |
| [Workloads] Job with parallelism and a CronJob in a fixed time zone | Workloads | medium |  |
| [Troubleshooting] Deployment shows 1/4 ready and no pods are Pending | Troubleshooting | medium | yes |

## Day 7: scheduling and multi-container pods

Lesson: [`notes/day-07-scheduling.md`](../notes/day-07-scheduling.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Workloads] Add a native sidecar that ships the app's log file | Workloads | easy |  |
| [Troubleshooting] Three pods are Pending: name each reason from the events | Troubleshooting | easy | yes |
| [Workloads] Run a monitoring pod on the control-plane node only | Workloads | medium |  |
| [Workloads] Spread replicas with topology spread constraints | Workloads | medium |  |
| [Workloads] Make room for a critical pod with PriorityClass preemption | Workloads | hard |  |

## Day 8: Services and DNS

Lesson: [`notes/day-08-services-and-dns.md`](../notes/day-08-services-and-dns.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Networking] Expose a Deployment on a fixed NodePort | Networking | easy |  |
| [Networking] Resolve Service, headless and pod names across namespaces | Networking | medium |  |
| [Networking] Add a static host entry and a stub domain to CoreDNS | Networking | medium |  |
| [Troubleshooting] Frontend can't reach the backend Service | Troubleshooting | medium | yes |
| [Troubleshooting] Cluster DNS is broken for every pod | Troubleshooting | hard | yes |

## Day 9: NetworkPolicy, Ingress, Gateway API, CNI

Lesson: [`notes/day-09-netpol-ingress-gateway-cni.md`](../notes/day-09-netpol-ingress-gateway-cni.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Networking] Only api pods from the frontend namespace may reach the db | Networking | medium |  |
| [Networking] Default-deny a namespace, then allow DNS and one egress path | Networking | medium |  |
| [Networking] Route two hosts and a path with one Ingress | Networking | medium |  |
| [Networking] Migrate an Ingress to a Gateway and HTTPRoute | Networking | medium |  |
| [Networking] Canary with Gateway API: 90/10 split plus a header override | Networking | hard |  |

## Day 10: storage

Lesson: [`notes/day-10-storage.md`](../notes/day-10-storage.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Storage] Static PV, PVC and a pod that writes to it | Storage | easy |  |
| [Storage] New default StorageClass with WaitForFirstConsumer and expansion | Storage | medium |  |
| [Storage] Three PVCs are Pending: find each mismatch | Storage | medium | yes |
| [Storage] Keep the data: protect a dynamic volume and reuse a Released PV | Storage | medium |  |

## Day 11: troubleshooting nodes

Lesson: [`notes/day-11-troubleshoot-cluster.md`](../notes/day-11-troubleshoot-cluster.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Troubleshooting] worker1 is NotReady (and must stay Ready after a reboot) | Troubleshooting | easy | yes |
| [Troubleshooting] kubelet on worker1 is in a restart loop | Troubleshooting | medium | yes |
| [Troubleshooting] worker1 NotReady while its pods still answer | Troubleshooting | medium | yes |
| [Troubleshooting] kubelet runs on worker1 but the node stays NotReady | Troubleshooting | hard | yes |
| [Troubleshooting] worker1 says the network plugin is not ready | Troubleshooting | hard | yes |

## Day 11: troubleshooting the control plane

Lesson: [`notes/day-11-troubleshoot-cluster.md`](../notes/day-11-troubleshoot-cluster.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Troubleshooting] Report resource usage and save container logs to files | Troubleshooting | easy |  |
| [Troubleshooting] New pods stay Pending with no events at all | Troubleshooting | medium | yes |
| [Troubleshooting] Scaling a Deployment does nothing | Troubleshooting | medium | yes |
| [Troubleshooting] kubectl: connection refused (API server keeps exiting) | Troubleshooting | hard | yes |
| [Troubleshooting] API server is down and there is no container to read logs from | Troubleshooting | hard | yes |

## Day 12: application troubleshooting drills

Lesson: [`notes/day-12-mock-1-strategy-and-pacing.md`](../notes/day-12-mock-1-strategy-and-pacing.md)

| Title | Domain | Difficulty | Break-and-fix |
|---|---|---|---|
| [Troubleshooting] App in CrashLoopBackOff because a config file is missing | Troubleshooting | easy | yes |
| [Troubleshooting] Two Deployments stuck in ImagePullBackOff | Troubleshooting | easy | yes |
| [Troubleshooting] Pod restarts every few seconds and never becomes Ready | Troubleshooting | medium | yes |
| [Troubleshooting] Read logs from a crashed container in a multi-container pod | Troubleshooting | medium |  |

## Mixed mocks

| Title | Domains | Tasks |
|---|---|---|
| Mock A: 60-minute mixed set (Days 2–7) | Cluster Arch, Workloads | 8 linked issues, 60 min |
| Mock B: 60-minute mixed set (Days 8–12) | Networking, Storage, Troubleshooting | 8 linked issues, 60 min |
| Mock C: 2-hour speed run weighted like the exam | Troubleshooting, Cluster Arch, Networking, Workloads, Storage | 16 linked issues, 120 min |

## Totals

| Domain | Exam weight | Issues |
|---|---|---|
| Troubleshooting | 30% | 21 |
| Cluster Arch | 25% | 20 |
| Networking | 20% | 8 |
| Workloads | 15% | 8 |
| Storage | 10% | 4 |
| Mixed mocks | — | 3 |

Difficulty: 16 easy, 32 medium, 13 hard. Break-and-fix scenarios: 20. Total: 64 issues.

Cluster Architecture has nearly as many issues as Troubleshooting because it has 7 topics (Days 1–5 and the two extra lessons), each with at least 3 issues.
