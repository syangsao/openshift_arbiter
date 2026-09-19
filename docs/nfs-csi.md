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
| NFS server | `nas01-80.syangsao.lab` (`192.168.80.20`, VLAN 80 / `bond0.80`) |
| Export path | `/nfsshare/csidriver` |
| StorageClass | `nfs-csi` (default) |
| VolumeSnapshotClass | `csi-nfs-snapclass` |

Running images confirmed: `nfsplugin:v4.13.4`, `csi-provisioner:v6.3.0`,
`csi-snapshotter:v8.6.0`, `snapshot-controller:v8.6.0`.

---

## 1. NFS server (prerequisite)

The NFS server is the **NAS** at `nas01-80.syangsao.lab` (`192.168.80.20`), reachable
on VLAN 80 — the `bond0.80` interface created by the NNCPs in
[networking.md](networking.md). The export is `/nfsshare/csidriver`.

> This NAS is an existing external server; it is **not** set up on a cluster node.
> (An earlier attempt used a control01 export, but the intended target is the NAS.)

Verify reachability from a cluster node:

```bash
# From any node that has bond0.80 (control01/control02)
getent hosts nas01-80.syangsao.lab        # → 192.168.80.20
mkdir -p /tmp/nas-test
mount -t nfs nas01-80.syangsao.lab:/nfsshare/csidriver /tmp/nas-test   # MOUNT OK
touch /tmp/nas-test/.w && rm /tmp/nas-test/.w && umount /tmp/nas-test  # WRITE OK
```

> The default NFSv3 mount works. If a share is v4-only, add `mountOptions: ["nfsvers=4.1"]`
> to the StorageClass (see the `-2`/`-3` variants in `~/mtv/nfs-csi/`).

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
> latest, **4.13.4** (the same flags). To track the guide exactly use 4.11.0;
> otherwise stay on the latest and re-run `helm repo update` + `helm upgrade`.

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

References the `nfs.csi.k8s.io` provisioner and points at the NAS export. Marked as
the **default** StorageClass. This is the file from `~/mtv/nfs-csi/storageclass.yaml`.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # must be a string, not bool
provisioner: nfs.csi.k8s.io
parameters:
  server: nas01-80.syangsao.lab                           # NAS on VLAN 80 (192.168.80.20)
  share: /nfsshare/csidriver                              # NAS export path
  subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}-${pv.metadata.name}
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: True
```

```bash
oc apply -f storageclass.yaml
oc get storageclass        # nfs-csi (default)
```

> **Pitfall 1 — annotation type:** the `is-default-class` value must be the string
> `"true"`. A bare YAML boolean makes `oc apply` fail with
> `json: cannot unmarshal bool into ... annotations of type string`.
>
> **Pitfall 2 — parameters are immutable:** Kubernetes forbids updating a StorageClass's
> `parameters` in place (`updates to parameters are forbidden`). To change the server/share,
> delete the StorageClass (and any PVCs/PVs using it) and re-apply. This is what happened
> when switching from a control01 export to the NAS.

> **Other shares** (not deployed on luke by default): `~/mtv/nfs-csi/` also holds
> `storageclass.yaml-2` (`nfs-csidriver2` → `nas01-80:/csidriver2`, NFS 4.1) and
> `storageclass.yaml-3` (`nfs-csidriver3` → `mirror.syangsao.net:/csidriver`, non-default,
> NFS 4.1). Only the default `nfs-csi` was applied here.

## 7. VolumeSnapshotClass — [`snapshotclass.yaml`](../snapshotclass.yaml)

Required to create snapshots. This is the file from `~/mtv/nfs-csi/snapshotclass.yaml`.

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
# 1) Write a file via a pod mounting the PVC (image: ubi9/ubi-minimal) → /data/test.txt

# 2) Snapshot the PVC
cat <<'EOF' | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: test-nfs-snap, namespace: csi-driver-nfs }
spec:
  volumeSnapshotClassName: csi-nfs-snapclass
  source: { persistentVolumeClaimName: test-nfs }
EOF
oc get volumesnapshot -n csi-driver-nfs   # READYTOUSE=true

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

**Result on luke (NAS-backed):** PVC `test-nfs` Bound; snapshot `test-nfs-snap`
`readyToUse=true`; restored PVC `test-nfs-restored` returned the identical file
contents. Full round-trip passed against `nas01-80:/nfsshare/csidriver`.

---

## Rebuild notes

- **NFS server** is external (the NAS) — nothing to recreate on cluster nodes. After a
  rebuild, just confirm `nas01-80.syangsao.lab` resolves and the `/nfsshare/csidriver`
  export is reachable from the nodes that carry `bond0.80`.
- **Helm release** is idempotent — re-run the `helm install` (or `helm upgrade`) with
  the same flags to restore the driver. The SCC grants (§5) must be re-applied if the
  namespace/ServiceAccounts are recreated.
- **StorageClass / SnapshotClass** (`oc apply -f`) are idempotent. Remember parameters
  are immutable — delete + re-apply to change server/share.
- The test PVCs/snapshot from §8 can be deleted after a rebuild; they exist only as
  proof of the round-trip.

## Files in this repo

| Path | Purpose |
|------|---------|
| [`storageclass.yaml`](../storageclass.yaml) | `nfs-csi` default StorageClass → NAS `nas01-80:/nfsshare/csidriver` |
| [`snapshotclass.yaml`](../snapshotclass.yaml) | `csi-nfs-snapclass` VolumeSnapshotClass |
