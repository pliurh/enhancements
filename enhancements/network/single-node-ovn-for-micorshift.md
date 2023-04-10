---
title: single-node-ovn-for-microshift
authors:
- "@pliurh"
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
- TBD
approvers: # A single approver is preferred, the role of the approver is to raise important questions, help ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.  Having multiple approvers makes it difficult to determine who is responsible for the actual approval.
- TBD
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
- Node
creation-date: 2023-04-10
last-updated: 2023-04-10
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
- https://issues.redhat.com/browse/NP-654
see-also:
- Node
replaces:
- Node
superseded-by:
- Node
---

# Single Node OVN-Kubernetes for Microshift

## Summary

In OVN Kubernetes the north-south traffic between the pod and the external
network is provided by the gateway logical router connected to the external
logical switch in the OVN logical topology. And there is shared gateway bridge
configured with OpenFlow rules to handle the traffic entering/leaving the node.
The gateway router, external logical switch and the shared gateway bridge
constitute the gateway infrastructure of a node. This document captures a
proposal for adding a new single node mode without gateway infrastructure in
OVN-K.

## Motivation

Micoshift is designed to run on IoT and edge devices, it can be deployed on
various kinds of devices with different network environment. We want Microshift
can be running without requiring any host network configuration.

### User Stories

* As an Microshift cluster administrator, I want the installation experience of
Microshift is similar as a normal application. I don't want to have any
networking interrupt during the installation.

* As an Microshift cluster administrator, I want to use a specific network
  devices (such as wifi, tap/tun) as the node default interface.

### Goals

* Get rid of the microshift-ovs-init.service (which invokes configure-ovs.sh
  script for configuring the shared gateway bridge on a node)
* Optimize the OVN-K logical topology for single node cluster

### Non-Goals

* Make the disabled gateway mode work in a multi-node cluster

## Proposal

In this single node mode, the gateway router, external logical switch and the
shared gateway bridge will not be created. The new logical topology is:

```none
        ┌─────────────┐
        │ eth0        │
    ┌───┤ 10.89.0.3/24├──────────────────────────────────┐
    │   └─┬───────────┘       node                       │
    │     │                                              │
    │     │                                              │
    │  ┌──┴──────┐ ┌───────────────────────────┐         │
    │  │routes + │ │ OVN Cluster Router        │         │
    │  │iptables │ │                           │         │
    │  └──┬──────┘ │        10.244.0.1/24      │         │
    │     │        └──────────────┬────────────┘         │
    │     │                       │                      │
    │     │        ┌──────────────┴────────────┐         │
    │     │        │ Node Switch               │         │
    │     │        └──────────────┬────────────┘         │
    │     │                       │                      │
    │     │  ┌──────────────┐     │     ┌──────────────┐ │
    │     └──┤mp0           ├─────┴─────┤Pod           │ │
    │        │10.244.0.2/24 │           │10.244.0.50/24│ │
    │        └──────────────┘           └──────────────┘ │
    │                                                    │
    └────────────────────────────────────────────────────┘

```

There is no change to the overlay path taken by the east-west traffic between
the pods. The path for following traffic case will be changed:

* Overlay pod to external
* Host to ClusterIP services backed by overlay pods
* Host to NodePort services backed by overlay pods
* Host to CLusterIP services backed by host pods
* Host to NodePort services backed by host pods
* External to NodePort services backed by overlay pods
* External to NodePort services backed by host pods

### Overlay Pod to External

A new iptables rule will added to masquerade the egress traffic from overlay pods.

```none
-A POSTROUTING -s 10.244.0.0/24 -j MASQUERADE
```

The general flow is:

1. Packet is sent to the OVN cluster router via the node switch.
2. According to the static routes in the cluster router, all the packets from
   pods will be forward to the mp0 of the same node.
3. Packet enters the host routing table, they will be masqueraded with the node
   IP before sent out through eth0.

### Host to ClusterIP Services Backed by Overlay Pods

We need to add a static route in the host routing table to steer the traffic
destined for service network to the OVN cluster router interface via mp0.

```none
10.96.0.0/16 via 10.244.0.1 dev ovn-k8s-mp0
```

The general flow is:
1. Packet is sent to the OVN cluster router through mp0.
2. The loadbalancer in the node switch DNAT the packet with the ip of the pod.
3. As it's single node cluster, the packet will be forward by the node switch to
   the pod.


### Host to NodePort Services Backed by Overlay Pods

This flow is similar as the host to ClusterIP service backed by overlay pods.
The only difference is that before entering OVN logical topology, the packet is
DNAT by host iptables by the following rule. Then the dest of the packet is a
clusterIP service.

```none
-A OVN-KUBE-NODEPORT -p tcp -m addrtype --dst-type LOCAL -m tcp --dport 32181 -j DNAT --to-destination 10.96.228.48:80
```

### Host to ClusterIP Services Backed by Host Pods

As we've got the static route for all the traffic destined for service network
to the OVN cluster route interface 10.244.0.1. The loadbalancer in the node switch
DNAT the packet with the nodeIP and send back to mp0. We will have a packet that
gets dest_ip and src_ip from the same host, kernel will drop such packets. To
avoid that, we need to add a SNAT rule in the cluter router, so that all packets
with src_ip of mp0 will be SNATed before sending back to mp0.

```none
    nat 0d059e20-dd79-4d08-8fed-47925eff4422
        external ip: "10.244.0.1"
        logical ip: "10.244.0.2"
        type: "snat"
```

The general flow is:
1. Packet is sent to the OVN cluster router through mp0.
2. The loadbalancer in the node switch DNAT the packet with the node IP.
3. Packet arrive at the OVN cluster router, according to a static route, the
   packet is sent back to mp0. 
4. As the src_ip of the packet is the mp0's, the packet is SNATed with the IP of
   the cluster router interface before sending to mp0.

### Host to NodePort Services Backed by Host Pods

This flow is similar as the host to ClusterIP service backed by host pods.
The only difference is that before entering OVN logical topology, the packet is
DNAT by host iptables by the following rule. Then the dest of the packet is a
clusterIP service.

```none
-A OVN-KUBE-NODEPORT -p tcp -m addrtype --dst-type LOCAL -m tcp --dport 32181 -j DNAT --to-destination 10.96.228.48:80
```

### External to NodePort Services Backed by Overlay pods

The general flow is:
1. Packet enters host through eth0
2. Packet is sent to the OVN cluster router through mp0.
3. The loadbalancer in the node switch DNAT the packet with the pod IP.
4. Packet is sent to the pod.

### External to NodePort Services Backed by Host pods

The general flow is:
1. Packet enters host through eth0
2. Packet is sent to the OVN cluster router through mp0.
3. The loadbalancer in the node switch DNAT the packet with the node IP.
4. Packet arrive at the OVN cluster router, according to a static route, the
   packet is sent back to mp0.

### Workflow Description

**cluster creator** is a human user responsible for deploying a cluster.

1. The cluster creator sits down at their keyboard...
2. ...
3. The cluster creator sees that their cluster is ready to receive
applications, and gives the application administrator their
credentials.

#### Variation [optional]

N/A

### API Extensions

N/A

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

### Drawbacks

The idea is to find the best form of an argument why this enhancement should
_not_ be implemented.  

What trade-offs (technical/efficiency cost, user experience, flexibility, 
supportability, etc) must be made in order to implement this? What are the reasons
we might not want to undertake this proposal, and how do we overcome them?  

Does this proposal implement a behavior that's new/unique/novel? Is it poorly
aligned with existing user expectations?  Will it be a significant maintenance
burden?  Is it likely to be superceded by something else in the near future?


## Design Details

### Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
> 1. This requires exposing previously private resources which contain sensitive
information.  Can we do this?

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
- [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
- `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

#### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

#### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

#### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
minor release stream without being required to pass through intermediate
versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
steps. So, for example, it is acceptable to require a user running 4.3 to
upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
continue to operate correctly in concert with older component
versions (aka "version skew"). For example, if a node is down, and
an operator is rolling out a daemonset, the old and new daemonset
pods must continue to work correctly even while the cluster remains
in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
misbehaving, it should be possible for the user to rollback to `N`. It is
acceptable to require some documented manual steps in order to fully restore
the downgraded cluster to its previous state. Examples of acceptable steps
include:
- Deleting any CVO-managed resources added by the new version. The
CVO does not currently delete resources that no longer exist in
the target version.

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
in the kubelet? How does an n-2 kubelet without this feature available behave
when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
or CNI may require updating that component before the kubelet.

### Operational Aspects of API Extensions

Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
Indicators) an administrator or support can use to determine the health of the API extensions

Examples (metrics, alerts, operator conditions)
- authentication-operator condition `APIServerDegraded=False`
- authentication-operator condition `APIServerAvailable=True`
- openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
API availability)

Examples:
- Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
- Fails creation of ConfigMap in the system when the webhook is not available.
- Adds a dependency on the SDN service network for all resources, risking API availability in case
of SDN issues.
- Expected use-cases require less than 1000 instances of the CRD, not impacting
general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
this enhancement)

#### Failure Modes

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
(e.g. which kube-controller-manager functionality will stop working), especially regarding
stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
and add them as reviewers to this enhancement.

#### Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
alerts, which log output in which component)

Examples:
- If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
- Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
- The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

- What consequences does it have on the cluster health?

Examples:
- Garbage collection in kube-controller-manager will stop working.
- Quota will be wrongly computed.
- Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
Disabling the conversion webhook will break garbage collection.

- What consequences does it have on existing, running workloads?

Examples:
- New namespaces won't get the finalizer "xyz" and hence might leak resource X
when deleted.
- SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
communication after some minutes.

- What consequences does it have for newly created workloads?

Examples:
- New pods in namespace with Istio support will not get sidecars injected, breaking
their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
consistency?

Examples:
- The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
will not block the creation or updates on objects when it fails. When the
webhook comes back online, there is a controller reconciling all objects, applying
labels that were not applied during admission webhook downtime.
- Namespaces deletion will not delete all objects in etcd, leading to zombie
objects when another namespace with the same name is created.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
