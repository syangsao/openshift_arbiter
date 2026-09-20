# Node Network Configuration (NNCP) — luke

Node networking is managed declaratively with **NodeNetworkConfigurationPolicy**
(NNCP) resources from the nmstate operator, plus two **NetworkAttachmentDefinition**
(NAD) resources for OVN/multus. The canonical set lives in
[`../luke-network-config.yaml`](../luke-network-config.yaml) — apply it as a unit:

> **Prerequisite — NMState instance:** the nmstate operator only registers the
> `NodeNetworkConfigurationPolicy` CRD after an `NMState` custom resource exists.
> On a fresh install the operator pod runs but creates just the `nmstates` CRD and
> waits; applying the NNCPs fails with `no matches for kind
> "NodeNetworkConfigurationPolicy" in version "nmstate.io/v1"`. Create the default
> instance first (idempotent):

```bash
cat <<'EOF' | oc apply -f -
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: cluster
spec: {}
EOF
# confirm the CRD is now registered before applying the NNCPs:
oc get crd nodenetworkconfigurationpolicies.nmstate.io
```

```bash
# mtv-test namespace is required by the macvlan NAD
oc create namespace mtv-test --dry-run=client -o yaml | oc apply -f -

oc apply -f luke-network-config.yaml
```

All five NNCPs should reach `Available: True`. Verify:

```bash
oc get nodenetworkconfigurationpolicy \
  -o custom-columns=NAME:.metadata.name,AVAIL:.status.conditions[?(@.type=="Available")].status,PROG:.status.conditions[?(@.type=="Progressing")].message
```

---

## The five NNCPs

### 1. `br-vmdata` — OVS bridge for VM data (all nodes)

A dedicated Open vSwitch bridge with `bond0` as a port, allowing all VLANs and
untagged traffic. No nodeSelector → applies to **every** node. This is the primary
VM data-plane bridge.

```yaml
kind: NodeNetworkConfigurationPolicy
metadata: { name: br-vmdata }
spec:
  desiredState:
    interfaces:
    - type: ovs-bridge
      name: br-vmdata
      state: up
      bridge:
        allow-extra-patch-ports: true   # OVN/OVS patch ports can attach
        options: { stp: false }
        port: [ { name: bond0 } ]
```

### 2. `control01-nfs-nncp` — NFS VLANs on control01 (VLAN 80 + 90)

Two VLAN sub-interfaces on `bond0` with static IPs, scoped to `control01`:

| Interface | VLAN | IP |
|-----------|------|----|
| `bond0.80` | 80 | `192.168.80.26/24` |
| `bond0.90` | 90 | `192.168.90.26/24` |

```yaml
spec:
  desiredState:
    interfaces:
    - { type: vlan, name: bond0.80, state: up, mtu: 1500,
        vlan: { base-iface: bond0, id: 80 },
        ipv4: { enabled: true, dhcp: false, address: [{ ip: 192.168.80.26, prefix-length: 24 }] } }
    - { type: vlan, name: bond0.90, state: up, mtu: 1500,
        vlan: { base-iface: bond0, id: 90 },
        ipv4: { enabled: true, dhcp: false, address: [{ ip: 192.168.90.26, prefix-length: 24 }] },
        ipv6: { enabled: false } }
  maxUnavailable: 1
  nodeSelector: { kubernetes.io/hostname: control01.syangsao.net }
```

### 3. `control02-nfs-nncp` — NFS VLANs on control02 (VLAN 80 + 90)

Identical shape to #2, scoped to `control02`, with `.27` addresses:

| Interface | VLAN | IP |
|-----------|------|----|
| `bond0.80` | 80 | `192.168.80.27/24` |
| `bond0.90` | 90 | `192.168.90.27/24` |

```yaml
spec:
  desiredState:
    interfaces:
    - { type: vlan, name: bond0.80, state: up, mtu: 1500,
        vlan: { base-iface: bond0, id: 80 },
        ipv4: { enabled: true, dhcp: false, address: [{ ip: 192.168.80.27, prefix-length: 24 }] } }
    - { type: vlan, name: bond0.90, state: up, mtu: 1500,
        vlan: { base-iface: bond0, id: 90 },
        ipv4: { enabled: true, dhcp: false, address: [{ ip: 192.168.90.27, prefix-length: 24 }] },
        ipv6: { enabled: false } }
  maxUnavailable: 1
  nodeSelector: { kubernetes.io/hostname: control02.syangsao.net }
```

> VLANs 80/90 carry the NFS storage endpoints for the control nodes. The `.26`/`.27`
> host addresses are stable across rebuilds — do not renumber them.

### 4. `vlan60-virtualnetwork` — VLAN 60 VM network (worker nodes)

This is the key piece for VM networking. It creates, on every **worker**-labeled
node:

1. A VLAN sub-interface `bond0.60` (VLAN 60 on `bond0`).
2. An OVS bridge `ovs-br-vlan-60` with `bond0.60` as its port.
3. An **OVN bridge mapping** binding that bridge to the OVN localnet `vlan-60`.

```yaml
spec:
  desiredState:
    interfaces:
    - { type: vlan, name: bond0.60, state: up,
        vlan: { base-iface: bond0, id: 60 } }
    - type: ovs-bridge
      name: ovs-br-vlan-60
      state: up
      bridge:
        allow-extra-patch-ports: true
        options: { stp: false }
        port: [ { name: bond0.60 } ]
    ovn:
      bridge-mappings:
      - { bridge: ovs-br-vlan-60, localnet: vlan-60, state: present }
  nodeSelector: { node-role.kubernetes.io/worker: "" }
```

The `ovn.bridge-mappings` entry is what makes OVN treat `ovs-br-vlan-60` as the
physical egress for the logical network named `vlan-60`. On luke, both control nodes
carry the `node-role.kubernetes.io/worker` label (SNO-style: master+worker), so this
NNCP lands on `control01` and `control02`. The OVN northbound database then shows a
localnet switch per node:

```
switch vlan.60_ovn_localnet_switch
    port vlan.60_ovn_localnet_port
        type: localnet
```

> **Rebuild note:** on a fresh install the control nodes may not yet have the
> `worker` role label. If this NNCP reports 0 target nodes, confirm
> `node-role.kubernetes.io/worker` is present (it is on luke because the control
> nodes are dual-role). The bridge mapping only takes effect where the OVS bridge
> exists.

### 5. `arbiter-ext-mtu` — MTU 1500 on arbiter (OVN overlay)

Sets MTU 1500 on the arbiter node's external interface, the OVS bond interface, and
the `br-ex` bridge so the OVN overlay path is consistent.

```yaml
spec:
  desiredState:
    interfaces:
    - { type: ethernet,     name: ens192, state: up, mtu: 1500 }
    - { type: ovs-interface, name: bond0,  state: up, mtu: 1500 }
    - { type: ovs-bridge,   name: br-ex,   state: up, mtu: 1500 }
  maxUnavailable: 0
  nodeSelector: { kubernetes.io/hostname: arbiter.syangsao.net }
```

---

## The two NADs (NetworkAttachmentDefinitions)

### `default/vlan-60` — OVN localnet (existing VMs)

An OVN-k8s overlay network using the `vlan-60` localnet, with a fixed OVN network-id
of 5. VMs/pods in `default` attach to this for VLAN-60 connectivity.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-60
  namespace: default
  annotations:
    k8s.ovn.org/network-id: "5"
    k8s.ovn.org/network-name: vlan-60
spec:
  config: |-
    { "cniVersion": "0.3.1", "name": "vlan-60",
      "type": "ovn-k8s-cni-overlay", "topology": "localnet",
      "netAttachDefName": "default/vlan-60", "mtu": 1500, "ipam": {} }
```

### `mtv-test/vlan-60` — macvlan bridge (MTV migration target)

A macvlan-in-bridge-mode NAD in the `mtv-test` namespace for MTV-migrated VMs. It
uses host-local IPAM on `192.168.60.0/24` (range `.10`–`.250`) with `eth1` as the
master. This is why the `mtv-test` namespace must exist before applying.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata: { name: vlan-60, namespace: mtv-test }
spec:
  config: |-
    { "cniVersion": "0.3.1", "name": "vlan-60",
      "plugins": [ { "type": "macvlan", "master": "eth1", "mode": "bridge",
        "ipam": { "type": "host-local", "subnet": "192.168.60.0/24",
                  "rangeStart": "192.168.60.10", "rangeEnd": "192.168.60.250" } } ] }
```

---

## Verification

```bash
# All NNCPs Available
oc get nodenetworkconfigurationpolicy \
  -o custom-columns=NAME:.metadata.name,AVAIL:.status.conditions[?(@.type=="Available")].status

# OVS bridges present on a control node
oc debug node/control01.syangsao.net -- chroot /host ovs-vsctl show | grep -E "br-vmdata|ovs-br-vlan-60"

# VLAN interfaces up
oc debug node/control01.syangsao.net -- chroot /host ip -brief addr show | grep -E "bond0\.(60|80|90)"

# OVN localnet switch for vlan-60 (via ovn-controller container)
ctr=$(crictl ps | grep ovn-controller | grep Running | head -1 | awk '{print $1}')
crictl exec $ctr ovn-nbctl show | grep -A2 "vlan.60_ovn_localnet"

# NADs present in both namespaces
oc get network-attachment-definitions -A
```

## Rebuild notes

- The whole file is idempotent — `oc apply -f luke-network-config.yaml` converges a
  rebuilt cluster to this exact state.
- The **only** prerequisite that must exist first is the `mtv-test` namespace
  (created in the quick-reference at the top).
- Static IPs (`.26`/`.27` on VLANs 80/90) and the OVN network-id (`5`) are stable
  identifiers — keep them unchanged across rebuilds so NFS exports and existing VM
  networking continue to resolve.
- If `vlan60-virtualnetwork` targets 0 nodes after a rebuild, check that the control
  nodes carry `node-role.kubernetes.io/worker` (they do on luke's dual-role layout).
