# etcd backup and restore

Target: snapshot in under 4 minutes, full restore in under 8.

## 1. Read the cert paths from the manifest

```bash
grep -E 'cert|key|listen-client' /etc/kubernetes/manifests/etcd.yaml
```

The exam may use non-default paths, so always read them; never type them from memory.

## 2. Snapshot

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db

etcdutl snapshot status /opt/etcd-backup.db -w table
```

## 3. Restore into a new directory

```bash
etcdutl snapshot restore /opt/etcd-backup.db --data-dir /var/lib/etcd-restore
```

Newer etcd versions moved restore to `etcdutl`. Use `etcdctl snapshot restore` only if `etcdutl` isn't installed.

## 4. Point etcd at the restored data

Edit `/etc/kubernetes/manifests/etcd.yaml` and change the `etcd-data` **hostPath** volume:

```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-restore    # was /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
```

## 5. Wait and verify

```bash
watch crictl ps            # etcd and kube-apiserver restart in 1–2 minutes
k get pods -A
```

## Common mistakes

- Restoring into the existing `/var/lib/etcd` directory instead of a new one
- Changing the `--data-dir` flag but not the hostPath volume (or the reverse). The **hostPath** is what matters for the container.
- Running the commands on the wrong host after an `ssh`
