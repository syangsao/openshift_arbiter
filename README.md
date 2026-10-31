# OpenShift Arbiter — Day-2 Operations Runbook

Day-2 setup for the **luke** cluster (`*.luke.syangsao.net`) after the initial
bare-metal install is complete. This page doubles as a runnable runbook: every step
lists the exact command, and the network config is committed as the canonical
artifact so the whole thing can be re-applied idempotently after a rebuild.

> **Purpose:** if this cluster is rebuilt, an agent (or human) revisits this page and
> executes the steps below to restore the full day-2 state: API/ingress certs,
> operators (nmstate / virt / mtv), NFS CSI storage, and the node network
> configuration.

## Target environment

| Item | Value |
|------|-------|
| Cluster name | `luke` |
| OpenShift version | 4.22.13 |
| Nodes | `control01`, `control02` (master+worker), `arbiter` (arbiter role) |
| Internal network | `192.168.40.0/24` |
| API domain | `api.luke.syangsao.net` |
| Ingress domain | `apps.luke.syangsao.net` |
| VM data VLAN | 60 (`192.168.60.0/24`) |
| NFS VLANs | 80 (`192.168.80.0/24`), 90 (`192.168.90.0/24`) |
| Cert issuer | ZeroSSL (ACME, DNS-01 via EasyDNS) |

## Table of contents

1. [API & ingress TLS certificates](docs/certs.md)
2. [Operators: virt, mtv, nncp](docs/operators.md)
3. [Node network configuration (NNCP)](docs/networking.md)
4. [NFS CSI driver (RWX storage + snapshots)](docs/nfs-csi.md)
5. [GitOps (Argo CD) — install all of the above via git](docs/gitops.md)

## Quick reference — restore after rebuild

Order matters: operators before their instances, the NMState instance before any
NNCP, and NFS CSI only after `bond0.80` exists (the NNCPs create it).

```bash
# 0) kubeconfig for luke is already the active context
oc whoami            # → admin/luke

# 1) Certificates (see docs/certs.md) — acme.sh issue + apply, both surfaces
ansible-playbook -i inventory.ini update_apps_cert.yml \
  -e apps_domain=apps.luke.syangsao.net -e acme_home=~/.acme.sh
# + API named cert: see docs/certs.md §2

# 2) Operators (see docs/operators.md) — in this order:
#    a) nmstate subscription → wait CSV Succeeded
#    b) NMState instance `cluster` (unlocks the NNCP CRD)          ← operators.md §1a
#    c) virt/HCO subscription → wait CSV Succeeded
#    d) HyperConverged instance (deploys KubeVirt/CDI/SSP)         ← operators.md §2a
#    e) CDI worker pod resources patch via HCO                     ← operators.md §2b
#    f) mtv subscription → wait CSV Succeeded

# 3) Network config (the canonical artifact in this repo) — after the NMState instance
oc create namespace mtv-test --dry-run=client -o yaml | oc apply -f -
oc apply -f luke-network-config.yaml
oc get nodenetworkconfigurationpolicy   # all Available: True

# 4) NFS CSI driver (see docs/nfs-csi.md) — after bond0.80 exists
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --version 4.13.4 \
  --create-namespace --namespace csi-driver-nfs \
  --set controller.runOnControlPlane=true --set controller.replicas=2 \
  --set controller.strategyType=RollingUpdate \
  --set externalSnapshotter.enabled=true \
  --set externalSnapshotter.customResourceDefinitions.enabled=false
oc adm policy add-scc-to-user privileged -z csi-nfs-node-sa -n csi-driver-nfs
oc adm policy add-scc-to-user privileged -z csi-nfs-controller-sa -n csi-driver-nfs
oc apply -f storageclass.yaml
oc apply -f snapshotclass.yaml
```

## Files

| Path | Purpose |
|------|---------|
| [`luke-network-config.yaml`](luke-network-config.yaml) | Canonical NNCP + NAD set (4 NNCPs + 2 NADs) — the single source of truth for node networking. Apply with `oc apply -f`. |
| [`docs/certs.md`](docs/certs.md) | API server named cert + default ingress (apps) wildcard cert, ACME/ZeroSSL. |
| [`docs/operators.md`](docs/operators.md) | Install nmstate, virt (HCO), and mtv operators via OLM subscriptions. |
| [`docs/nfs-csi.md`](docs/nfs-csi.md) | csi-driver-nfs install (Helm, Option 1), StorageClass + SnapshotClass, PVC/snapshot round-trip. |
| [`docs/gitops.md`](docs/gitops.md) | Drive the day-2 operator installs through Argo CD (GitOps) — apps, RBAC, sync order, and what's still missing. |
| [`storageclass.yaml`](storageclass.yaml) | `nfs-csi` default StorageClass → NAS `nas01-80:/nfsshare/csidriver`. |
| [`snapshotclass.yaml`](snapshotclass.yaml) | `csi-nfs-snapclass` VolumeSnapshotClass for NFS snapshots. |

## Prerequisites for this runbook

- `oc` CLI with a kubeconfig that has **cluster-admin** on luke (context `admin`).
- `helm` >= 3.14 (NFS CSI install — see [docs/nfs-csi.md](docs/nfs-csi.md) §2).
- `ansible-core` >= 2.14 and `acme.sh` configured for the EasyDNS DNS provider
  (for certificate issuance/renewal).
- Network baseline present from day-0 install: each node has a working `bond0`
  (the two NICs bonded) carrying VLANs 60/80/90, and the cluster network is up.
