# Day 8 Lesson: Services, EndpointSlices and cluster DNS

**Story:** [day-08-services-dns](../stories/day-08-services-dns.md) · **Read time:** ~35 min · **Then:** 2 hours in the lab

**Why this matters:** Services & Networking is **20%** of the exam. Today covers half of that domain, and tomorrow (NetworkPolicy, Ingress, Gateway API) covers the other half. Services and DNS also turn up all through **Troubleshooting (30%)**: "the frontend can't reach the backend" is nearly always one of four causes: a wrong selector, a wrong targetPort, the wrong namespace in the DNS name, or broken CoreDNS. Each has a single command that shows it. The goal today is to run the debug chain (**DNS → Service → EndpointSlice → pod labels → container port**) from memory, in that order, in under 4 minutes.

On EKS, `type: LoadBalancer` gives you an NLB, the VPC CNI gives pods VPC IPs, and CoreDNS is a managed add-on you never open. Today you see what's underneath: a Service is **a virtual IP plus a set of iptables (or nftables) rules that kube-proxy writes on every node**, and DNS is a Deployment in `kube-system` reading a ConfigMap you can break.

> **Version notes (checked today).**
> - **`v1/Endpoints` is deprecated since v1.33** ([blog](https://kubernetes.io/blog/2025/04/24/endpoints-deprecation/)). `k get endpoints` still works on your v1.35 lab and on the v1.37 exam, but prints a warning: `Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice`. That's harmless. kube-proxy reads **EndpointSlices**, so use `k get endpointslices` (or `k get ep`, which is shorter). Both are fine in an answer.
> - **`spec.trafficDistribution: PreferSameZone` / `PreferSameNode` are GA in v1.35** ([KEP-3015](https://www.kubernetes.dev/resources/keps/3015/)). `PreferClose` is now a deprecated alias for `PreferSameZone`. This is unlikely to be tested on a single-zone exam cluster, but recognize the field if you see it.
> - The exam is on **Kubernetes v1.37** (see Day 2). Nothing in today's lesson differs between your v1.35 lab and v1.37.

---

## 1. How a Service actually works

```
  k expose deploy web --port=80 --target-port=8080
                   │
                   ▼
  kube-apiserver ── allocates ClusterIP 10.96.x.y from --service-cluster-ip-range (10.96.0.0/12 on kubeadm)
                   │      (NodePort type: also allocates a port from 30000-32767)
                   ▼
  EndpointSlice controller (in kube-controller-manager)
      watches: Service.spec.selector  +  Pods (labels, IP, Ready condition)
      writes:  EndpointSlice web-abc12  label kubernetes.io/service-name=web
               endpoints: 192.168.1.10 (ready), 192.168.1.11 (ready), port 8080
                   │
                   ▼
  kube-proxy (DaemonSet, every node)  watches Services + EndpointSlices
      writes iptables/nftables NAT rules:
        dst 10.96.x.y:80        ─DNAT─► random ready endpoint 192.168.1.x:8080
        dst <anyNodeIP>:3xxxx   ─DNAT─► same endpoints      (NodePort)
                   │
                   ▼
  Calico routes the pod IP to the right node.  No process ever "listens" on the ClusterIP.
```

Points to take from that diagram:

- **The ClusterIP is virtual.** You can't ping it: ICMP isn't DNAT'd, so ping fails even on a healthy Service. Test with TCP (`wget`, `curl`, `nc`).
- **The selector is the only link** between a Service and its pods. It matches **pod labels**, not Deployment labels. A typo there gives a Service with **no endpoints**, and connections are refused or time out. Nothing logs an error.
- **Only Ready pods** become ready endpoints. A failing readiness probe removes the pod from the slice. The pod is Running, yet it gets no traffic.
- **`port`** is what clients connect to on the ClusterIP. **`targetPort`** is the container's port (a number, or a **name** from `containerPort` names). **`nodePort`** is the port on every node's IP. If `targetPort` is wrong, the endpoints exist but connections are refused.
- kube-proxy on kubeadm runs in **iptables** mode by default. Check with `k -n kube-system get cm kube-proxy -o yaml | grep mode` (empty `mode: ""` means iptables). `sudo iptables -t nat -L KUBE-SERVICES -n | grep web` shows the rules for your Service.

### Service types

| Type | What you get | Reach it from | Notes |
|---|---|---|---|
| **ClusterIP** (default) | Virtual IP inside the cluster | Pods, and nodes (kube-proxy runs there) | `clusterIP` is immutable once set |
| **NodePort** | ClusterIP **+** a port on every node (30000–32767) | `<any-node-IP>:<nodePort>` from outside | Range is the apiserver flag `--service-node-port-range` |
| **LoadBalancer** | NodePort **+** an external LB from a cloud controller | LB address | On kubeadm with no cloud controller: `EXTERNAL-IP <pending>` forever, but the NodePort works. That isn't a fault you need to fix |
| **ExternalName** | Only a DNS **CNAME** to an outside name | DNS | No selector, no ClusterIP, no endpoints, no proxying |
| **Headless** (`clusterIP: None`) | No VIP. DNS returns the **pod IPs** directly | DNS | kube-proxy ignores it. Used by StatefulSets |

---

## 2. Creating Services: fastest routes

```bash
# expose: copies the selector from the Deployment's spec.selector (or a pod's labels)
k expose deploy web --port=80 --target-port=8080                         # ClusterIP
k expose deploy web --port=80 --target-port=8080 --name=web-np --type=NodePort
k expose pod nginx --port=80 --name=nginx-svc                            # pod: selector = ALL of its labels
k run nginx --image=nginx --port=80 --expose                             # pod + ClusterIP svc in one go

# create service: selector is app=<name>. Only useful when the pods really carry app=<name>
k create service nodeport web --tcp=80:8080 --node-port=30080            # --tcp=<port>:<targetPort>
k create service clusterip db --clusterip="None" --tcp=5432:5432         # headless
k create service externalname ext --external-name=example.com

# NodePort on a SPECIFIC port with expose: there is no --node-port flag on expose
k expose deploy web --port=80 --target-port=8080 --type=NodePort $do > svc.yaml   # add nodePort: 30080
# or after the fact:
k patch svc web-np -p '{"spec":{"ports":[{"port":80,"nodePort":30080}]}}'
```

`k create deploy web` labels its pods `app=web`, so `k create service nodeport web --node-port=30080` matches them and is the fastest way to a fixed nodePort. For a Deployment with other labels, use `expose ... $do` and add the port in vim. Always check the selector afterwards.

### Complete YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: d8
spec:
  type: NodePort                 # ClusterIP | NodePort | LoadBalancer | ExternalName
  selector:
    app: web                     # must match the POD template labels (spec.template.metadata.labels)
  ports:
  - name: http                   # required when there's more than one port; becomes the SRV record name
    protocol: TCP
    port: 80                     # ClusterIP:80
    targetPort: 8080             # container port; can be a name, e.g. targetPort: http
    nodePort: 30080              # optional; omit and one is assigned. 30000-32767
  # sessionAffinity: ClientIP    # sticky by client IP (default None)
  # externalTrafficPolicy: Local # NodePort/LB only: keep the client source IP, only route to pods on the receiving node
  # internalTrafficPolicy: Local # ClusterIP: only route to endpoints on the same node as the client
---
apiVersion: v1
kind: Service
metadata:
  name: db                       # headless, for a StatefulSet with serviceName: db
spec:
  clusterIP: None
  selector: {app: db}
  ports:
  - port: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: ext
spec:
  type: ExternalName
  externalName: api.example.com  # a DNS name, not a URL and not an IP
```

**Named targetPort** (worth it on the exam because it survives port changes):

```yaml
# in the pod template
ports:
- name: http
  containerPort: 8080
# in the Service
targetPort: http
```

### A Service without a selector

If you leave `selector` out, the controller creates no EndpointSlice. You write one yourself and point it at any IP, such as a database outside the cluster. Label it `kubernetes.io/service-name: <svc>` and give its port the **same name** as the Service port:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: legacy-db-1
  labels:
    kubernetes.io/service-name: legacy-db
addressType: IPv4
ports:
- name: ""                       # matches an unnamed Service port
  port: 5432
  protocol: TCP
endpoints:
- addresses: ["10.0.0.50"]
```

---

## 3. EndpointSlices: where you look first

```bash
k get svc web -o wide                                         # SELECTOR column
k get endpointslices -l kubernetes.io/service-name=web        # ENDPOINTS column: IPs, or <unset>
k get ep web                                                  # older, shorter; prints the deprecation warning
k describe svc web | grep -E 'Selector|TargetPort|Endpoints|NodePort'
k get pods -l app=web -o wide --show-labels                   # does anything match the selector?
k get endpointslice -l kubernetes.io/service-name=web -o yaml | grep -A3 conditions   # ready / serving / terminating
```

| What you see | Cause | Fix |
|---|---|---|
| Endpoints `<none>` / slice `<unset>`, pods are running | Selector doesn't match pod labels (typo, `app` vs `run`, wrong value) | `k edit svc` and fix the selector, or `k label pod` |
| Endpoints `<none>`, pods running but `0/1 READY` | Readiness probe failing | Fix the probe or the app (`k describe pod`) |
| Endpoints listed, but `connection refused` | `targetPort` ≠ the port the container listens on | Check `k get pod -o yaml \| grep containerPort`, or `k exec <pod> -- netstat -tlnp` / `ss -tlnp`. Fix `targetPort` |
| Endpoints listed, DNS fine, but timeouts from one namespace only | NetworkPolicy (Day 9) | `k get netpol -A` |
| `nslookup` fails for every name | CoreDNS (section 5) | `k -n kube-system get pods -l k8s-app=kube-dns` |

`containerPort` in a pod spec is **informational only**. A container listening on 8080 accepts traffic on 8080 even if `containerPort` says 80. The Service's `targetPort` has to match what the process **actually** listens on.

---

## 4. Cluster DNS: the names

The cluster domain is `cluster.local` (kubelet `clusterDomain`). The DNS Service is `kube-dns` in `kube-system` at **10.96.0.10** on kubeadm (the 10th IP of the service range).

| Record | Name | Returns |
|---|---|---|
| Service A | `web.d8.svc.cluster.local` | ClusterIP |
| Headless Service A | `db.d8.svc.cluster.local` | **One A record per ready pod** IP |
| StatefulSet pod (through headless svc) | `db-0.db.d8.svc.cluster.local` | That pod's IP (stable name) |
| Pod with `hostname` + `subdomain` = a headless svc | `<hostname>.<subdomain>.d8.svc.cluster.local` | Pod IP |
| Pod A (by IP) | `192-168-1-10.d8.pod.cluster.local` | Pod IP (dashes, not dots) |
| SRV for a named port | `_http._tcp.web.d8.svc.cluster.local` | Port + target name |
| ExternalName | `ext.d8.svc.cluster.local` | **CNAME** `api.example.com` |

### How short names resolve: `/etc/resolv.conf` in a pod

```
nameserver 10.96.0.10
search d8.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- From a pod in `d8`, `web` → tries `web.d8.svc.cluster.local` first → found.
- `web.other` → tries `web.other.d8.svc...` (miss), then `web.other.svc.cluster.local` → found. So **`<svc>.<ns>`** is the shortest cross-namespace name. A bare `web` from another namespace **does not** find it.
- `ndots:5`: any name with fewer than 5 dots goes through the search list first. That's why `google.com` makes several cluster queries before the real one. You can tune it per pod with `dnsConfig`.
- The kubelet writes this file. The nameserver comes from `clusterDNS` in `/var/lib/kubelet/config.yaml`, and the domain from `clusterDomain` there.

### dnsPolicy

| `dnsPolicy` | resolv.conf comes from |
|---|---|
| `ClusterFirst` (default) | CoreDNS, as above |
| `Default` | The **node's** resolv.conf (no cluster names!). Yes, the name is misleading |
| `ClusterFirstWithHostNet` | CoreDNS, for pods with `hostNetwork: true` (which otherwise fall back to `Default`) |
| `None` | Only what you put in `spec.dnsConfig` (`nameservers`, `searches`, `options`) |

---

## 5. CoreDNS: the pieces

```bash
k -n kube-system get deploy coredns                     # 2 replicas on kubeadm
k -n kube-system get pods -l k8s-app=kube-dns -o wide   # label is still kube-dns, for compatibility
k -n kube-system get svc kube-dns                       # 10.96.0.10, ports 53/UDP, 53/TCP, 9153 (metrics)
k -n kube-system get endpointslice -l kubernetes.io/service-name=kube-dns
k -n kube-system get cm coredns -o yaml                 # key: Corefile
k -n kube-system logs -l k8s-app=kube-dns               # add the `log` plugin to see each query
```

The kubeadm default Corefile, line by line:

```
.:53 {
    errors                                   # log errors to stdout
    health {                                 # :8080/health → liveness probe
       lameduck 5s
    }
    ready                                    # :8181/ready → readiness probe
    kubernetes cluster.local in-addr.arpa ip6.arpa {   # answers *.cluster.local from the API
       pods insecure                         # enables the pod A records (1-2-3-4.ns.pod...)
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153                         # metrics
    forward . /etc/resolv.conf {             # everything else → the NODE's resolvers
       max_concurrent 1000
    }
    cache 30
    loop                                     # detects forwarding loops and exits (CrashLoopBackOff)
    reload                                   # picks up ConfigMap edits in ~30-60 s, no restart needed
    loadbalance
}
```

Common edits:

```
# a stub domain: send *.corp.local to a specific server (add as a NEW block, outside .:53)
corp.local:53 {
    errors
    cache 30
    forward . 10.0.0.53
}

# inside .:53, a static override
    hosts {
       10.0.0.99 legacy.example.com
       fallthrough
    }

# log every query (debugging only)
    log
```

After `k -n kube-system edit cm coredns`, wait for `reload`, or force it with `k -n kube-system rollout restart deploy coredns`. A syntax error makes the new pods crash. `k -n kube-system logs` shows the line number.

**`loop` crash:** if the node's `/etc/resolv.conf` points at `127.0.0.53` (systemd-resolved on Ubuntu), `forward . /etc/resolv.conf` would send queries back to CoreDNS itself. kubeadm avoids this by setting the kubelet's `resolvConf: /run/systemd/resolve/resolv.conf` in `/var/lib/kubelet/config.yaml`. If CoreDNS is in CrashLoopBackOff with `[FATAL] plugin/loop: Loop ... detected`, that's the place to look.

---

## 6. The debug flow (task 5 and acceptance criterion 2)

Run it in this order. Each step rules out one layer.

```bash
# 0. a throwaway client pod in the CLIENT's namespace (busybox 1.28: its nslookup handles search domains properly)
k run t -n d8 --image=busybox:1.28 --rm -it --restart=Never -- sh

# 1. DNS: does the name resolve?
nslookup web                         # same namespace
nslookup web.other                   # another namespace
nslookup kubernetes.default          # sanity check: if this fails too, DNS itself is broken (section 5)

# 2. Service: does the ClusterIP answer on the service port?
wget -qO- -T 2 web:80                # -T 2: don't hang for long. Use the port from `k get svc`
wget -qO- -T 2 192.168.1.10:8080     # straight to a pod IP + container port, bypassing the Service

# 3-5. back on cp1
k get svc web -o wide                                    # selector, port, targetPort
k get endpointslice -l kubernetes.io/service-name=web    # empty → selector or readiness
k get pods --show-labels -o wide                         # labels vs selector
k get pod <pod> -o jsonpath='{.spec.containers[*].ports}{"\n"}'   # vs targetPort
```

How to read the results:

| DNS | `wget svc` | `wget podIP:port` | Endpoints | Diagnosis |
|---|---|---|---|---|
| fails | — | — | — | Wrong namespace in the name, or CoreDNS down |
| ok | fails | ok | **empty** | Selector or readiness |
| ok | refused | ok on the real port | present | **targetPort** wrong |
| ok | fails | fails | present | App not listening, or NetworkPolicy |
| ok | ok | ok | present | Works. Re-read the task: wrong port number in the question? |

Non-interactive one-liners save time on the exam:

```bash
k run t --image=busybox:1.28 --rm -it --restart=Never -- nslookup web.other
k run t --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- -T 2 web.other:80
k run t --image=nicolaka/netshoot --rm -it --restart=Never -- dig +search web.other   # if you need dig
```

---

## 7. Hands-on walkthrough on cp1 + worker1

From cp1. Check `k config current-context` first. Then `k create ns d8 && k config set-context --current --namespace=d8`.

**Step 1: the four Service types (task 1).**

```bash
k create deploy web --image=nginx:1.27 --replicas=2 --port=80
k expose deploy web --port=80                                 # ClusterIP
k expose deploy web --port=8080 --target-port=80 --name=web-8080   # port ≠ targetPort
k expose deploy web --port=80 --type=LoadBalancer --name=web-lb
k get svc                                                     # web-lb: EXTERNAL-IP <pending>, but PORT(S) 80:3xxxx
k create service clusterip web-hl --clusterip=None --tcp=80:80   # headless, selector app=web-hl: matches nothing (yet)
k get ep web-hl                                               # <none>: create service used app=web-hl
k set selector svc web-hl app=web                             # fix it imperatively
k get ep web-hl                                               # 2 pod IPs
```

`k set selector` is the fastest selector fix. It **replaces** the whole selector.

**Step 2: NodePort on a fixed port, curl from the node (acceptance criterion 1).** ⏱ Target < 3 min.

```bash
# --- start the timer ---
k create deploy shop --image=nginx:1.27 --replicas=2
k expose deploy shop --port=80 --type=NodePort $do > shop-svc.yaml
vim shop-svc.yaml                       # under ports: add  nodePort: 30080
k apply -f shop-svc.yaml
k get svc shop                          # 80:30080/TCP
curl -s localhost:30080 | grep -i title
curl -s $(k get node worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'):30080 | grep -i title
# --- stop the timer ---
```

Both nodes answer, even though the shop pods may all be on worker1. That's kube-proxy's NodePort rule on every node. Now try `externalTrafficPolicy: Local` (`k patch svc shop -p '{"spec":{"externalTrafficPolicy":"Local"}}'`). Then `k scale deploy shop --replicas=1`, find which node the pod is on, and curl the **other** node's IP on 30080. It hangs, because that node has no local endpoint. Set it back to `Cluster` afterwards.

Look at the rules once: `sudo iptables -t nat -L KUBE-NODEPORTS -n | grep 30080`, then follow the `KUBE-SVC-...` chain with `sudo iptables -t nat -L KUBE-SVC-XXXX -n` to see the per-endpoint probability split.

**Step 3: break and fix (task 2, acceptance criterion 2).** ⏱ Target < 4 min for both faults.

Set it up, then give yourself a minute away from the screen so you don't remember the exact edits:

```bash
k create deploy api --image=nginx:1.27 --replicas=2
k expose deploy api --port=80
k patch svc api -p '{"spec":{"selector":{"app":"apii"}}}'                  # fault 1: selector typo
k patch svc api --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":8080}]'   # fault 2
```

Now fix it using only the debug flow in section 6:

```bash
# --- start the timer ---
k run t --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- -T 2 api   # fails
k get ep api                                    # <none>
k get svc api -o wide ; k get pods --show-labels | grep api     # app=apii vs app=api
k set selector svc api app=api
k get ep api                                    # 2 IPs, port 8080
k run t --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- -T 2 api   # connection refused
k get pod -l app=api -o jsonpath='{.items[0].spec.containers[0].ports}'      # empty: k create deploy without --port; nginx listens on 80
k patch svc api --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":80}]'
k run t --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- -T 2 api | grep title   # Welcome to nginx
# --- stop the timer ---
```

Then add a readiness probe that always fails (`k edit deploy api`, add `readinessProbe: {httpGet: {path: /nope, port: 80}}`). Watch `k get ep api` empty out while the new pods show `0/1 Running`. Compare `k get endpointslice -l kubernetes.io/service-name=api -o yaml | grep -A3 conditions`: you'll see `ready: false`. Undo with `k rollout undo deploy api`.

**Step 4: DNS names (task 3, acceptance criterion 3).**

```bash
k create ns other
k -n other create deploy backend --image=nginx:1.27
k -n other expose deploy backend --port=80
# first try, from d8:
k run t --image=busybox:1.28 --rm -it --restart=Never -- nslookup backend.other
k run t --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- -T 2 backend.other | grep title
k run t --image=busybox:1.28 --rm -it --restart=Never -- nslookup backend            # FAILS: wrong namespace via search list
```

Then in one interactive pod (`k run t --image=busybox:1.28 --rm -it --restart=Never -- sh`):

```sh
cat /etc/resolv.conf                              # nameserver 10.96.0.10, search d8.svc.cluster.local ...
nslookup backend.other.svc.cluster.local          # FQDN
nslookup web-hl                                   # headless: 2 addresses (pod IPs), not a ClusterIP
nslookup web                                      # 1 address: the ClusterIP
nslookup 192-168-1-10.d8.pod.cluster.local        # use a real pod IP from k get pods -o wide, dots → dashes
nslookup kubernetes.default                       # 10.96.0.1
```

Make a stable per-pod name without a StatefulSet. Apply this pod in `d8` and run `nslookup box1.web-hl` (it needs a label that `web-hl` selects, so it's `app: web`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: box
  labels: {app: web}
spec:
  hostname: box1
  subdomain: web-hl            # must equal the headless Service name
  containers:
  - name: c
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]
```

**Step 5: CoreDNS (task 4).**

```bash
k -n kube-system get deploy,pods,svc -l k8s-app=kube-dns       # deploy is named coredns, svc is named kube-dns
k -n kube-system get cm coredns                                # the ConfigMap has no k8s-app label, so get it by name
k -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'
sudo grep -A1 clusterDNS /var/lib/kubelet/config.yaml          # 10.96.0.10
k -n kube-system get cm kubeadm-config -o yaml | grep -i subnet   # podSubnet 192.168.0.0/16, serviceSubnet 10.96.0.0/12
sudo grep service-cluster-ip-range /etc/kubernetes/manifests/kube-apiserver.yaml
```

Break DNS and diagnose it as if it were an exam task:

```bash
k -n kube-system scale deploy coredns --replicas=0
k run t --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default   # times out
k -n kube-system get ep kube-dns                                                       # <none>: the giveaway
k -n kube-system scale deploy coredns --replicas=2
```

Then add a `hosts` block that maps `legacy.example.com` to `10.0.0.99` (section 5), wait ~60 s, and `nslookup legacy.example.com` from a pod. Then add the `log` plugin and watch `k -n kube-system logs -l k8s-app=kube-dns -f` while you run lookups. You'll see each `ndots` search-list miss. Remove both edits when you're done.

**Cleanup:** `k delete ns d8 other` and `k config set-context --current --namespace=default`. Check the Corefile is back to the default and both coredns pods are `1/1`.

---

## 8. Traps that cost points

- **Pinging a ClusterIP** to test a Service. It never answers. Use `wget`/`curl`/`nc`.
- **Selector copied from the Deployment's `metadata.labels`** instead of the pod template labels. They often differ. The Service matches **pods**.
- **`k expose pod`** copies **all** the pod's labels as the selector, including `pod-template-hash` if you expose a Deployment's pod. That Service breaks on the next rollout. Expose the **Deployment**.
- **`k create service` sets `app=<service-name>`** as the selector. If the Service name differs from the pods' `app` label, it gets no endpoints. Check `k get ep` every time.
- **`--port` vs `--target-port`** on `k expose`. If you omit `--target-port`, it defaults to `--port`. When the task says "the app listens on 8080, expose it on 80", you need both.
- **nodePort out of range** (`provided port is not in the valid range`). It must be 30000–32767 unless the apiserver flag says otherwise.
- **Testing from the wrong namespace with a short name.** A bare `web` only resolves from the Service's own namespace. Use `web.<ns>`.
- **busybox `nslookup` quirks.** Newer busybox images sometimes don't apply the search list or print `NXDOMAIN` noise for AAAA. `busybox:1.28` is the usual safe choice. If the output looks odd, use the FQDN.
- **`dnsPolicy: Default`** is not the default. It means "use the node's DNS", and cluster names stop resolving.
- **Forgetting `-n kube-system`** when looking for CoreDNS, and searching for `coredns` labels. The pod label is **`k8s-app=kube-dns`**, and so is the Service name.
- **Waiting forever on a LoadBalancer** `<pending>` on kubeadm. With no cloud controller that's normal. Use the NodePort it allocated.
- **ExternalName with an IP or a URL.** It needs a bare hostname. It also creates no endpoints, so an empty `k get ep` is expected.
- **Editing `clusterIP` or changing a ClusterIP Service to headless** in place. `clusterIP` is immutable. Delete and recreate (`k replace --force -f svc.yaml`).
- **Multi-port Services without `name:`** on each port. They're rejected: names are required when there's more than one port.
- **Assuming readiness doesn't matter.** A Running but not Ready pod gets no traffic.

---

## 9. Self-check: answer before you open the lab

1. A pod in namespace `shop` has to reach Service `pay` in namespace `billing` on port 8443. What's the shortest DNS name that works, and why does plain `pay` fail?
2. `k get ep api` shows two IPs, but `wget api` from a test pod gives "connection refused". `wget <podIP>:3000` works. What's wrong, and what's the one-line fix?
3. `k get ep api` shows `<none>` but `k get pods -l app=api` lists two Running pods. Give two possible causes and the command that tells them apart.
4. What does `nslookup db` return for a headless Service `db` with three ready pods, compared with a normal ClusterIP Service?
5. Write the commands to expose Deployment `web` (container listens on 8080) as a NodePort on port 31000, with Service port 80.
6. Every DNS lookup from every pod times out, including `kubernetes.default`. List the first three things you check, in order.
7. What's in a pod's `/etc/resolv.conf` on a kubeadm cluster, which component writes it, and where does the nameserver IP come from?
8. You need `*.corp.local` resolved by 10.0.0.53 cluster-wide. Which object do you edit, and what do you add?

<details>
<summary>Answers</summary>

1. `pay.billing` (with port 8443: `pay.billing:8443`). The search list is `shop.svc.cluster.local svc.cluster.local cluster.local`. `pay` only expands to `pay.shop.svc.cluster.local`, which doesn't exist. `pay.billing` expands to `pay.billing.svc.cluster.local` through the second search entry. The FQDN `pay.billing.svc.cluster.local` also works.
2. The Service's `targetPort` isn't 3000 (the container listens on 3000), so kube-proxy DNATs to a port nothing listens on. Endpoints exist because the selector matches. Fix: `k patch svc api --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":3000}]'` (or `k edit svc api`).
3. (a) The Service selector doesn't match the pods' labels. (b) The pods aren't Ready (failing readiness probe). Tell them apart with `k get pods -l app=api` (READY `0/1` vs `1/1`) together with `k get svc api -o wide` (SELECTOR column). If the pods are 1/1 and listed by `-l app=api` but the Service selector says something else, it's (a). You can also read `k get endpointslice -l kubernetes.io/service-name=api -o yaml`: with (b) the endpoints are there with `ready: false`, and with (a) there are no endpoints at all.
4. Headless: three A records, one per ready pod IP, and no ClusterIP. Normal Service: one A record, the ClusterIP. kube-proxy has no rules for a headless Service. The client picks a pod IP itself.
5. `k expose deploy web --port=80 --target-port=8080 --type=NodePort $do > s.yaml`, add `nodePort: 31000` under the port in vim, then `k apply -f s.yaml`. (Or expose, then `k patch svc web -p '{"spec":{"ports":[{"port":80,"nodePort":31000}]}}'`.) Check with `curl <nodeIP>:31000`.
6. (1) `k -n kube-system get pods -l k8s-app=kube-dns`: are the CoreDNS pods Running and Ready, or CrashLoopBackOff? Read their logs. (2) `k -n kube-system get ep kube-dns`: does the Service have endpoints? (3) `cat /etc/resolv.conf` in a pod: is the nameserver the `kube-dns` ClusterIP (10.96.0.10), and does it match `clusterDNS` in `/var/lib/kubelet/config.yaml`? After those: the Corefile for edits and syntax errors, kube-proxy pods, and NetworkPolicies blocking port 53.
7. `nameserver 10.96.0.10`, `search <ns>.svc.cluster.local svc.cluster.local cluster.local`, `options ndots:5`. The **kubelet** writes it when it creates the pod sandbox. The nameserver is `clusterDNS` and the domain is `clusterDomain` from `/var/lib/kubelet/config.yaml` (kubeadm fills them in from the `serviceSubnet` and cluster config).
8. The `coredns` ConfigMap in `kube-system` (key `Corefile`). Add a separate server block: `corp.local:53 { errors; cache 30; forward . 10.0.0.53 }` (one directive per line). The `reload` plugin picks it up within about a minute, or run `k -n kube-system rollout restart deploy coredns`. Then test with `nslookup something.corp.local` from a pod.

</details>

---

**Done when:** every [Day 8 acceptance criterion](../stories/day-08-services-dns.md#acceptance-criteria) passes: a Deployment exposed as NodePort on a fixed nodePort and curled from the node in < 3 min; a Service with a wrong selector **and** a wrong targetPort fixed in < 4 min using the debug flow; a Service in another namespace resolved from a busybox pod on the first try. Tomorrow (Day 9) builds directly on today's Services: NetworkPolicies decide which pods can reach these endpoints, and Ingress and Gateway API routes point at the Services you made today by `name` and `port`.
