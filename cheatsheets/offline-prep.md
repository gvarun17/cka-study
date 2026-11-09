# Offline prep: commands to run before disconnecting

Run these once, on a good connection. After that the lab works with no internet (SSH to the VMs is local and never needs it).

On Apple Silicon, `polinux/stress` and `registry.k8s.io/hpa-example` may lack arm64 builds, so they're left out; the [Mac runbook](../labs/mac-runbook.md) uses busybox and python instead.

**This only works for the local Multipass lab** (cp1 + worker1 as VMs on your Mac). An EC2 lab needs internet to reach. On an Apple Silicon Mac the VMs are arm64; every image below is multi-arch, and `apt-get download` picks the right architecture. Two VMs at 2 vCPU / 4 GB need a Mac with 16 GB RAM to stay comfortable.

## 1. Laptop

```bash
cd ~/cka-study && git pull                                  # lessons, stories, cheat sheets
git clone --depth 1 --recurse-submodules --shallow-submodules \
  https://github.com/kubernetes/website ~/k8s-website       # offline docs source (~1–2 GB)
multipass list                                              # cp1 + worker1 Running
```

PSI Secure Browser: download it from your Linux Foundation exam portal and run the system check (manual step).

## 2. cp1: save manifests and charts

```bash
mkdir -p ~/offline && cd ~/offline
curl -L -o ingress-nginx.yaml   https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
curl -L -o metrics-server.yaml  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
curl -L -o local-path.yaml      https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.8.1" > ngf-crds.yaml

helm repo add podinfo https://stefanprodan.github.io/podinfo && helm repo update
helm pull podinfo/podinfo
helm pull oci://quay.io/jetstack/charts/cert-manager
helm pull oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 2.8.1
ls -lh ~/offline
```

## 3. cp1: install the add-ons (skip any already Running)

```bash
k apply -f ~/offline/metrics-server.yaml
k -n kube-system patch deploy metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
k apply -f ~/offline/local-path.yaml
k apply -f ~/offline/ingress-nginx.yaml
helm upgrade --install cert-manager ~/offline/cert-manager-*.tgz -n cert-manager --create-namespace --set crds.enabled=true
k apply -f ~/offline/ngf-crds.yaml
helm upgrade --install ngf ~/offline/nginx-gateway-fabric-2.8.1.tgz -n nginx-gateway --create-namespace --set nginx.service.type=NodePort
```

Create one Gateway so NGF pulls its nginx data-plane image (then delete it):

```bash
k create ns gwtest
cat <<EOF | k apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: test-gw, namespace: gwtest}
spec:
  gatewayClassName: nginx
  listeners: [{name: http, port: 80, protocol: HTTP}]
EOF
sleep 60; k get pods -A | grep -vE 'Running|Completed'     # expect only the header line
```

## 4. cp1: pull every image onto BOTH nodes

```bash
# images the add-ons use right now
k get pods -A -o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | sort -u > ~/offline/images.txt
# images the lessons use
cat >> ~/offline/images.txt <<'EOF'
nginx:1.27
nginx:1.28
httpd:2.4
busybox:1.36
busybox:1.28
busybox:latest
perl:5.40
python:3.12-alpine
nicolaka/netshoot:latest
curlimages/curl:latest
EOF
sort -u -o ~/offline/images.txt ~/offline/images.txt

for i in $(cat ~/offline/images.txt); do sudo crictl pull "$i" >/dev/null && echo "cp1 ok  $i" || echo "cp1 FAIL $i"; done
scp ~/offline/images.txt worker1:/tmp/
ssh worker1 'for i in $(cat /tmp/images.txt); do sudo crictl pull "$i" >/dev/null && echo "worker1 ok  $i" || echo "worker1 FAIL $i"; done'

k delete ns gwtest
```

## 5. Optional: v1.36 upgrade practice (Day 3 again)

```bash
# cp1
sudo sed -i 's#/v1.35/#/v1.36/#' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && apt-cache madison kubeadm | head -3           # pick the newest, e.g. 1.36.2-1.1
V=1.36.2                                                            # <- set from the line above
mkdir -p ~/offline/debs && cd ~/offline/debs
apt-get download kubeadm=$V-1.1 kubelet=$V-1.1 kubectl=$V-1.1 cri-tools kubernetes-cni
dpkg-deb -x kubeadm_${V}*.deb /tmp/kadm
sudo /tmp/kadm/usr/bin/kubeadm config images pull --kubernetes-version v$V
/tmp/kadm/usr/bin/kubeadm config images list --kubernetes-version v$V | grep -E 'kube-proxy|pause' > /tmp/worker-imgs.txt
scp -r ~/offline/debs worker1:~/ && scp /tmp/worker-imgs.txt worker1:/tmp/
ssh worker1 'for i in $(cat /tmp/worker-imgs.txt); do sudo crictl pull $i; done'
```

Offline, install with `sudo apt-mark unhold kubeadm && sudo dpkg -i ~/offline/debs/kubeadm_*.deb` instead of `apt-get install`. The rest of the Day 3 procedure is unchanged.

## 6. Verify, then snapshot

```bash
sudo crictl images | wc -l ; ssh worker1 'sudo crictl images | wc -l'
k run offline-test --image=nginx:1.27 && k get pod offline-test -w   # Running; then: k delete pod offline-test
```

On the laptop:

```bash
multipass stop --all
multipass snapshot cp1 --name offline-ready
multipass snapshot worker1 --name offline-ready
multipass start --all
# restore later: multipass stop cp1 && multipass restore cp1.offline-ready
```

## Offline rule

Always use a tagged image (`nginx:1.27`), never plain `nginx`. An untagged or `:latest` image defaults to `imagePullPolicy: Always`, so the kubelet tries to download it and fails offline. Add `--image-pull-policy=IfNotPresent` to `k run` when you do use one of the `:latest` images above.
