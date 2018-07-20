# 1. Introduction
Provide BGP EVPN Multicast Type-6 Selective Multicast Ethernet Tag Route support

# 2. Problem statement
Currently, all BUM traffic is carried over the IMET routes, thus flooding all
compute nodes irrespective of whether there is any active receiver present or
not in each of those compute-nodes.

As part of this feature, EVPN Type-6 SMET routes are supported in order to build
and use multicast trees selectively, on a per <S, G> basis.

# 3. Proposed solution

EVPN [6] proposes Type-6 SMET routs in order to send/receive traffic selectively
based on the presence/absence of active receivers for a particular <S, G> or
<*, G> multicast group. This can be utilized by contrail-controller to advertise
the reach-ability of interested multicast receivers.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
TBD: Allowed IGMP groups on a per VMI/VN/Global configuration

## 3.3 User workflow impact
Support for SMET Routes is automatically enabled. As a fall-back mechanism, we
could implement a flag in contrail-control.conf to disable advertising and/or
processing SMET routes.

## 3.4 UI changes
UI shall provide a way to enable/disable IGMP at virtual-machine-interface,
virtual-network or global-system-config level.

In addition, we should be able to accept a list of <S,G>/<*,G> and allow joins
for only those, on a per VMI/VN/Global level. (TBD)

## 3.5 Notification impact
####Describe any log, UVE, alarm changes

# 4. Implementation

## 4.1 Capability negotiation
EVPN Type-6 SMET routes capability is indicated by attaching a specific bgp
community attribute "Ethernet Multicast flags Community (MF)" to the IMET
routes. It is possible that one control-node supports SMET routes and other
does not (especially during upgrades). If the protocol sequence is correctly
followed, this should work correctly. Those that do not support SMET always
floods the packets over IMET routes. This shall ensure backward compatibility.
Also it is to be noted that only control-node generates the SMET route. We call
this the multicast master control-node which computes level-1 edge replication
tree.

## 4.2 Concurrency Model

Today all EVPN multicast routes (IMET, etc.) are not processed concurrently.
They all fall onto partition 0. We could consider processing the routes in
parallel on per <S, G> basis. EvpnTable::HashFunction() needs to be tweaked
appropriately to achieve this. Careful code analysis is necessary in order to
ensure that there is no unsafe EVPN data structures during multiple groups
parallel processing.

## 4.3 Phase-1 Events (For Release 5.1)

In Phase-1 receivers are always inside the contrail cluster and sender is always
outside the cluster. Support for <*, G> is aimed. Support for <S, G> is handled
at a lower priority.

## 4.4 SMET Route advertisement

Only one of the control-nodes which computes the Level-1 tree (aka global mcast
master node) shall originate the Type-6 route. Using route-listener callback on
the global ermvpn route, the master should originate a type-6 route when ever
global ermvpn route is originated (or updated). The nexthop for this type-6
SMET route shall be the forest node in the tree. (which is associated with the
global ermvpn route). This scheme is very similar to MVPN where in type-4 leafad
route is originated towards the sender. In this case, EVPN type-6 route is
originated instead. OTOH, if global ermvpn route is deleted, then the originated
Type6 SMET route is also automatically deleted and hence withdrawn from the rest
of the network.

## 4.13 Multicast Edge Replicated Tree

Build one tree for each C-<*/S, G> within each virtual network when there is at
least one receiver present completely using existing ermvpn mechanisms. As said
before, forest node of the tree is used as the ingress entry point for multicast
traffic to enter the tree. Tree can be built completely using existing ErmVpn
feature.

Master control-node which builds level-1 tree shall also be solely responsible
for managing type-6 routes. Today, this is simply decided based on control-node
with the lowest router-id.

Agent shall continue to advertise routes related to BUM traffic over xmpp to
vrf.ermvpn.0 table. (IMET route). In addition, all IGMP joins are aggregated to
specific ermvpn routes and advertised to the same vrf.ermvpn.0 table.

Control-node shall build an edge replicated multicast tree like how it does so
already. PMSI Tunnel information is gathered from the forest node of the locally
computed tree. (bottom-most and right-most leaf node in the replication tree).
However, unlike for MVPN Type-4 routes, for EVPN Type-6 SMET routes, no PMSI
tunnel information can/is to be encoded. Instead just the nexthop which points
to the forest compute node should suffice. During forwarding, sender uses PMSI
information sent along with the IMET route from the associated SMET nexthop node
to do the actual multicast packet forwarding.

When ever tree is recomputed (and only if the forest node changes), Type-6 route
is updated and re-advertised as necessary. If a new node is selected as as the
forest node, SMET route can be updated and re-advertised (implicit withdraw and
new update)

ErmVpn Tree built inside the contrail cluster is fully bi-directional and self
contained. Vrouter would flood the packets within the tree only so long as the
packet was originated from one of the nodes inside the tree (in the oif list).
In case of MVPN, "Input Tunnel Attribute" was added to assist the RPF check for
traffic coming in from the gateway. However, in case of EVPN+VXLAN, label shall
always point to the VN Table and multicast mac look up is done again inside the 
table to figure out how to forward the packet. Hence, for this scheme, we do not
propose addition of any "Input Tunnel Attribute" to relax the RPF check. This
will be an expensive operation, especially when the SMET route is sent to many
different BGP EVPN TOR Routers.

For Phase I, since sender is always outside the contrail cluster, this mechanism
should suffice to receive multicast traffic, for any <*, G>/<S, G> multicast
traffic and then flood over the selective tree.

## 4.5 Sender inside contrail-cluster

When sender is present within the contrail-cluster (which would be the case
for test environments if not for production in R5.1, mcast mac route must be
downloaded to all agents in the network (which has at least one VMI in the
associated virtual-network) with the forest node as the nexthop. This can be
achieved by sending the GlobalErmVpnRoute also to all the agents. Today, only
routes of type ErmVpnPrefix::NativeRoute are sent to agents over xmpp. This can
enable the agent to program a multicast mac entry with next-hop olist same as
that of the forest node in each of the compute in the cluster as applicable.

## 4.6 SMET Routes reception

Once the sender inside the contrail-cluster support is added, when we receive
one or more Type-6 SMET routes from remote non-control-node BGP Peers, we need
to stitch those SMET based receivers also to the tree. This can be done in a way
similar to how received IMET routes are already processed today.

When ever a new SMET route is received, a new EvpnRemoteMcastNode() object is
created corresponding to the new SMET route received (which is specific to a
<S,G> or <*,G> multicast group. This is how IMET routes are processed today.

In EvpnLocalMcastNode::GetUpdateInfo(), remote_mcast_node_list is walked to
form the bgp olist. For SMET routes, logic can be similar, except that we now
need one remote_mcast_node_list per group basis. IOW, IMET routes was specific
to broad-cast address, where as SMET routes is for a specific multicast group
address.

For each group, we shall maintain a remote_mcast_node_list that tracks the list
of all EvpnRemoteMcastNode() that advertised an SMET route for that group.

During forwarding, each node in the tree replicates packet to all of its
neighbors. In addition, when the packet first enters the tree (from a VMI), it
is also flooded to all EvpnRemoteMcastNode() (SMET route advertisers).

Care should be taken to ensure that if a packet is received from the remote
site to the forest node, (because we had advertised the SMET route), it is not
replicated back to all other EvpnRemoteMcastNode(). It is only to be replicated
to local VMIs and to neighboring connected nodes in the tree.

## 4.15 Agent Implementation

In addition to what is already supported as part of MVPN feature, agent should
o Configurable IGMP groups ?

o Accept ethernet multicast traffic from remote SDN gateway and send them to
  the neighboring nodes in the tree (in addition to local VMIs as applicable)

o Accept ethernet multicast traffic from local VMIs and send it to the olist
  associated with the specific group address.

# 5. Performance and scaling impact

##5.2 Forwarding performance
TBD

# 6. Upgrade
By associating Ethernet Multicast flags Community (MF) with the IMET routes,
control-node should signal to bgp peers its intent of support to SMET routes.
If SMET route is not supported, agent should continue to send multicast traffic
over the BUM tree (which is the tree associated with the broadcast route)

# 7. Deprecations
N/A

# 8. Dependencies

## 8.1
1. SMET Routes support in JUNOS (QFX)

# 9. Testing
## 9.1 Unit tests

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
[1]: [BGP MPLS-Based Ethernet VPN](https://tools.ietf.org/html/rfc7432)

[2]: [BGP Encodings and Procedures for Multicast in MPLS/BGP IP VPNs](https://tools.ietf.org/html/rfc6514)

[3]: [Ingress Replication Tunnels in Multicast VPN](https://tools.ietf.org/html/rfc7988)

[4]: [Extranet Multicast in BGP/IP MPLS VPNs](https://tools.ietf.org/html/rfc7900)

[5]: [BGP/MPLS IP Virtual Private Networks (VPNs)](https://tools.ietf.org/html/rfc4364)

[6]: [EVPN IGMP Proxy](https://tools.ietf.org/html/draft-sajassi-bess-evpn-igmp-mld-proxy-02)

[7]: [Contrail MVPN Functional Specification](https://github.com/Juniper/contrail-specs/blob/master/bgp_mvpn.md)
