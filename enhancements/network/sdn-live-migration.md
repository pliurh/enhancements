---
title: sdn-live-migration
authors:
  - "@pliurh"
reviewers:
  - "@danwinship"
  - "@trozet"
  - "@dcbw"
  - "@russellb"
approvers:
  - "@danwinship"
  - "@trozet"
  - "@dcbw"
  - "@russellb"
api-approvers:
  - "@danwinship"
  - "@trozet"
  - "@dcbw"
  - "@russellb"
creation-date: 2022-03-18
last-updated: 2022-04-21
tracking-link:
  - https://issues.redhat.com/browse/SDN-2612
status: implementable
---

# SDN Live Migration

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)


## Summary
Migrating the CNI network provider network of a running cluster from
OpenShift SDN to OVN Kubernetes without service interruption. During the
migration, we will partition the cluster into two sets of nodes controlled by
different network plugins. We will utilize the Hybrid overlay feature of
OVN Kubernetes to connect the networks of the two CNI network plugins. So that
pods on each side can still talk to the pods on the other side.

## Motivation

For some Openshift users, they have very high requirement on service
availability. The current SDN migration solution, that will cause a service
interruption, is not acceptable.

### Goals

- Migrate the cluster network provider from OpenShift SDN to OVN Kubernetes for a 
  existing cluster.
  - The OVN Kubernetes will be running in the multi-zone IC mode.
- This solution shall work in all platforms managed by the SD team.
  - ARO (Azure Red Hat OpenShift)
  - OSDv4 (OpenShift Dedicated) on GCP
  - OSDv4 (OpenShift Dedicated) on AWS
  - ROSA Classic (Red Hat OpenShift Service on AWS)
- This is in-place migration without requiring extra nodes.
- The impact to workload of the migration shall be similar as an OCP upgrade.
- The solution can work in scale, e.g. in a large cluster with 100+ nodes.
- The migration operation shall be able to be rolled back if needed.

### Non-Goals

- Support for migration to other network providers
- The necessary GUI change in Openshift Cluster Manager.
- Ensure the following features remain working during live migration:
  - Egress IP
  - Egress Router
  - Egress Firewall
  - Multicast
- Migration from SDN Multi-tenant mode
- Migrate to OVN Kubernetes single-zone IC mode.
- Support Hypershift clusters

## Pre-requisites

- OVN-IC multi-zone mode in OCP is dev-complete.

## Proposal

The key problem of do a live SDN migration is that, we need to the connectivity 
of the cluster network during the migration, when pods are attached to different
networks. We propose to utilize the OVN Kubernetes hybrid overlay feature to
connect the networks owned by OpenShift SDN and OVN Kubernetes.

- We will run different plugins on different nodes, but both plugins will know
  how to reach pods owned by the other plugin, so all pods/services/etc remain
  connected.
- During migration, CNO will take original-plugin nodes one by one and convert
  them to destination-plugin nodes, rebooting them in the process.
- The cluster network CIDR will remain unchanged, so does the node host subnet
  of each node.
- NetworkPolicy will work correctly throughout the migration.

### Limitations

- The following features which are only supported by OpenShift SDN but not by
  OVN Kubernetes, will stop working when the migration is started.
  - Multitenant Isolation
- The Egress Router feature is supported by both OpenShift SDN and OVN
  Kubernetes but with different design and implementation. Therefore even with
  the same name, it have different API and configure logic between OpenShift
  SDN and OVN Kubernetes. Also, the supported platforms and modes are different
  between the two network providers. So users need to evaluate before conducting
  the live migration, and have to migrate the configuration manually after the
  migration is complete.
- The following features are supported by both OpenShift SDN and OVN Kubernetes.
  And we have support automated converting for the configuration. However, users
  shall not expect these feature can function as normal during the live
  migration phase.
  - Multicast
  - Egress IP
  - Egress Firewall

### User Stories

The service delivery (SD) team (managed OpenShift services ARO, OSD, ROSA) has a
unique set of requirements around downtime, node reboots, and high degree of
automation. Specifically, SD need a way to migrate their managed fleet in a way
that is no more impactful to the customer's workloads than an OCP upgrade, and
that can be done at scale, in a safe, automated way that can be made
self-service and not require SD to negotiate maintenance windows with customers.
The current migration solution needs to be revisited to support these
(relatively) more stringent requirements.

### Risks and Mitigations

## Design Details

The existing ovn-kubernetes hybrid overlay feature was developed for hybrid
Windows/Linux clusters. Each ovn-kubernetes node manages an external-to-OVN OVS
bridge, named br-ext, which acts as the VXLAN source and endpoint for packets
moving between pods on the node and their cluster-external destination. The
br-ext SDN switch acts as a transparent gateway and routes traffic towards
windows nodes.

In the SDN live migration use case, we can enhance this feature to connects the
nodes managed by different CNI plugins. To minimize the implementation effort,
and the code maintainability, we will try to reused the hybrid overlay code and
only make necessary changes to both CNI plugins. OVN Kubernetes nodes (OVN-IC
zones) will have full-mesh connections to all OpenShift SDN Nodes through VXLAN
tunnel.

On the OVN Kubernetes side, all the cross CNI traffic shall follow the same path
of current hybrid overlay implementation. For OVN Kubernetes, we need to do
following enhancements:

1. We need to prevent OVN-K from allocating subnet for each host. We need to
   reuse the host subnet allocated by OpenShift SDN.
2. We need to modify OVN-K to allow overlapping between the cluster network and
   the Hybrid overlay CIDR.
3. We need to modify OVN-K to allow adding/removing hybrid overlay nodes on the
   fly.
4. We need to allow `hybrid-overlay-node` running on the linux nodes using
   OpenShift SDN as CNI plugin, currently it is designed to only run on windows
   nodes. It is responsible for: 
   - collecting the MAC address of the host primary interface, and set the node
     annotation `k8s.ovn.org/hybrid-overlay-distributed-router-gateway-mac`.
   - removing the pod annotations added by OVN Kubernetes for pods running on
     local node.

On the OpenShift SDN side, when a node is converted to OVN Kubernetes, it shall
be almost transparent to the control-plane of OpenShift SDN. But we still need
to introduce a 'migration mode' for OpenShift SDN by:

1. Change ingress NetworkPolicy processing to be based entirely on pod IPs
   rather than using namespace VNIDs, since packets from ovn nodes will have
   the VNID 0 set.
2. To be compatible with Windows node VXLAN implementation, OVN-K hybrid overlay
   uses the peer host interface MAC as VXLAN inner dest MAC for egress traffic.
   Therefore when packets arrive at the br0 of the SDN node, they cannot be
   forwarded to the pod interface correctly. We need to add flows for each pod
   to change the dst MAC to the pod interface MAC.

### The Traffic Path

#### Packets going from OpenShift SDN to OVN Kubernetes

On the SDN side, it doesn't need to know if the peer node is a SDN node or a OVN
node. We reuse the existing VXLAN tunnel rules one the SDN side.
- Egress NetworkPolicy rules and service proxying happen as normal
- When the packet reaches table 90, it will hit a "send via vxlan" rule that was
  generated based on a HostSubnet object.

On the OVN side:
- OVN accepts the packet via the VXLAN tunnel, ignore the VNID set by SDN and
  then just routes it normally.
- Ingress NetworkPolicy processing will happen when the packet reaches the
  destination pod's switch port, just like normal.
  - Our NetworkPolicy rules are all based on IP addresses, not "logical input
    port", etc, so it doesn't matter that the packets came from outside OVN and
    have no useful OVN metadata.

#### Packets going from OVN Kubernetes to OpenShift SDN

On the ovn side:
- The packet just follow the same path of hybrid overlay. The packet just has to
  get routed out the VXLAN tunnel with VNID 0.

On the sdn side:
- We have to change ingress NetworkPolicy processing to be based entirely on pod
  IPs rather than using namespace VNIDs, since packets from ovn nodes won"t have
  the VNID set. There is already code to generate the rules that way though,
  because egress NetworkPolicy already works that way.

### Workflow Description

#### Migration Setup

1. The admin kicks off the migration process by updating the custom resource
   `network.operator`:
   
   ```bash
   $ oc patch Network.operator.openshift.io cluster --type='merge' --patch '{"spec":{"migration":{"networkType":"OVNKubernetes","isLive":true}}}"
   ```
   
   CNO will check if there is any unsupported feature (refer the
   [Limitations](#limitations) section) enabled in the cluster. If yes, it will
   set an warning message under the `status.migration` field of the custom
   resource `network.operator`.

2. CNO will redeploy the openshift-sdn in the __migration mode__. It will add a
   condition check to the wrapper script of the sdn DaemonSet, to see if the
   bridge `br-ex` exists on the node . If `br-ex` exists, it means the node has
   already been updated by MCO, therefore it's ready for running OVN Kubernetes,
   the sdn pod would just do "sleep infinity" rather than actually launching the
   `openshift-sdn-node` process.

3. CNO will also deploy OVN Kubernetes to the cluster with hybrid overlay
   enabled. The ovnkube-node DaemonSet wrapper script also has the logic to
   check if the bridge br-ex exists on the node. If `br-ex` doesn't exist, it
   means the node has yet been updated by MCO, so instead of starting the
   `ovnkube-node` process, it would run `hybrid-overlay-node` process that
   discover the necessary hybrid overlay node information and annotate node
   accordingly e.g.:

   ```yaml
   k8s.ovn.org/hybrid-overlay-distributed-router-gateway-mac: 00-c2-f5-92-28-ad
   k8s.ovn.org/hybrid-overlay-node-subnet: 192.168.111.20/24
   ```

   otherwise it would launch the `ovnkube-node` process.

#### Migration

We can kick off the migration from CNO, by:

```bash
$ oc patch Network.config.openshift.io cluster --type='merge' --patch '{"spec":{"networkType":"OVNKubernetes"}'
```

CNO will update the `status.networkType` field of the `network.config` CR. It
will trigger MCO to apply new MachineConfig to each node:

1. MCO will 
   - rerender the MachineConfig for each node
   - try to cordon and drain the nodes based on
     `MachineConfigPools.spec.maxUnavailable`
2. CNO will be able to find a node is being drained by MCO by watching the MCO
   node annotations, it will clear the `k8s.ovn.org/hybrid-overlay-xxx` node
   annotation then set the annotation `k8s.ovn.org/node-subnets:` according to
   the HostSubnet object of the node to bypass the ovnkube node subnet
   allocation. 
3. CNO will also update the hybrid overlay configuration and the onvkube-master
   pods will take the updated hybrid overlay configuration then update the OVN
   database.
4. MCO will drain and reboot the node.
5. After booting up, br-ex will be created and ovnkube-node will run in hybrid
   overlay mode on the node.
6. The wrapper script of ovnkube-node DaemonSet will see br-ex, then launch the
   `ovnkube-node` process.
7. MCO will uncordon the node. Pods will be created on the node using
   ovn-kubernetes as the default CNI plugin.

The above process will be repeated for each node, until all the nodes have been
applied to the new MachineConfig and converted to OVN Kubernetes.

#### Migration Cleanup

Once migration is complete, CNO will:
- delete the openshift-sdn DaemonSets
- redeploy ovn-kubernetes in "normal" mode (no migration mode config, no node
  affinity, hybrid overlay disabled).
- remove the migration-related labels from the nodes

### API

A new field `spec.migration.isLive` will be introduced to the CRD
`networks.operator.openshift.io`. Users can set this flag to conduct a live
migration or a offline migration.

To start the migration, users need to update the `network.operator`
CR by adding:

CNO will deploy both OpenShift SDN and OVN Kubernetes pods in 'migration' mode.

```json
{ 
  "spec": { 
    "migration": {
      "networkType": "OVNKubernetes",
      // isLive indicates whether we're doing a live migration or not.
      "isLive": true
    }
  } 
}
```

On removing of the `spec.migration` field, CNO will start the migration cleanup
and redeploy the target CNI in 'normal' mode.

CNO will also report the migration state to under the `status` of
`network.operator` CR.

```json
{ 
  "status": { 
    "migration": {
      // The state can be 'Setup', 'Working', 'Done' or 'Error', 
      "state": "Working",
      // the reason needs to be filled when state is 'Error'.
      "reason": ""
    }
  } 
}
```

### Rollback

Users shall be able to rollback to openshift-sdn after the migration is
complete. The migration is bidirectional, thus users can follow the similar
above-mentioned procedure to conduct the rollback.

### Lifecycle Management

This is a one-time operation for a cluster, therefore no lifecycle management.

### Test Plan

We need to setup a CI job which can run the sig-network test against a cluster
that is in an intermediate state of live migration, specifically some nodes are
running with OVN Kubernetes, and the rests are running with OpenShift SDN. So
that we can ensure the cluster network remains functioning during the migration.

We also need to work closely with the Performance & Scale test team and SD team
to test the live migration on a large (100+ nodes) cluster.

### Graduation Criteria

Graduation criteria follows:

#### Dev Preview -> Tech Preview

- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

#### Tech Preview -> GA

- More testing (upgrade, scale)
- Add gating CI jobs on the relevant GitHub repos
- Sufficient time for feedback

#### Removing a deprecated feature
N/A

### Upgrade / Downgrade Strategy

This is a one-time operation for a cluster, therefore no upgrade / downgrade
strategy.

### Drawbacks
N/A

### Version Skew Strategy
N/A

### API Extensions
N/A

### Operational Aspects of API Extensions
N/A

#### Failure Modes
N/A

#### Support Procedures
N/A

## Implementation History
N/A

## Alternatives

Instead of switch the network provider for a existing cluster, we can spin up a
new cluster then move the workload to it.

## Infrastructure Needed
N/A
