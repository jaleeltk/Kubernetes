# PVC Migration: Ceph RBD (RWO) → NetApp NFS via Trident (RWX)

Standard operating procedure to move an application's data from a Ceph RBD PVC to a NetApp NFS PVC provisioned by Trident, so several pods can share the same volume.

## Repository layout

```
.
├── README.md                              # this SOP
├── manifests/
│   ├── 00-premigration-snapshot.yaml      # optional Ceph snapshot before migrating
│   ├── 01-data-netapp-pvc.yaml            # target RWX PVC on NetApp
│   └── 02-copy-helper-pod.yaml            # helper pod mounting both PVCs
└── tools/
    └── Dockerfile.rsync                   # rsync image for air-gapped clusters
```

## 1. Purpose and scope

- **Applies to:** Deployments that store shared files (uploads, reports, models, configs).
- **Does not apply to:** databases (PostgreSQL, MySQL, MongoDB, etc.). Keep those on Ceph block storage.
- **Downtime:** the application is stopped from Step 2 to Step 7. Agree on a maintenance window first.
- **Example names:** namespace `myapp-ns`, Deployment `myapp`, source PVC `data-ceph`, target PVC `data-netapp`, target StorageClass `netapp-nas`. Replace with real values.

## 2. Prerequisites and pre-checks

- [ ] Change request approved and application owner informed.
- [ ] Application is not a database.
- [ ] NetApp Trident StorageClass exists: `kubectl get sc`
- [ ] NetApp export policy allows the worker nodes, with superuser (root) access so file owners are kept during the copy.
- [ ] Note the source PVC size and used space:

```bash
kubectl get pvc data-ceph -n myapp-ns
kubectl exec -n myapp-ns deploy/myapp -- df -h /data
```

- [ ] Protect the source volume:

```bash
OLDPV=$(kubectl get pvc data-ceph -n myapp-ns -o jsonpath='{.spec.volumeName}')
kubectl patch pv $OLDPV -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

- [ ] Take a Ceph snapshot of the source PVC (if VolumeSnapshot is set up) and wait for `READYTOUSE=true`:

```bash
kubectl apply -f manifests/00-premigration-snapshot.yaml
kubectl get volumesnapshot -n myapp-ns
```

- [ ] Check internet access from the cluster (decides the copy method):

```bash
kubectl run nettest -n myapp-ns --rm -it --image=alpine:3.20 --restart=Never -- \
  sh -c "apk add --no-cache rsync && echo INTERNET-OK"
```

If the image pull fails, use the internal registry path for the image. If `INTERNET-OK` does not print, the cluster has no internet: use Option A, B, or C2.

## 3. Choosing a copy method

Use `cp -a` by default: it works everywhere with no install. Use rsync for large volumes or when the copy must be re-run.

| Option | Command | Needs internet | Best for |
| --- | --- | --- | --- |
| A. cp | `cp -a /old/. /new/` | No | Small and medium data, one-time copy |
| B. tar | `tar cf - . \| (cd /new && tar xpf -)` | No | Many small files |
| C1. rsync (public) | `apk add rsync` + `rsync -aH --numeric-ids /old/ /new/` | Yes | Large data, re-runnable copy |
| C2. rsync (internal image) | `rsync -aH --numeric-ids /old/ /new/` | No | Large data in air-gapped clusters |

All options keep file permissions, owners, and timestamps when run as root.

## 4. Procedure: prepare

### Step 1. Create the NetApp NFS PVC (RWX)

```bash
kubectl apply -f manifests/01-data-netapp-pvc.yaml
kubectl get pvc -n myapp-ns
```

**Expected:** `data-netapp` is `Bound`, access mode `RWX`. Size it at least as large as the used space on the source.

### Step 2. Stop the application

```bash
kubectl scale deploy myapp -n myapp-ns --replicas=0
kubectl get deploy,pods -n myapp-ns
```

**Expected:** `myapp 0/0`, no pods. Downtime starts here.

### Step 3. Create the helper pod with both PVCs mounted

Set `image` in the manifest according to section 3.

```bash
kubectl apply -f manifests/02-copy-helper-pod.yaml
kubectl get pod copy -n myapp-ns
```

**Expected:** `copy` is `Running`. The source is mounted read-only.

## 5. Procedure: copy the data

### Step 4. Log in to the helper pod and copy

```bash
kubectl exec -it copy -n myapp-ns -- sh
```

Check both sides:

```sh
ls -lR /old | head -50
ls -lR /new
```

Run **one** option:

**Option A: cp (no internet)**

```sh
cp -a /old/. /new/
```

**Option B: tar (no internet)**

```sh
cd /old
tar cf - . | (cd /new && tar xpf -)
```

**Option C1: rsync with internet**

```sh
apk add --no-cache rsync
rsync -aH --numeric-ids /old/ /new/
```

**Option C2: rsync without internet**

Build `tools/Dockerfile.rsync` once where internet is available, push it to the internal registry, and use it as the helper pod image. Then:

```sh
rsync -aH --numeric-ids /old/ /new/
```

rsync flags: `-a` keeps permissions/owners/timestamps/symlinks, `-H` keeps hard links, `--numeric-ids` copies UID/GID numbers. The trailing `/` on `/old/` copies the contents, not the folder.

## 6. Procedure: verify and switch over

### Step 5. Verify the copy, then delete the helper pod

Inside the helper pod:

```sh
find /old -type f | wc -l
find /new -type f | wc -l
du -sh /old /new
(cd /old && find . -type f -exec md5sum {} + | sort -k2) > /tmp/old.md5
(cd /new && find . -type f -exec md5sum {} + | sort -k2) > /tmp/new.md5
diff /tmp/old.md5 /tmp/new.md5 && echo VERIFY-OK
ls -ln /new | head
exit
```

**Expected:** same file count, `VERIFY-OK`, same owners (UID/GID). `du` sizes can differ slightly between Ceph and NFS.

```bash
kubectl delete pod copy -n myapp-ns
```

### Step 6. Point the Deployment to the NetApp PVC

Change the PVC name (`claimName`), not the StorageClass:

```bash
kubectl patch deploy myapp -n myapp-ns --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/0/persistentVolumeClaim/claimName","value":"data-netapp"}]'

kubectl get deploy myapp -n myapp-ns \
  -o jsonpath='{.spec.template.spec.volumes[*].persistentVolumeClaim.claimName}'; echo
```

**Expected:** `data-netapp`. If the Deployment has several volumes, change index `0` to the right one (or use `kubectl edit`).

### Step 7. Start the application

```bash
kubectl scale deploy myapp -n myapp-ns --replicas=1
kubectl get pods -n myapp-ns
```

**Expected:** pod `Running`. Downtime ends here.

### Step 8. Validate

```bash
kubectl exec -n myapp-ns deploy/myapp -- ls -lR /data | head -30
kubectl exec -n myapp-ns deploy/myapp -- df -h /data
kubectl logs -n myapp-ns deploy/myapp --tail=20
```

**Expected:** files present, `df` shows the NetApp data LIF, no errors in logs. Application owner confirms.

## 7. Rollback and cleanup

Keep the Ceph PVC until the application owner signs off; it is the rollback.

**Rollback:**

```bash
kubectl scale deploy myapp -n myapp-ns --replicas=0
kubectl patch deploy myapp -n myapp-ns --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/0/persistentVolumeClaim/claimName","value":"data-ceph"}]'
kubectl scale deploy myapp -n myapp-ns --replicas=1
```

Data written to NetApp after the switch is not on Ceph; copy it back the same way if needed.

**Cleanup (after sign-off):**

```bash
kubectl delete pvc data-ceph -n myapp-ns
kubectl get pv $OLDPV        # Released (Retain kept the Ceph image)
kubectl delete pv $OLDPV
kubectl delete volumesnapshot data-ceph-premigration -n myapp-ns
```

## 8. Lab test reference

Tested in a home lab (kubeadm, 1 master + 1 worker) with stand-ins for both storage systems.

| Item | Lab test | Production |
| --- | --- | --- |
| Source StorageClass | `local-path` (RWO) | Ceph RBD (RWO) |
| Target StorageClass | `nfs-rwx` (NFS CSI driver, Rocky NFS server) | NetApp Trident `ontap-nas` (RWX) |
| Source / target PVC | `data-old` / `data-new` | `data-ceph` / `data-netapp` |
| Copy method used | rsync with internet (C1) | Depends on cluster internet access |
| `df -h /data` after switch | Rocky NFS server IP | NetApp data LIF IP |
| Result | Data intact, app running on RWX volume | |
