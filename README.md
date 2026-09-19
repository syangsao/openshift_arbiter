# OpenShift Arbiter — Day-2 Operations Runbook

Day-2 setup for the **luke** cluster (`*.luke.syangsao.net`) after the initial
bare-metal install is complete. This page doubles as a runnable runbook: every step
lists the exact command, and the network config is committed as the canonical
artifact so the whole thing can be re-applied idempotently after a rebuild.

> **Purpose:** if this cluster is rebuilt, an agent (or human) revisits this page and
> executes the steps below to restore the full day-2 state: API/ingress certs,
> operators (virt / mtv / nncp), and the node network configuration.

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

## Quick reference — restore after rebuild

```bash
# 0) kubeconfig for luke is already the active context
oc whoami            # → admin/luke

# 1) Certificates (see docs/certs.md)
ansible-playbook -i inventory.ini update_apps_cert.yml \
  -e apps_domain=apps.luke.syangsao.net -e acme_home=~/.acme.sh
# + API named cert: see docs/certs.md §2

# 2) Operators (see docs/operators.md)
oc apply -f - <<'EOF'   # nmstate, virt, mtv subscriptions
...
EOF

# 3) Network config (the canonical artifact in this repo)
oc create namespace mtv-test --dry-run=client -o yaml | oc apply -f -
oc apply -f luke-network-config.yaml
```

## Files

| Path | Purpose |
|------|---------|
| [`luke-network-config.yaml`](luke-network-config.yaml) | Canonical NNCP + NAD set — the single source of truth for node networking. Apply with `oc apply -f`. |
| [`docs/certs.md`](docs/certs.md) | API server named cert + default ingress (apps) wildcard cert, ACME/ZeroSSL. |
| [`docs/operators.md`](docs/operators.md) | Install nmstate, virt (HCO), and mtv operators via OLM subscriptions. |
| [`docs/networking.md`](docs/networking.md) | How the NNCPs are configured: bridges, VLANs, OVN localnet mapping, NADs. |

## Prerequisites for this runbook

- `oc` CLI with a kubeconfig that has **cluster-admin** on luke (context `admin`).
- `ansible-core` >= 2.14 and `acme.sh` configured for the EasyDNS DNS provider
  (for certificate issuance/renewal).
- Network baseline present from day-0 install: each node has a working `bond0`
  (the two NICs bonded) carrying VLANs 60/80/90, and the cluster network is up.
