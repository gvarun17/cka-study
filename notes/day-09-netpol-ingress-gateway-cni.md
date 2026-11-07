# Day 9 Lesson: NetworkPolicy, Ingress, Gateway API and CNI

**Story:** [day-09-netpol-ingress-gateway](../stories/day-09-netpol-ingress-gateway.md) · **Read time:** ~40 min · **Then:** 2–2.5 hours in the lab

**Why this matters:** Services & Networking is **20%** of the exam. Yesterday covered Services and DNS, and today covers the rest of the domain. Expect at least one NetworkPolicy task, one Ingress or Gateway API task (often "migrate this Ingress to an HTTPRoute" or "split traffic between two versions"), and possibly "install a CNI that supports NetworkPolicy". NetworkPolicy is the topic that costs the most points per minute. The YAML is short, but one misplaced `-` turns an AND into an OR, and the policy passes `k apply` while letting the wrong traffic through. The rule today: **you haven't finished a policy until you've seen it block one connection and allow another.**

On EKS you used Security Groups for Pods or the VPC CNI's network policy agent, the AWS Load Balancer Controller for Ingress, and never thought about `/etc/cni/net.d`. Here you'll see that a NetworkPolicy is **only an object in etcd** until a CNI that supports policy (Calico, on your lab) turns it into iptables/eBPF rules on each node. Ingress and Gateway resources are also **only objects** until a controller you installed reads them.

> **Version notes (checked today).**
> - **Ingress NGINX was retired in March 2026** ([Kubernetes blog, Nov 2025](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/); [Steering/SRC statement, Jan 2026](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/)). The final releases came out on Mar 19, 2026 (`controller-v1.15.1`), and there will be no more security fixes. It's still the simplest controller to install for a **lab**, so we use it below, but don't run it in production. The **Ingress API** (`networking.k8s.io/v1`) is **not** deprecated. It's GA and frozen: no new features, and the project points new work at Gateway API.
> - **Gateway API v1.6.0** is current (released Jun 30, 2026; [blog](https://kubernetes.io/blog/2026/08/03/gateway-api-v1-6-release/)). v1.6 moved **TCPRoute and UDPRoute** to Standard. **v1.5** (Feb 2026, [blog](https://kubernetes.io/blog/2026/04/21/gateway-api-v1-5/)) moved **ListenerSet, TLSRoute, ReferenceGrant (now `v1`), the HTTPRoute CORS filter**, and client-certificate validation to Standard. v1.5 also added a ValidatingAdmissionPolicy, `safe-upgrades.gateway.networking.k8s.io`, that **blocks installing experimental CRDs over standard ones and blocks downgrading below 1.5**. If a CRD apply is rejected, that policy is usually why. Gateway, GatewayClass and HTTPRoute have been `gateway.networking.k8s.io/v1` since v1.0. That's all you need for the exam.
> - **NGINX Gateway Fabric 2.8.1** is the current release ([install docs](https://docs.nginx.com/nginx-gateway-fabric/install/helm/)). It bundles a pinned Gateway API CRD set, so install the CRDs from NGF's tag rather than mixing versions.
> - The exam is on **Kubernetes v1.37** (see Day 2). NetworkPolicy (`networking.k8s.io/v1`) hasn't changed in years. Nothing here differs between your v1.35 lab and the exam.

---

## 1. How NetworkPolicy actually works

```
  k apply -f np.yaml ──► kube-apiserver ──► etcd       (that's ALL Kubernetes itself does)
                                │
                                ▼ watch NetworkPolicies, Pods, Namespaces
                   calico-node (DaemonSet, every node)  ◄── the CNI's policy agent
                                │
                                ▼ writes iptables (or eBPF) rules per pod interface (cali*)
                     packet to/from a pod ──► allowed or dropped AT THE NODE
```

- **No enforcing CNI, no effect.** Flannel and the bare `bridge` plugin accept NetworkPolicy objects and silently ignore them. Calico, Cilium and Antrea enforce them. If a task says "install a CNI that supports NetworkPolicy", Flannel is the wrong answer.
- **A pod starts non-isolated.** It accepts everything and can send anywhere.
- **A pod becomes isolated for ingress** as soon as **any** policy with `Ingress` in `policyTypes` selects it. Egress works the same way. From then on, only traffic that **some** policy allows gets through.
- **Policies are additive.** They're a union of allow-lists. There's no "deny" rule and no ordering. Two policies on the same pod mean "allowed if either allows it". You can't write a policy that takes away what another one grants.
- **Both ends must agree.** For A → B: if A is egress-isolated, an egress rule on A must allow B, **and** if B is ingress-isolated, an ingress rule on B must allow A.
- **Replies are implicit.** Enforcement is stateful. If the request is allowed, the reply is too. You never write a rule for return traffic.
- **Ports are pod ports, not Service ports.** kube-proxy DNATs `ClusterIP:80 → podIP:8080` **before** the policy is checked at the destination pod. A policy for a Service `80 → targetPort 8080` must say `port: 8080`.
- **Node ↔ local pod is always allowed.** The docs say traffic between a pod and the node it runs on is allowed regardless of policy, which is why kubelet liveness and readiness probes keep working under default-deny.

### The shape of a policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow
  namespace: backend            # a policy only selects pods in ITS OWN namespace
spec:
  podSelector:                  # WHICH pods this policy protects. {} = every pod in the namespace
    matchLabels: {app: db}
  policyTypes: [Ingress, Egress]
  ingress:                      # list of rules; a rule matches if from AND ports both match
  - from:                       # list of peers, ORed
    - podSelector: {...}        #   pods in the policy's own namespace
    - namespaceSelector: {...}  #   all pods in matching namespaces
    - ipBlock: {cidr: 10.0.0.0/8, except: [10.1.0.0/16]}
    ports:                      # list, ORed. Omitted = all ports
    - {protocol: TCP, port: 5432}
    - {protocol: TCP, port: 6000, endPort: 6010}   # port range
  egress:
  - to: [...]
    ports: [...]
```

### `policyTypes`, and empty vs missing

| You write | Meaning |
|---|---|
| `policyTypes` omitted | `Ingress` always, plus `Egress` **only if** an `egress:` section exists. Always write it out |
| `policyTypes: [Ingress]` and no `ingress:` | **Deny all ingress** to selected pods |
| `ingress: [{}]` (one empty rule) | **Allow all** ingress |
| `ingress: - from: []` / `from` omitted, with `ports:` | Allow from **anywhere**, but only those ports |
| `podSelector: {}` in `spec` | Every pod in the namespace |
| `- podSelector: {}` in `from` | Every pod in **the policy's namespace** |
| `- namespaceSelector: {}` in `from` | Every pod in **every** namespace (not external IPs) |

### AND vs OR: the trap the story names

```yaml
# (A) ONE peer with two selectors  →  AND
#     "pods labeled role=api THAT ARE IN namespaces labeled team=web"
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: {team: web}
      podSelector:              # no dash: same list item as namespaceSelector
        matchLabels: {role: api}

# (B) TWO peers  →  OR
#     "any pod in namespaces labeled team=web, OR pods labeled role=api in THIS namespace"
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: {team: web}
    - podSelector:              # dash: a second list item
        matchLabels: {role: api}
```

Say it out loud: **"a dash starts a new peer, and peers are ORed. Selectors inside one peer are ANDed."** A `podSelector` on its own, with no `namespaceSelector` beside it, **always means the policy's own namespace**. That's why (B)'s second peer doesn't match `role=api` pods in other namespaces. Cross-namespace pod matching needs form (A).

The same AND/OR logic applies one level up: within one `ingress` rule, `from` **AND** `ports` must both match, and separate rules (separate `-` under `ingress:`) are ORed.

### Selecting namespaces by name

Since v1.22 every namespace gets the immutable label **`kubernetes.io/metadata.name=<name>`**. Use it rather than adding your own label, unless the task names a label:

```yaml
- namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: frontend
```

`k get ns --show-labels` shows it. If the task says "namespaces labeled `env=prod`", check that the label actually exists on the namespace. If it doesn't, the task wants you to add it (`k label ns frontend env=prod`).

### ipBlock

`ipBlock` is for traffic **from outside the cluster or to outside it**: CIDRs of a corporate network, a database VM, a metadata endpoint. Don't use it to match pods, because pod IPs change and ingress traffic is often SNATed by the load balancer or the node. `except` must fall inside `cidr`.

---

## 2. The four policies you'll write most

There's **no `kubectl create networkpolicy`**. Copy from the [docs page](https://kubernetes.io/docs/concepts/services-networking/network-policies/), which has default-deny examples near the bottom, or type from memory. Keep a skeleton in your fingers.

```yaml
# 1. default deny ALL ingress and egress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: backend}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# 2. ...then let every pod in the namespace still use cluster DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-dns, namespace: backend}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: kube-system}
      podSelector:
        matchLabels: {k8s-app: kube-dns}
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
---
# 3. acceptance criterion 1: only role=api pods in namespace frontend may reach db on 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-from-api, namespace: backend}
spec:
  podSelector:
    matchLabels: {app: db}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: frontend}
      podSelector:                       # AND: no dash
        matchLabels: {role: api}
    ports:
    - {protocol: TCP, port: 5432}
---
# 4. allow egress only to an external CIDR on 443 (plus the DNS policy above)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: egress-to-partner, namespace: backend}
spec:
  podSelector:
    matchLabels: {app: worker}
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock: {cidr: 203.0.113.0/24}
    ports:
    - {protocol: TCP, port: 443}
```

Notes on those:

- **DNS egress (policy 2)** is the thing people forget. Once egress is denied, a pod can't resolve `db`. `wget db:5432` then fails with `bad address 'db'` and you wrongly blame the db policy. The `kube-dns` pods listen on **53** (the Corefile says `.:53`), so port 53 is right here. Note the AND form again: kube-dns pods **in** kube-system.
- **The "only X may reach db" task (policy 3)** with nothing else in the namespace: that one policy is enough. It isolates the db pods for ingress and allows only the one peer. You don't need a default-deny as well, unless the task asks for it.
- If the source namespace (`frontend`) has a default-deny egress, you also need an **egress** policy there that allows `role=api` → db on 5432 (plus DNS). Read the existing policies in **both** namespaces: `k get netpol -A`.

### Reading a policy quickly

```bash
k get netpol -A                          # what exists, where. POD-SELECTOR column
k describe netpol db-from-api -n backend # prints the rules in English: "Allowing ingress traffic: To Port: 5432/TCP From: NamespaceSelector: ... PodSelector: ..."
```

`k describe` is the fastest way to catch AND vs OR. A single `From:` block listing both selectors is AND. Two `From:` blocks, or a `----------` separator between them, means OR.

---

## 3. Testing every policy: before and after

```bash
# a client with known labels in a known namespace
k -n frontend run api --image=busybox:1.36 -l role=api -- sleep 3600
k -n frontend run other --image=busybox:1.36 -l role=web -- sleep 3600

# wget against the Service name, short timeout
k -n frontend exec api   -- wget -qO- -T 2 db.backend:5432     # expect: allowed
k -n frontend exec other -- wget -qO- -T 2 db.backend:5432     # expect: "download timed out"
```

How to read the result:

| Output | Meaning |
|---|---|
| Page content (or a protocol error from a non-HTTP server) | TCP connected, so it's **allowed** |
| `wget: download timed out` | Packets dropped, so **blocked by policy** (or nothing routes there) |
| `wget: can't connect ... Connection refused` | Reached the pod, nothing listens on that port. Not a policy problem |
| `wget: bad address 'db.backend'` | **DNS** failed. Usually egress denied without the DNS allow |

Always test **one allowed and one denied** case, and test **before** applying the policy too, so you know the connection worked to begin with.

`nc` is the alternative for non-HTTP ports: `nc -zv -w 2 db.backend 5432` (busybox `nc` supports `-z` in recent images). Timeouts mean blocked.

---

## 4. Ingress: how it works

```
 client ─► node:NodePort (or cloud LB) ─► ingress-nginx controller pod ─► Service web:80 ─► pods
                                              ▲   (reads Ingress objects
                                              │    whose ingressClassName = nginx)
                                     Ingress: host app.example.com, path /api → svc api:8080
```

- An **Ingress** is only routing rules (host + path → Service:port). An **Ingress controller** (a pod you install) watches Ingresses for **its** IngressClass and configures its proxy.
- The **IngressClass** object names the controller (`spec.controller: k8s.io/ingress-nginx`). An Ingress picks one with `spec.ingressClassName`. If you omit it, the class annotated `ingressclass.kubernetes.io/is-default-class: "true"` is used. If there's no default class, **no controller picks it up** and the ADDRESS stays empty. The old annotation `kubernetes.io/ingress.class` is deprecated.
- The controller usually proxies **straight to the pod IPs** from the EndpointSlices, so Service `port` must exist and the endpoints must be ready. The Day 8 debug chain still applies.
- **pathType:** `Exact` (only `/foo`), `Prefix` (by path element: `/foo` matches `/foo` and `/foo/bar` but **not** `/foobar`), `ImplementationSpecific` (up to the controller). It's required.
- **Precedence:** longest matching path wins; Exact beats Prefix at the same length. Host rules without a host (`host` omitted) match any host.
- **TLS:** `spec.tls[].secretName` references a `kubernetes.io/tls` Secret **in the same namespace** as the Ingress, with keys `tls.crt` and `tls.key`.

### Fastest route: `k create ingress`

```bash
# the path ending in * means pathType Prefix; without * it's Exact
k create ingress web --class=nginx \
  --rule="app.example.com/*=web:80" \
  --rule="app.example.com/api*=api:8080" $do > ing.yaml

# TLS on a host rule (Secret 'app-tls' must exist in the namespace)
k create secret tls app-tls --cert=tls.crt --key=tls.key
k create ingress web-tls --class=nginx --rule="app.example.com/*=web:80,tls=app-tls"

# default backend, and an annotation
k create ingress catchall --class=nginx --default-backend=web:80 \
  --annotation=nginx.ingress.kubernetes.io/rewrite-target=/
```

### Complete YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: d9
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # controller-specific; only if the task asks
spec:
  ingressClassName: nginx
  tls:
  - hosts: [app.example.com]
    secretName: app-tls
  defaultBackend:                   # optional: anything that matches no rule
    service: {name: web, port: {number: 80}}
  rules:
  - host: app.example.com           # host-based routing
    http:
      paths:
      - path: /api                  # path-based routing
        pathType: Prefix
        backend:
          service:
            name: api
            port: {number: 8080}    # or  port: {name: http}
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port: {number: 80}
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: admin, port: {number: 80}}
```

`backend.service.port` is the **Service port**, not the targetPort. The old `serviceName`/`servicePort` fields are from `extensions/v1beta1`, which was removed long ago. If you see them in a task file, the task wants you to fix them.

Checking an Ingress:

```bash
k get ingressclass                         # NAME nginx, CONTROLLER k8s.io/ingress-nginx; (default) marker
k get ing                                  # CLASS, HOSTS, ADDRESS (empty = no controller has picked it up)
k describe ing web                         # Rules table with backends and their endpoint IPs; Events
curl -s -H 'Host: app.example.com' http://<nodeIP>:<nodePort>/api
curl -s --resolve app.example.com:<nodePort>:<nodeIP> http://app.example.com:<nodePort>/
```

---

## 5. Gateway API: how it works

Gateway API splits Ingress into three objects, one per role:

```
  GatewayClass  "nginx"            ← infra provider: which controller (cluster-scoped, like IngressClass)
       ▲ gatewayClassName
  Gateway       "web-gw"  (ns infra)  ← cluster operator: listeners = port + protocol + hostname (+ TLS)
       ▲ parentRefs                      allowedRoutes decides which namespaces may attach
  HTTPRoute     "app"     (ns d9)     ← app team: matches (path, headers, query, method) → backendRefs (Service:port, weight)
       │ backendRefs
       ▼
  Service web-v1:80 (80%) / web-v2:80 (20%)
```

What Ingress can't do natively and HTTPRoute can, all without annotations: **header and query matching, weighted traffic splitting, request/response header changes, redirects and URL rewrites, mirroring, and cross-namespace routing**. That's why exam tasks use it for canary splits.

Key rules:

- **The CRDs aren't built in.** Kubernetes doesn't ship Gateway API. You install the CRDs (standard channel) plus an implementation (NGINX Gateway Fabric, Envoy Gateway, Istio, Cilium, Traefik...). On the exam they'll almost certainly already be installed. Check `k get gatewayclass` and `k api-resources | grep gateway`.
- **Attachment is two-way.** The route names the Gateway in `parentRefs` (with `namespace:` if it's elsewhere). The Gateway's listener must allow that namespace: `allowedRoutes.namespaces.from: Same` (the default), `All`, or `Selector`. If the route lives in another namespace and the listener says `Same`, the route is **not accepted**.
- **Cross-namespace backends need a ReferenceGrant** in the **backend's** namespace. Same-namespace backends don't.
- **Match logic matches NetworkPolicy:** inside one `matches` item, `path` AND `headers` AND `queryParams` AND `method` must all match. Separate `matches` items are ORed. Separate `rules` are separate routes. Precedence: Exact path > longest prefix > method > most header matches > most query matches.
- **`weight`** is relative, not a percentage. `80`/`20` and `4`/`1` mean the same. `weight: 0` sends nothing. Omitted means 1.
- **Status is where you debug.** Gateway conditions `Accepted` and `Programmed`. HTTPRoute `status.parents[].conditions`: `Accepted` (attached to the listener) and `ResolvedRefs` (the backends exist and are allowed).

There's **no imperative `k create` for these**. Copy from the [kubernetes.io Gateway API page](https://kubernetes.io/docs/concepts/services-networking/gateway/), which has a full Gateway and HTTPRoute, and use `k explain httproute.spec.rules --recursive` for field names.

### Complete YAML

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gw
  namespace: d9
spec:
  gatewayClassName: nginx               # from: k get gatewayclass
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.example.com"           # optional; restricts which route hostnames attach
    allowedRoutes:
      namespaces:
        from: Same                      # Same (default) | All | Selector
  - name: https
    protocol: HTTPS
    port: 443
    hostname: app.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: app-tls                   # kubernetes.io/tls Secret in the Gateway's namespace
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app
  namespace: d9
spec:
  parentRefs:
  - name: web-gw
    sectionName: http                   # optional: attach to one listener only
  hostnames: ["app.example.com"]
  rules:
  # rule 1: header match → v2 only (the "beta testers" pattern)
  - matches:
    - path: {type: PathPrefix, value: /}
      headers:
      - {type: Exact, name: x-canary, value: "true"}
    backendRefs:
    - {name: web-v2, port: 80}
  # rule 2: everything else under / → 80/20 split (acceptance criterion 2)
  - matches:
    - path: {type: PathPrefix, value: /}
    backendRefs:
    - {name: web-v1, port: 80, weight: 80}
    - {name: web-v2, port: 80, weight: 20}
  # rule 3: /api → another Service, with a header added on the way
  - matches:
    - path: {type: PathPrefix, value: /api}
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - {name: x-from, value: gateway}
    backendRefs:
    - {name: api, port: 8080}
---
# HTTP → HTTPS redirect: a route attached only to the http listener
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: redirect, namespace: d9}
spec:
  parentRefs: [{name: web-gw, sectionName: http}]
  hostnames: ["secure.example.com"]
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect: {scheme: https, statusCode: 301}
---
# route in namespace d9 → Service in namespace shared: allowed by a grant IN 'shared'
apiVersion: gateway.networking.k8s.io/v1      # v1 since Gateway API 1.5 (v1beta1 still served)
kind: ReferenceGrant
metadata: {name: allow-d9-routes, namespace: shared}
spec:
  from:
  - {group: gateway.networking.k8s.io, kind: HTTPRoute, namespace: d9}
  to:
  - {group: "", kind: Service}
```

Path match types: `Exact`, `PathPrefix`, `RegularExpression` (implementation-specific). Header match types: `Exact` (default), `RegularExpression`. `URLRewrite` with `path: {type: ReplacePrefixMatch, replacePrefixMatch: /}` is the equivalent of nginx's `rewrite-target`.

### Ingress → HTTPRoute mapping (for "migrate this Ingress" tasks)

| Ingress | Gateway API |
|---|---|
| `ingressClassName: nginx` | Gateway `gatewayClassName`, and the route's `parentRefs` → that Gateway |
| `rules[].host` | HTTPRoute `hostnames` (one route per host, or several hostnames in one) |
| `paths[].path` + `pathType: Prefix` | `matches[].path: {type: PathPrefix, value: ...}` |
| `pathType: Exact` | `type: Exact` |
| `backend.service.name` / `port.number` | `backendRefs[].name` / `port` |
| `tls[].secretName` | Gateway listener `protocol: HTTPS`, `tls.certificateRefs` |
| `defaultBackend` | A rule with no `matches` (matches `/` prefix) |
| `rewrite-target` annotation | `URLRewrite` filter |
| canary annotations | `weight` on `backendRefs` |

### Checking Gateway API objects

```bash
k get gatewayclass                          # ACCEPTED True
k get gateway -n d9                         # CLASS, ADDRESS, PROGRAMMED True
k describe gateway web-gw -n d9             # Listeners: Attached Routes count, conditions per listener
k get httproute -n d9                       # HOSTNAMES
k describe httproute app -n d9              # Status → Parents → Conditions: Accepted, ResolvedRefs (+ reason)
k get httproute app -n d9 -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
```

| Condition | Typical reason when False | Fix |
|---|---|---|
| Gateway `Accepted=False` | `InvalidParameters`, unknown class | `gatewayClassName` typo: compare with `k get gatewayclass` |
| Gateway `Programmed=False` | No address yet | The controller isn't running, or its Service has no address. Check the controller pods |
| Route `Accepted=False` | `NotAllowedByListeners`, `NoMatchingListenerHostname`, `NoMatchingParent` | Namespace not allowed, hostname mismatch, or `parentRefs` name/sectionName typo |
| Route `ResolvedRefs=False` | `BackendNotFound`, `RefNotPermitted` | Service name/port wrong, or a cross-namespace backend without a ReferenceGrant |

---

## 6. CNI: where things live

```
 kubelet ──CRI──► containerd ──creates pod sandbox (netns)──► runs CNI plugin
                                   reads:  /etc/cni/net.d/   (first file alphabetically, e.g. 10-calico.conflist)
                                   execs:  /opt/cni/bin/<type>  (calico, calico-ipam, bandwidth, portmap, loopback ...)
                               plugin: creates the veth (cali*), assigns an IP from IPAM, programs routes
```

| Thing | Where |
|---|---|
| CNI network config | `/etc/cni/net.d/` on **every node** (`10-calico.conflist` plus a `calico-kubeconfig`) |
| CNI plugin binaries | `/opt/cni/bin/` |
| containerd's CNI settings | `/etc/containerd/config.toml`, the CRI plugin's `conf_dir` / `bin_dir` (defaults to the two paths above) |
| Pod CIDR given to kubeadm | `--pod-network-cidr` → `k -n kube-system get cm kubeadm-config -o yaml \| grep podSubnet` |
| Calico's own view (operator install) | `k get installation default -o yaml` (ipPools, encapsulation), `k get tigerastatus` |
| Calico pods | `calico-system` namespace: `calico-node` DaemonSet, `calico-kube-controllers`, `calico-typha` |
| Routes to other nodes' pods | `ip route` on a node: one route per remote node's block via `vxlan.calico` or `tunl0` |

Symptoms of a missing or broken CNI:

- Nodes **`NotReady`**, and `k describe node` says `container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized`.
- CoreDNS stays **`Pending`** or `ContainerCreating`. Pods show `failed to setup network for sandbox` in their events.
- `/etc/cni/net.d/` is empty on that node.

Installing one from a manifest (the exam phrasing is usually "install Calico/Flannel/Cilium using this manifest URL, making sure it supports NetworkPolicy"):

```bash
# Calico, operator method (Day 1). The operator manifest is too big for client-side apply annotations:
k create -f https://raw.githubusercontent.com/projectcalico/calico/<VERSION>/manifests/tigera-operator.yaml
# custom-resources.yaml: check its cidr matches podSubnet BEFORE creating it
curl -sO https://raw.githubusercontent.com/projectcalico/calico/<VERSION>/manifests/custom-resources.yaml
grep -n cidr custom-resources.yaml          # 192.168.0.0/16 by default
k create -f custom-resources.yaml
k get pods -n calico-system -w ; k get nodes          # Ready once calico-node is Running
```

If the task gives a manifest URL, use **exactly that URL**. If the pod CIDR differs from the manifest default, edit the CIDR before you apply.

---

## 7. Hands-on walkthrough on cp1 + worker1

From cp1. Check `k config current-context` first.

```bash
k create ns d9 && k config set-context --current --namespace=d9
k create ns frontend ; k create ns backend
```

**Step 1: the db and some clients (setup for task 1).** We need something that listens on 5432 and speaks HTTP, so `wget` gives clear results. Python's built-in server does that:

```bash
k -n backend create deploy db --image=python:3.12-alpine --replicas=2 -- python -m http.server 5432
k -n backend expose deploy db --port=5432
k -n backend get pods --show-labels           # app=db (create deploy sets it)

k -n frontend run api   --image=busybox:1.36 -l role=api -- sleep 3600
k -n frontend run web   --image=busybox:1.36 -l role=web -- sleep 3600
k -n d9       run spy   --image=busybox:1.36 -l role=api -- sleep 3600   # right label, wrong namespace

# BEFORE: all three connect
for p in frontend/api frontend/web d9/spy; do
  echo "== $p"; k -n ${p%/*} exec ${p#*/} -- wget -qO- -T 2 db.backend:5432 | head -1
done
```

**Step 2: default deny, then watch DNS break (tasks 1 and 3).**

```bash
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: frontend}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
k -n frontend exec api -- wget -qO- -T 2 db.backend:5432      # bad address 'db.backend': DNS is blocked first
k -n frontend exec api -- nslookup db.backend                  # times out
```

That error is the lesson. Now apply `allow-dns` from section 2 in **frontend** (change the namespace), and retry: the name resolves, but the connection now **times out**. Egress to the db isn't allowed yet. Add an egress allow from `role=api` to the db pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: api-to-db, namespace: frontend}
spec:
  podSelector: {matchLabels: {role: api}}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: backend}}
      podSelector: {matchLabels: {app: db}}
    ports:
    - {protocol: TCP, port: 5432}
```

`api` connects, and `web` times out. That's egress isolation on the client side doing the work. Delete the three frontend policies before the timed run: `k -n frontend delete netpol --all`.

**Step 3: acceptance criterion 1, timed.** ⏱ Target **< 6 min**, including both tests.

```bash
# --- start the timer ---
vim db-np.yaml                                   # policy 3 from section 2, typed, not pasted
k apply -f db-np.yaml
k -n backend describe netpol db-from-api         # ONE From: block with both selectors = AND
k -n frontend exec api -- wget -qO- -T 2 db.backend:5432 | head -1   # allowed
k -n frontend exec web -- wget -qO- -T 2 db.backend:5432             # download timed out
k -n d9       exec spy -- wget -qO- -T 2 db.backend:5432             # download timed out
# --- stop the timer ---
```

**Step 4: break it into an OR on purpose (acceptance criterion 3).** Edit `db-np.yaml` and put a `-` in front of `podSelector:` so it becomes a second peer. Apply it and rerun the three tests:

- `frontend/api` is allowed (whole namespace frontend matches the first peer).
- `frontend/web` is now **allowed**, for the same reason. That's the leak.
- `d9/spy` is still blocked: the second peer, `podSelector` alone, only means `role=api` pods **in backend**.

Compare `k describe netpol` output for both versions. Then explain the difference out loud, without looking, in two sentences. That's criterion 3. Put the dash back.

Try also: change `port: 5432` to a Service port that differs from the container port (expose db as `--port=80 --target-port=5432`, so `wget db.backend:80`) and see that the policy must still say **5432**.

**Step 5: ipBlock and namespaceSelector by your own label.**

```bash
k label ns frontend team=web
# in db-np.yaml, change the namespaceSelector to  matchLabels: {team: web}  → apply → api still allowed, web still blocked
k label ns frontend team-                     # remove the label → api is now blocked too: the selector matches no namespace
k label ns frontend team=web
```

Then see the node exception. Run `k -n backend get pods -o wide`, and from the node that hosts a db pod, `curl -s -m 2 <that-db-pod-IP>:5432 | head -1`. It answers even though no policy allows the node: node → local pod is always allowed. That's also why ipBlock is for **external** CIDRs. Add a peer `- ipBlock: {cidr: 203.0.113.0/24}` to see how it renders in `k describe netpol`, then remove it.

**Step 6: Ingress (task 5).** Install ingress-nginx's bare-metal manifest. It gives you a NodePort Service. Use the final release, for the lab only (see the version notes):

```bash
k apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
k -n ingress-nginx wait --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=180s
k get ingressclass                                    # nginx
k -n ingress-nginx get svc ingress-nginx-controller   # 80:3xxxx  → note the HTTP nodePort
export IP=$(k get node worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
export NP=$(k -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
```

Backends: two versions that answer differently, so you can see which one served you:

```bash
k create deploy web-v1 --image=nginx:1.27      # "Welcome to nginx!"
k create deploy web-v2 --image=httpd:2.4       # "It works!"
k create deploy api    --image=nginx:1.27
k expose deploy web-v1 --port=80 ; k expose deploy web-v2 --port=80 ; k expose deploy api --port=8080 --target-port=80
```

Path-based, then host-based:

```bash
k create ingress paths --class=nginx --rule="/*=web-v1:80" --rule="/api*=api:8080"
curl -s $IP:$NP/       | grep -o '<title>.*</title>'      # nginx (web-v1)
curl -s $IP:$NP/api    | grep -o '<title>.*</title>'      # 404 from nginx inside api: /api is passed through unchanged
k delete ing paths

k create ingress hosts --class=nginx --rule="v1.example.com/*=web-v1:80" --rule="v2.example.com/*=web-v2:80"
curl -s -H 'Host: v1.example.com' $IP:$NP/ | grep -o '<title>.*</title>'
curl -s -H 'Host: v2.example.com' $IP:$NP/                 # It works!
curl -s -H 'Host: nope.example.com' $IP:$NP/ -o /dev/null -w '%{http_code}\n'   # 404 from the controller's default backend
```

The `/api` 404 is worth a minute: Ingress forwards the path as-is. Add `--annotation=nginx.ingress.kubernetes.io/rewrite-target=/` (only valid for ingress-nginx) and the api backend gets `/`. Then remove `--class=nginx` from one Ingress and watch ADDRESS stay empty in `k get ing`, because there's no default IngressClass. Fix it either way: `ingressClassName`, or `k annotate ingressclass nginx ingressclass.kubernetes.io/is-default-class=true`.

TLS, if time allows: cert-manager from Day 5 is still installed, or make a self-signed pair with `openssl req -x509 -nodes -newkey rsa:2048 -subj /CN=v1.example.com -keyout tls.key -out tls.crt -days 30`, then `k create secret tls v1-tls --cert=tls.crt --key=tls.key` and `--rule="v1.example.com/*=web-v1:80,tls=v1-tls"`. Test with `curl -k --resolve v1.example.com:<httpsNodePort>:$IP https://v1.example.com:<httpsNodePort>/`.

**Step 7: Gateway API (task 6).** Install the CRDs from NGF's pinned tag, then NGF with a NodePort data plane:

```bash
k kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.8.1" | k apply -f -
k api-resources --api-group=gateway.networking.k8s.io    # gatewayclasses, gateways, httproutes, grpcroutes, referencegrants ...
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway \
  --set nginx.service.type=NodePort
k -n nginx-gateway get pods
k get gatewayclass                                       # nginx   ACCEPTED True
```

Apply the `web-gw` Gateway from section 5 with **only the `http` listener**, and drop its `hostname` line. NGF then creates an nginx data-plane Deployment and a NodePort Service for that Gateway in `d9`:

```bash
k get gateway web-gw                                     # PROGRAMMED True
k get svc                                                # the Gateway's Service: 80:3xxxx
export GNP=<that nodePort>
```

Now **acceptance criterion 2, timed.** ⏱ Target **< 6 min**.

```bash
# --- start the timer ---
vim split.yaml        # HTTPRoute: parentRefs web-gw, hostnames app.example.com, one rule, backendRefs web-v1 80 / web-v2 20
k apply -f split.yaml
k describe httproute split | grep -A3 -E 'Type: +(Accepted|ResolvedRefs)'   # both True
for i in $(seq 1 50); do curl -s -H 'Host: app.example.com' $IP:$GNP/ | grep -o -E 'Welcome to nginx|It works'; done | sort | uniq -c
#   ~40 Welcome to nginx / ~10 It works
# --- stop the timer ---
```

Then add rule 1 from section 5 (header `x-canary: true` → web-v2) and check that `curl -H 'x-canary: true' ...` always returns "It works!". Then break things and read the status:

- Change `parentRefs` to `web-gw2`: `Accepted=False`, reason `NoMatchingParent`.
- Change a backend to `web-v3`: `ResolvedRefs=False`, reason `BackendNotFound`.
- Move the HTTPRoute to namespace `frontend` (with `namespace: d9` in parentRefs): `NotAllowedByListeners`. Fix it with `allowedRoutes.namespaces.from: All` on the listener.
- Keep it in frontend, but point a backend at `web-v1` with `namespace: d9`: `RefNotPermitted`. Fix it with a ReferenceGrant **in d9** allowing HTTPRoutes from frontend.

Last, rewrite the `hosts` Ingress from step 6 as an HTTPRoute by hand, using the mapping table. It's a likely exam task.

**Step 8: CNI files (task 7).** On cp1 and on worker1:

```bash
ls -l /etc/cni/net.d/ ; sudo cat /etc/cni/net.d/10-calico.conflist | head -30   # "type": "calico", ipam "calico-ipam"
ls /opt/cni/bin/
sudo grep -n -A3 'cni' /etc/containerd/config.toml                               # conf_dir / bin_dir (if set)
k get installation default -o jsonpath='{.spec.calicoNetwork.ipPools}{"\n"}'     # cidr, encapsulation
ip route | grep -E 'vxlan|tunl|cali' | head
```

Optional, worker1 only, if you have a snapshot or 15 minutes to spare: `sudo mv /etc/cni/net.d/10-calico.conflist /root/` and restart the kubelet. Watch `k describe node worker1` for `cni plugin not initialized` after the next sandbox creation (delete a pod on worker1 to force one). Restore the file, and the node goes back to Ready on its own. Calico would also rewrite the file when its node pod restarts. This is one of Day 11's break-and-fix scenarios.

**Cleanup:** `k delete ns d9 frontend backend` and `k config set-context --current --namespace=default`. Keep ingress-nginx and NGF installed. They're useful for Day 11 and the mocks.

---

## 8. Traps that cost points

- **Policy in the wrong namespace.** A NetworkPolicy only selects pods in its own `metadata.namespace`. "Protect the db pods" means the policy goes in the **db's** namespace.
- **A stray dash.** `- podSelector` under a `- namespaceSelector` is OR, not AND. Read `k describe netpol` every time.
- **`podSelector` alone in `from`** means the policy's own namespace, not "anywhere".
- **Forgetting DNS** after denying egress. Every test then fails with `bad address` and you chase the wrong policy.
- **Using the Service port** in `ports:`. Policies see the pod's port after DNAT.
- **`policyTypes` omitted** with only an `ingress` section: egress is **not** restricted. With `policyTypes: [Egress]` and no `egress` section, all egress is denied, including DNS.
- **Expecting a deny rule.** There isn't one. To narrow access, find and change the policy that's too broad (`k get netpol -n <ns>`). Adding another policy can only allow more.
- **Labels on the namespace don't exist.** `namespaceSelector: {matchLabels: {team: web}}` matches nothing if nobody labeled the namespace. Check `k get ns --show-labels`.
- **Testing with ping.** NetworkPolicy rules here are TCP/UDP-port based, and ICMP behaves differently per CNI. Use `wget -T 2` or `nc -w 2`.
- **Ingress without `ingressClassName`** and no default class: ADDRESS stays empty, nothing routes.
- **Ingress `pathType` missing** (it's required), or `Exact` when the task meant "everything under /api". `k create ingress` without a trailing `*` gives **Exact**.
- **Ingress TLS Secret in another namespace.** It must be in the Ingress's namespace and be type `kubernetes.io/tls`.
- **HTTPRoute `parentRefs` without `namespace:`** when the Gateway is in another namespace. The default is the route's own namespace.
- **Cross-namespace `backendRefs` without a ReferenceGrant**, or with the grant in the wrong namespace. It goes in the **target** (Service's) namespace.
- **Weights read as percentages.** They're relative. `weight: 8` and `weight: 2` is also 80/20. If the task says 80/20, write 80 and 20 so the grader sees exactly that.
- **Hostname mismatch** between a listener's `hostname` and the route's `hostnames`. The route attaches to no listener: `NoMatchingListenerHostname`.
- **Not checking route status.** `k apply` succeeds for a route that never attaches. Read `Accepted` and `ResolvedRefs`.
- **Picking Flannel** when the task requires NetworkPolicy support. And not checking that the CNI manifest's pod CIDR matches the cluster's `podSubnet`.

---

## 9. Self-check: answer before you open the lab

1. Namespace `backend` has no policies. You apply one policy that selects `app=db` with `policyTypes: [Ingress]` and a single rule allowing `role=api` from namespace `frontend`. What can now reach the other pods in `backend` (not db)? And what can the db pods reach outbound?
2. Write the `from` block that means "pods labeled `role=api` in namespace `frontend`", then the version that accidentally means something broader. Say in words what the broader one allows.
3. You applied default-deny egress in `frontend`, plus an egress rule allowing `role=api` → `app=db` in `backend` on 5432. `wget db.backend:5432` from the api pod prints `bad address 'db.backend'`. What's missing, and what does the fix look like?
4. Service `db` has `port: 80, targetPort: 5432`. Which port goes in the NetworkPolicy, and why?
5. An Ingress shows an empty ADDRESS column a minute after you created it. Name the two most likely causes and the command that tells them apart.
6. Write an HTTPRoute that attaches to Gateway `main-gw` in namespace `infra`, for host `shop.example.com`, and sends 90% of traffic to `shop-v1:8080` and 10% to `shop-v2:8080`. What else must be true for the route to be Accepted?
7. An HTTPRoute in namespace `apps` points at Service `auth` in namespace `security`. Its status shows `ResolvedRefs=False, reason RefNotPermitted`. What do you create, in which namespace?
8. A newly joined node is `NotReady` with `cni plugin not initialized`. Which two directories do you check on that node, and what do you expect to find there with Calico?

<details>
<summary>Answers</summary>

1. Everything still reaches the other pods: only `app=db` pods are selected, so only they are isolated, and only for ingress. The db pods can still reach anything outbound. The policy has no Egress type, so egress isn't isolated. Replies to allowed inbound connections work regardless, because enforcement is stateful.
2. AND (correct), one peer:
   ```yaml
   - from:
     - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: frontend}}
       podSelector: {matchLabels: {role: api}}
   ```
   OR (broader), two peers, with a dash before `podSelector`. It allows **every pod in `frontend`**, plus `role=api` pods in the **policy's own** namespace (`backend`).
3. DNS egress. With egress denied, the pod can't reach kube-dns, so the name never resolves. Add an egress rule (in `frontend`, `podSelector: {}` or at least `role=api`) to `namespaceSelector kubernetes.io/metadata.name=kube-system` AND `podSelector k8s-app=kube-dns` on UDP 53 and TCP 53.
4. **5432**, the container port. kube-proxy rewrites `ClusterIP:80` to `podIP:5432` before the packet reaches the destination pod's interface, where the CNI checks the policy.
5. (a) No `ingressClassName` and no default IngressClass, so no controller claims it. (b) The controller isn't running, or its Service has no address yet. `k get ing` (CLASS column) plus `k get ingressclass` tells you (a). `k -n ingress-nginx get pods,svc` tells you (b). `k describe ing` shows no events from the controller in both cases. On bare metal with a NodePort controller, ADDRESS may show a node IP or stay empty depending on the controller's config, so test with curl through the nodePort before you call it broken.
6. ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata: {name: shop, namespace: <route ns>}
   spec:
     parentRefs:
     - {name: main-gw, namespace: infra}
     hostnames: ["shop.example.com"]
     rules:
     - backendRefs:
       - {name: shop-v1, port: 8080, weight: 90}
       - {name: shop-v2, port: 8080, weight: 10}
   ```
   For `Accepted`, a listener on `main-gw` must allow routes from the route's namespace (`allowedRoutes.namespaces.from: All`, or `Selector` matching it, since `Same` only covers `infra`). Its `hostname` must also be empty, `shop.example.com`, or a wildcard covering it. For `ResolvedRefs`, both Services must exist in the route's namespace with port 8080.
7. A **ReferenceGrant in `security`** (the target's namespace): `from: [{group: gateway.networking.k8s.io, kind: HTTPRoute, namespace: apps}]`, `to: [{group: "", kind: Service}]` (optionally `name: auth`).
8. `/etc/cni/net.d/` should contain `10-calico.conflist` (plus `calico-kubeconfig`), and `/opt/cni/bin/` should contain the `calico`, `calico-ipam`, `portmap`, `loopback` etc. binaries. With the operator install, both are written by the `calico-node` pod on that node, so if they're missing, check `k -n calico-system get pods -o wide` for that node's calico-node pod (Pending, CrashLoopBackOff, or an image pull failure).

</details>

---

**Done when:** every [Day 9 acceptance criterion](../stories/day-09-netpol-ingress-gateway.md#acceptance-criteria) passes: the `role=api`-from-`frontend` → db:5432 policy written and verified (one allowed, one denied test) in < 6 min; an HTTPRoute splitting 80/20 between two Services, Accepted and verified with a curl loop, in < 6 min; and the AND vs OR selector difference explained without notes. Tomorrow (Day 10, storage) moves from the network to the disk: PVs, PVCs and StorageClasses use the same "an object is only a request until a controller acts on it" pattern you saw today with NetworkPolicy and Gateway.
