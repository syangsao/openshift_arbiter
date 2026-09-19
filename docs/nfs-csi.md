# NFS CSI Driver (csi-driver-nfs) — luke

Connects the luke cluster to an NFS server for **ReadWriteMany (RWX)** storage via
the [kubernetes-csi/csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs)
provisioner, installed with Helm. Includes the external snapshotter so PVCs can be
snapshotted and restored.

> **Source guide:** [Deploying csi-driver-nfs to OpenShift](https://hackmd.io/@johnsimcall/BJeW2Y5mT)
> (Helm/command-line method, **Option 1**). This page records the exact steps run on
> luke, with the cluster-specific values filled in.

## What was installed (2026-09-19)

| Component | Value |
|-----------|-------|
| Chart | `csi-driver-nfs/csi-driver-nfs` **4.13.4** (latest; guide's Option 1 pins 4.11.0, upgraded to latest) |
| Release / namespace | `csi-driver-nfs` in `csi-driver-nfs` |
| Controller replicas | 2 (HA), `runOnControlPlane=true`, RollingUpdate |
| External snapshotter | enabled; CRDs **not** re-created (OpenShift ships them) |
| NFS server | `control01.syangsao.net` (`192.168.40.26`) |
| Export path | `/var/lib/nfs/exports/openshift` |
| StorageClass | `nfs-csi` (default) |
| VolumeSnapshotClass | `csi-nfs-snapclass` |

Running images confirmed: `nfsplugin:v4.13.4`, `csi-provisioner:v6.3.0`,
`csi-snapshotter:v8.6.0`, `snapshot-controller:v8.6.0`.

---

## 1. NFS server (prerequisite)

luke had no running NFS server, so one was set up on **control01**. The export must
live on a **writable** filesystem — on RHCOS the root `/` is read-only, so use
`/var/lib/nfs/exports/...` (on `/dev/sda4`, writable).

```bash
# Run on control01 (via oc debug node or direct access)
mkdir -p /var/lib/nfs/exports/openshift
chmod 755 /var/lib/nfs/exports/openshift
chown -R nfsnobody:nfsnobody /var/lib/nfs/exports/openshift

# Export to all hosts (internal lab network)
echo "/var/lib/nfs/exports/openshift *(insecure,no_root_squash,async,rw)" > /etc/exports

# Start + enable the NFS server
systemctl enable --now nfs-server
exportfs -av

# Verify
showmount -e localhost
# /var/lib/nfs/exports/openshift  *
```

Verify reachability from another node:

```bash
# From control02
mkdir -p /tmp/nfs-test
mount -t nfs 192.168.40.26:/var/lib/nfs/exports/openshift /tmp/nfs-test
touch /tmp/nfs-test/f && rm /tmp/nfs-test/f && umount /tmp/nfs-test   # WRITE OK
```

> `nfs-utils` was already present on the RHCOS nodes (`rpc.nfsd`, `exportfs`). The
> export path is stable across rebuilds only if you recreate it — see Rebuild notes.

---

## 2. Install Helm CLI

```bash
curl -L -o $HOME/bin/helm https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64
chmod a+x $HOME/bin/helm
helm version --short     # v3.19.x
```

## 3. Add the chart repo

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm search repo -l csi-driver-nfs   # latest = 4.13.4
```

## 4. Install the chart (Option 1, latest version)

The guide's Option 1 installs everything but does **not** try to recreate the
external-snapshot CRDs (OpenShift already has them; re-creating fails with an
ownership-metadata error).

```bash
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --version 4.13.4 \
  --create-namespace \
  --namespace csi-driver-nfs \
  --set controller.runOnControlPlane=true \
  --set controller.replicas=2 \
  --set controller.strategyType=RollingUpdate \
  --set externalSnapshotter.enabled=true \
  --set externalSnapshotter.customResourceDefinitions.enabled=false
```

> **Version note:** the guide pins `--version 4.11.0`. luke was installed with the
> latest, **4.13.4** (the same flags). If you later want to track the guide exactly,
> use 4.11.0; otherwise stay on the latest and re-run `helm repo update` +
> `helm upgrade` periodically.

Expected pods after install (all `Running`):

```
csi-nfs-controller-<x>-<y>   5/5   Running   # x2 replicas
csi-nfs-node-<z>             3/3   Running   # one per node (daemonset)
snapshot-controller-<x>-<y>  1/1   Running
```

## 5. Grant the privileged SCC

The chart's ServiceAccounts need the `privileged` SCC to run on OpenShift (the guide
notes this grants broad permissions; a least-privilege SCC is left as a TODO upstream).

```bash
oc adm policy add-scc-to-user privileged -z csi-nfs-node-sa -n csi-driver-nfs
oc adm policy add-scc-to-user privileged -z csi-nfs-controller-sa -n csi-driver-nfs
```

## 6. StorageClass — [`storageclass.yaml`](../storageclass.yaml)

References the `nfs.csi.k8s.io` provisioner and points at the luke NFS server/export.
Marked as the **default** StorageClass.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # must be a string, not bool
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.40.26                                  # control01 internal IP
  share: /var/lib/nfs/exports/openshift                  # NFS export path
  subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}-${pv.metadata.name}
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: True
```

```bash
oc apply -f storageclass.yaml
# If your NFS server is v4-only, add:  mountOptions: ["nfsvers=4.1"]
oc get storageclass        # nfs-csi (default)
```

> **Pitfall hit on luke:** the `is-default-class` annotation value must be the string
> `"true"`. Writing it as a bare YAML boolean (`true`) makes `oc apply` fail with
> `json: cannot unmarshal bool into ... annotations of type string`.

## 7. VolumeSnapshotClass — [`snapshotclass.yaml`](../snapshotclass.yaml)

Required to create snapshots.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-nfs-snapclass
driver: nfs.csi.k8s.io
deletionPolicy: Delete
```

```bash
oc apply -f snapshotclass.yaml
oc get volumesnapshotclass   # csi-nfs-snapclass
```

> NFS snapshots are **slow** — the driver literally runs `tar -czf` over the share.
> For VM snapshots that exceed the default 5-minute deadline, raise
> `failureDeadline` in the `VirtualMachineSnapshot` (e.g. `1h0m0s`).

## 8. Verify: PVC + snapshot round-trip

### Create a test PVC (RWX)

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: test-nfs, namespace: csi-driver-nfs }
spec:
  accessModes: [ReadWriteMany]
  resources: { requests: { storage: 1Gi } }
  volumeMode: Filesystem
  storageClassName: nfs-csi
EOF
oc get pvc -n csi-driver-nfs     # test-nfs  Bound  RWX  nfs-csi
```

### Write data, snapshot, restore

```bash
# 1) Write a file via a pod mounting the PVC (image: ubi9/ubi-minimal)
#    → /data/test.txt

# 2) Snapshot the PVC
cat <<'EOF' | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: test-nfs-snap, namespace: csi-driver-nfs }
spec:
  volumeSnapshotClassName: csi-nfs-snapclass
  source: { persistentVolumeClaimName: test-nfs }
EOF
oc get volumesnapshot -n csi-driver-nfs   # READYTOUSE=true, RESTORESIZE=170

# 3) Restore into a new PVC via dataSource
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: test-nfs-restored, namespace: csi-driver-nfs }
spec:
  accessModes: [ReadWriteMany]
  resources: { requests: { storage: 1Gi } }
  volumeMode: Filesystem
  storageClassName: nfs-csi
  dataSource:
    name: test-nfs-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOF
# Mount test-nfs-restored in a pod → /data/test.txt matches the original. ✓
```

**Result on luke:** PVC `test-nfs` Bound; snapshot `test-nfs-snap` `readyToUse=true`
(restore size 170 bytes); restored PVC `test-nfs-restored` returned the identical
file contents. Full round-trip passed.

---

## Rebuild notes

- **NFS server** is the only non-cluster-managed piece: recreate
  `/var/lib/nfs/exports/openshift` + `/etc/exports` + `systemctl enable --now
  nfs-server` on control01 after a rebuild (RHCOS wipes `/etc` and `/var` changes).
- **Helm release** is idempotent — re-run the `helm install` (or `helm upgrade`) with
  the same flags to restore the driver. The SCC grants (§5) must be re-applied if the
  namespace/ServiceAccounts are recreated.
- **StorageClass / SnapshotClass** (`oc apply -f`) are idempotent.
- The test PVCs/snapshot from §8 can be deleted after a rebuild; they exist only as
  proof of the round-trip.

## Files in this repo

| Path | Purpose |
|------|---------|
| [`storageclass.yaml`](../storageclass.yaml) | `nfs-csi` default StorageClass → luke NFS server |
| [`snapshotclass.yaml`](../snapshotclass.yaml) | `csi-nfs-snapclass` VolumeSnapshotClass |
