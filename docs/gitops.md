# GitOps (Argo CD) — luke day-2

Drives the day-2 operator installs through **OpenShift GitOps (Argo CD)** so a
rebuild converges by syncing git, not by re-running `oc apply`/`helm` by hand.
The manifests and ArgoCD `Application`s live in
[`syangsao/openshift_gitops`](https://github.com/syangsao/openshift_gitops) — this
page is the luke-specific wiring: which apps cover what, what to add, and how to
bootstrap after a rebuild.

> **Relationship to the other docs:** [operators.md](operators.md),
> [networking.md](networking.md), and [nfs-csi.md](nfs-csi.md) are the *manual*
> paths (the ones that were run on the 2026-09-19 rebuild). This page is the
> *GitOps* path for the next rebuild. Both converge to the same state; GitOps adds
> continuous reconciliation (`selfHeal`) and drift detection on top.

## What Argo CD covers (and what it can't)

| Day-2 item | ArgoCD app (in `openshift_gitops`) | Source path | Notes |
|------------|-----------------------------------|-------------|-------|
| nmstate operator | `nmstate-operator` | `operators/nmstate` | ns + OperatorGroup + Subscription |
| NMState instance (unlocks NNCP CRDs) | `nmstate-instance` | `operators/nmstate-instance` | must sync *after* the operator is up |
| virt/HCO operator | `virt-operator` | `operators/virt` | ns + OperatorGroup + Subscription |
| HyperConverged instance (data plane) | `virt-instance` | `operators/virt-instance` | deploys KubeVirt/CDI/SSP/AAQ |
| NFS CSI driver | `nfs-csi` | `operators/nfs-csi` | kustomize: controller + node + snapshotter + RBAC |
| MTV operator (Forklift) | `mtv-operator` | `operators/mtv` | ns + OperatorGroup + Subscription (channel `release-v2.12`) |
| NNCPs + NADs (node networking) | `luke-networking` | *(arbiter repo)* `luke-network-config.yaml` | cluster-scoped; sync *after* `nmstate-instance` so the NNCP CRD exists |

**Still manual (not ArgoCD-managed):**

- **TLS certificates** — see [§7](#7-certs-gitops-friendly-or-not). Secrets +
  IngressController/APIServer patches; doable via ArgoCD but the ACME issuance
  stays external.

> **Namespace note:** the `virt-operator` app deploys into
> `openshift-virtualization-operator` (the upstream KubeVirt namespace), whereas
> the manual path in [operators.md](operators.md) used `openshift-cnv`. Both are
> valid; pick one and keep it consistent so the Subscription and the
> HyperConverged instance live in the same namespace.

## 1. Bootstrap: install the GitOps operator (manual, once per rebuild)

Argo CD can't manage its own installation — this is the only manual step after a
rebuild. Use the script from `openshift_gitops`:

```bash
git clone git@github.com:syangsao/openshift_gitops.git
cd openshift_gitops
./scripts/install-gitops-operator.sh
```

Or manually (see `guides/01-install-gitops-operator.md`):

```bash
oc create namespace openshift-gitops-operator
oc apply -f operators/gitops/operator-group.yaml
oc apply -f operators/gitops/subscription.yaml
# wait for CSV Succeeded + ArgoCD pods in openshift-gitops
oc get csv -n openshift-gitops-operator -w
```

Verify: `./scripts/check-gitops-operator.sh` → **FULLY INSTALLED**.

## 2. Grant RBAC to the Argo CD controller

The application-controller ServiceAccount needs cluster-wide rights to create the
operator namespaces, OperatorGroups, Subscriptions, and CRs. The repo ships
per-operator ClusterRole/Bindings (`operators/argocd-applications/*-rbac.yaml`).
Apply them all:

```bash
for f in operators/argocd-applications/*-rbac.yaml; do oc apply -f "$f"; done
```

> For a quick rebuild you can instead bind the controller SA to `cluster-admin`
> (what the manual path effectively had). Least-privilege is the per-resource
> bindings above.

## 3. Create the Argo CD Applications

Apply every app manifest. Each points at the `openshift_gitops` repo, syncs
automated with `prune` + `selfHeal`:

```bash
for f in operators/argocd-applications/*-app.yaml; do oc apply -f "$f"; done
```

This creates (in `openshift-gitops`): `nmstate-operator`, `nmstate-instance`,
`virt-operator`, `virt-instance`, `nfs-csi`.

**Sync order matters.** The `*-instance` apps depend on their operator being up:

1. `nmstate-operator` → wait for CSV `Succeeded`
2. `nmstate-instance` (now the NNCP CRD exists)
3. `virt-operator` → wait for CSV `Succeeded`
4. `virt-instance` (deploys the virt data plane)
5. `nfs-csi`

Argo CD retries out-of-order apps automatically, but to avoid a flood of
`CrashLoopBackOff`/`OutOfSync` noise on a fresh cluster, sync the operators first:

```bash
alias argocd='argocd --grpc-web --grpc-web-root-path /'
HOST=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')
PW=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o json | jq -r '.data["admin.password"]' | base64 -d)
argocd login "$HOST" --username admin --password "$PW" --grpc-web-root-path / --skip-test-tls

# operators first
for a in nmstate-operator virt-operator nfs-csi; do argocd app sync "$a"; done
# then instances, once their operators are Succeeded
for a in nmstate-instance virt-instance; do argocd app sync "$a"; done
```

## 4. Verify

```bash
argocd app get all          # every app Synced + Healthy
oc get csv -n openshift-nmstate -n openshift-cnv --no-headers   # Succeeded
oc get kubevirt cdi ssp -n openshift-cnv --no-headers           # Deployed
oc get pods -n csi-driver-nfs --no-headers                       # Running
```

## 5. MTV operator app

`operators/mtv/` in `openshift_gitops` (namespace, OperatorGroup, Subscription —
same shape as `operators/nmstate/`) plus `argocd-applications/mtv-operator-app.yaml`
and `mtv-rbac.yaml`. The Subscription pins `channel: release-v2.12`,
`startingCSV: mtv-operator.v2.12.8` (see [operators.md §3](operators.md#3-migration-toolkit-for-virtualization-mtv)).
The OperatorGroup is included — required on a fresh rebuild, see
[operators.md](operators.md).

## 6. Networking (NNCP + NAD) app

`argocd-applications/luke-networking-app.yaml` (+ `luke-networking-rbac.yaml`) in
`openshift_gitops` sources the canonical NNCP + NAD set from the **arbiter** repo
where the artifact lives:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: luke-networking, namespace: openshift-gitops }
spec:
  project: default
  source:
    repoURL: 'https://github.com/syangsao/openshift_arbiter.git'
    targetRevision: main
    path: luke-network-config.yaml   # the 5 NNCPs + 2 NADs
  destination: { server: 'https://kubernetes.default.svc' }   # cluster-scoped
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

> **Prerequisite:** the `NMState` instance (the `nmstate-instance` app) must be
> synced first so the NNCP CRD exists — otherwise this app fails with `no matches
> for kind "NodeNetworkConfigurationPolicy"`. The `mtv-test` namespace (needed by
> the macvlan NAD) is created by the same file.

## 7. Certs — GitOps-friendly or not?

The cert *secrets* and the IngressController/APIServer patches
([certs.md](certs.md)) are declarative and *can* be ArgoCD-managed: commit the two
TLS secrets (or use an external-secrets source) plus a `defaultCertificate` /
`namedCertificates` patch. But the **ACME issuance** (`acme.sh --issue --dns
dns_easydns`) is inherently external — it runs on a host with EasyDNS credentials,
not in-cluster. So the practical split is:

- **GitOps:** the secrets + controller patches (drift-heals if someone reverts them).
- **Manual/cron:** `acme.sh` renewal (~90-day ZeroSSL certs), then push the new
  material to the repo so Argo CD syncs it.

For a rebuild, the manual path in [certs.md](certs.md) is simpler and was what ran
on 2026-09-19; move it to GitOps only if you want continuous cert drift detection.

## Rebuild checklist (GitOps path)

```bash
# 0) kubeconfig active for luke
oc whoami                          # → admin/luke

# 1) install GitOps operator (manual bootstrap)
cd openshift_gitops && ./scripts/install-gitops-operator.sh

# 2) RBAC + apps
for f in operators/argocd-applications/*-rbac.yaml; do oc apply -f "$f"; done
for f in operators/argocd-applications/*-app.yaml;   do oc apply -f "$f"; done

# 3) sync operators (incl. mtv), then instances, then networking (see §3)
#    order: nmstate-operator → virt-operator → mtv-operator → nfs-csi
#           → nmstate-instance → virt-instance → luke-networking
# 4) certs: manual per certs.md, or GitOps-managed secrets (§7)
# 5) verify: argocd app get all → all Synced+Healthy
```
