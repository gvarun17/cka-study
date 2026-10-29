# kubeadm upgrade (one minor version)

Target: control plane in under 10 minutes, each worker in under 6.

## Control-plane node

```bash
# 1. Point apt at the new minor version, then find the exact patch version
sudo sed -i 's/v1\.OLD/v1.NEW/' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm

# 2. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='<ver>-*'
sudo apt-mark hold kubeadm
kubeadm version

# 3. Plan and apply
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v<ver>

# 4. Drain, then upgrade kubelet + kubectl
kubectl drain <cp-node> --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='<ver>-*' kubectl='<ver>-*'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# 5. Bring it back
kubectl uncordon <cp-node>
```

## Each worker node

```bash
kubectl drain <worker> --ignore-daemonsets        # from the control plane
ssh <worker>
  # update the apt repo as in step 1, then:
  sudo apt-mark unhold kubeadm && sudo apt-get install -y kubeadm='<ver>-*' && sudo apt-mark hold kubeadm
  sudo kubeadm upgrade node                       # NOT apply
  sudo apt-mark unhold kubelet kubectl
  sudo apt-get install -y kubelet='<ver>-*' kubectl='<ver>-*'
  sudo apt-mark hold kubelet kubectl
  sudo systemctl daemon-reload && sudo systemctl restart kubelet
exit                                              # don't forget
kubectl uncordon <worker>
```

## Common mistakes

- Upgrading to the wrong patch version. Use exactly the version the question names.
- Running `kubeadm upgrade apply` on a worker
- Forgetting `daemon-reload` and the kubelet restart, so the node keeps reporting the old version
- Forgetting `exit` after SSH, so the next question runs on the wrong host
