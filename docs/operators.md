# Operators: virt, mtv, nncp (luke)

Three operators are installed via OLM subscriptions from the `redhat-operators`
catalog. All use **Automatic** install-plan approval and a pinned `startingCSV`, so
a rebuild restores the exact same versions.

| Operator | Namespace | Subscription | Channel | Pinned CSV |
|----------|-----------|--------------|---------|------------|
| Kubernetes NMState (NNCP) | `openshift-nmstate` | `kubernetes-nmstate-operator` | `stable` | `kubernetes-nmstate-operator.4.22.0-202609090959` |
| OpenShift Virtualization (HCO) | `openshift-cnv` | `kubevirt-hyperconverged` | `stable` | `kubevirt-hyperconverged-operator.v4.22.9` |
| Migration Toolkit for Virtualization | `openshift-mtv` | `mtv-operator` | `release-v2.12` | `mtv-operator.v2.12.8` |

> **Install order matters:** nmstate first (networking must exist before virt),
> then virt, then mtv. Each subscription is independent, but the NNCPs in
> [networking.md](networking.md) depend on the nmstate operator being present.

---

## 1. Kubernetes NMState Operator (NNCP)

Provides the `NodeNetworkConfigurationPolicy` (NNCP) CRD used to configure node
networking declaratively.

```bash
oc create namespace openshift-nmstate --dry-run=client -o yaml | oc apply -f -

cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: kubernetes-nmstate-operator.4.22.0-202609090959
EOF
```

Wait for the CSV to be Succeeded:

```bash
oc get csv -n openshift-nmstate -w
# NAME                                        PHASE
# kubernetes-nmstate-operator.4.22.0-...     Succeeded
```

---

## 2. OpenShift Virtualization (HCO)

The HyperConverged Cluster Operator deploys the full virt stack: KubeVirt, CDI,
SSP, AAQ, HostPathProvisioner, ClusterNetworkAddons, and the migration controller.

```bash
oc create namespace openshift-cnv --dry-run=client -o yaml | oc apply -f -

cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubevirt-hyperconverged
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: kubevirt-hyperconverged-operator.v4.22.9
EOF
```

Verify the operator pods are up:

```bash
oc get pods -n openshift-cnv
# hco-operator, virt-operator, cdi-operator, ssp-operator, aaq-operator,
# cluster-network-addons-operator, hostpath-provisioner-operator,
# kubevirt-migration-operator, virt-platform-autopilot  → all Running
```

### 2a. Creating the HyperConverged instance (data plane)

> **Status on luke (2026-09-19):** the HCO operator is installed and healthy, but
> the `HyperConverged` custom resource has **not** been created yet, so the virt
> data plane (KubeVirt/CDI/SSP pods) is not deployed. The operator logs show
> `HCO not found, skipping reconciliation`. To bring up the full virt stack, apply:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec: {}
EOF
```

Once applied, HCO creates the `KubeVirt`, `CDI`, `SSP`, `AAQ`, and
`NetworkAddonsConfig` resources and their pods appear in `openshift-cnv`. Watch:

```bash
oc get kubevirt cdi ssp -n openshift-cnv -w
```

> **Note:** the CSV's initialization example uses the annotation
> `deployOVS: "false"` (OVN is provided by OpenShift, not a separate OVS deploy).
> The plain `spec: {}` above is the standard minimal instance. Add
> `metadata.annotations.deployOVS: "false"` only if you need to match that exact
> initialization behavior.

---

## 3. Migration Toolkit for Virtualization (MTV)

The MTV operator (Forklift) provides vSphere→OpenShift VM migration. It is
namespace-scoped; the `openshift-mtv` namespace holds the forklift pods.

```bash
oc create namespace openshift-mtv --dry-run=client -o yaml | oc apply -f -

cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: mtv-operator
  namespace: openshift-mtv
spec:
  channel: release-v2.12
  installPlanApproval: Automatic
  name: mtv-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: mtv-operator.v2.12.8
EOF
```

Verify the forklift pods:

```bash
oc get pods -n openshift-mtv
# forklift-api, forklift-controller, forklift-operator, forklift-ova-proxy,
# forklift-ui-plugin, forklift-validation, forklift-volume-populator-controller
# → all Running
```

> MTV migration targets (vCenter credentials, MigCluster) are configured separately
> per migration job — see the `openshift_mtv` repo for the go-nfc/Belay plugin and
> vSphere provider manifests. The operator install itself is just the subscription
> above.

---

## 4. Verify all three

```bash
# Subscriptions at latest known CSV
oc get subscription -A | grep -E "nmstate|hyperconverged|mtv"

# CSVs Succeeded
oc get csv -n openshift-nmstate \
  -n openshift-cnv \
  -n openshift-mtv 2>/dev/null || \
for ns in openshift-nmstate openshift-cnv openshift-mtv; do
  echo "== $ns =="; oc get csv -n $ns --no-headers | awk '{print $1, $NF}'
done

# Operator pods healthy
for ns in openshift-nmstate openshift-cnv openshift-mtv; do
  echo "== $ns =="; oc get pods -n $ns --no-headers | grep -v Running || echo "  all running"
done
```

## Rebuild note

All three subscriptions are idempotent — re-applying the same `startingCSV` after a
rebuild restores the identical operator versions. The only non-idempotent piece is
the `HyperConverged` instance (§2a), which must be (re)created to deploy the virt
data plane.
