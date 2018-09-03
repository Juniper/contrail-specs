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

A change needed in xmpp schema which is used for communication between
Control node and compute node for relaying route information.

```
diff --git a/schema/xmpp_enet.xsd b/schema/xmpp_enet.xsd
index c05e06e..3b01474 100644
--- a/schema/xmpp_enet.xsd
+++ b/schema/xmpp_enet.xsd
@@ -47,6 +47,9 @@ xsd:targetNamespace="http://www.contrailsystems.com/xmpp-enet-cfg.xsd">
     <xsd:element name="ethernet-tag" type="xsd:integer"/>
     <xsd:element name="mac" type="xsd:string"/>
     <xsd:element name="address" type="xsd:string"/>
+    <xsd:element name="group" type="xsd:string"/>
+    <xsd:element name="source" type="xsd:string"/>
+    <xsd:element name="flags" type="xsd:integer"/>
 </xsd:complexType>

 <xsd:complexType name="EnetSecurityGroupListType">
```

Additional change in schema that is relevant for IGMP and EVPN multicast
is described as below.

Schema will have fields to configure IGMP mode at the Global System
Configuration level, at Virtual Network level and at VMI level.

Schema will also have list of <S,G>/<*,G> that will be allowed by IGMP. This
list is available at the Virtual Network level. The schema changes is along
the following lines:

<xsd:element name="multicast-policy" type="ifmap:IdentityType"/>
<!--Identifier "multicast-policy" is a node to which list of multicast (S,G)
    will hang from. Currently this will be linked to virtual-network. -->

<xsd:complexType name="MulticastSourceGroup">
    <xsd:all>
        <xsd:element name="source-address" type="IpAddressType" required='true' description='Multicast Source Address'/>
        <xsd:element name="group-address" type="IpAddressType" required='true' description='Multicast Group Address'/>
        <xsd:element name="action" type="SimpleActionType" required='true' description='Pass or deny action for (S,G) matching this rule'/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="MulticastSourceGroups">
    <xsd:element name="multicast-source-group" type="MulticastSourceGroup" maxOccurs="unbounded"/>
</xsd:complexType>

<xsd:element name="multicast-source-groups" type="MulticastSourceGroups"/>
<!--#IFMAP-SEMANTICS-IDL
     ListProperty('multicast-source-groups', 'multicast-policy', 'optional',
                  'CRUD', 'List of Multicast (S,G) Addresses.') -->

<xsd:element name="virtual-network-multicast-policy"/>
<!--#IFMAP-SEMANTICS-IDL
     Link('virtual-network-multicast-policy',
          'virtual-network', 'multicast-policy', ['ref'], 'optional', 'CRUD',
          'Reference to multicast policy. Each multicast policy has a list of
           (S,G) Addresses.') -->

TBD: Could the above schema changes be extended to VMI level?

## 3.3 User workflow impact
Support for SMET Routes is automatically enabled. As a fall-back mechanism, we
could implement a flag in contrail-control.conf to disable advertising and/or
processing SMET routes.

## 3.4 UI changes
UI shall provide a way to enable/disable IGMP at virtual-machine-interface,
virtual-network or global-system-config level.

In addition, we should be able to accept a list of <S,G>/<*,G> and allow joins
for only those on a per VN level.

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

## 4.5 Multicast Edge Replicated Tree

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

## 4.6 Sender inside contrail-cluster

When sender is present within the contrail-cluster (which would be the case
for test environments if not for production in R5.1, mcast mac route must be
downloaded to all agents in the network (which has at least one VMI in the
associated virtual-network) with the forest node as the nexthop. This can be
achieved by sending the GlobalErmVpnRoute also to all the agents. Today, only
routes of type ErmVpnPrefix::NativeRoute are sent to agents over xmpp. This can
enable the agent to program a multicast mac entry with next-hop olist same as
that of the forest node in each of the compute in the cluster as applicable.

## 4.7 SMET Routes reception

Once the sender inside the contrail-cluster support is added, when we receive
one or more Type-6 SMET routes from remote non-control-node BGP Peers, we need
to stitch those SMET based receivers also to the tree. This can be done in a way
similar to how received IMET routes are already processed today.

When ever a new SMET route is received, a new EvpnRemoteMcastNode() object is
created corresponding to the new SMET route received (which is specific to a
<S,G> or <*,G> multicast group. This is how IMET routes are processed today.

In EvpnLocalMcastNode::GetUpdateInfo(), remote_mcast_node_list is walked to
form the bgp olist. With SMET route support, we’ll change the data strucuture so
that there is separate list per <S/*,G>.

For SMET routes, logic can be similar, except that we now need one
remote_mcast_node_list per group basis. IOW, IMET routes was specific
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

## 4.8 Assisted replication
Contrail VRouter already supports edge replication tree. Hence any one VRouter
node does not need to provide an assisted replication per-se. However, if the
nexthops associated with SMET routes indicates support for Assisted replication,
then VRouter should flood the packets to only one such external node which can
act as assisted replicator. This is only applicable when we start to support
sender inside the contrail-cluster (Not targeted for R5.1)


## 4.9 Agent Implementation

In addition to what is already supported as part of MVPN feature, Agent should:

o Have configurable IGMP <S,G>/<*,G> list at the VN level.

o Accept multicast traffic from remote SDN gateway and send them to the
  neighboring nodes in the tree in addition to local VMIs, if any.

o Accept multicast traffic from local VMIs and send it to the olist
  associated with the specific <*,G> group.

Note: Future contrail release might have a per-VMI <S,G>/<*,G> list.

Agent should also send SMET equivalent XMPP messages to the control BGP
for each of the <*,G> multicast groups learnt through IGMP.

Agent should also handle XMPP route updates from the control BGP and update
the vrouter accordingly.

## 4.10 vRouter Implementation
IGMP packets in vRouter are trapped at the per-VMI level if IGMP gets
enabled at the VMI. This can happen when IGMP is enabled specifically
at the VMI or at the VN or at the global system configurationn.

If IGMP is not enabled at the VMI, IGMP packets are forwarded using
the broadcast MAC entry to flood the packet in the VN. This also means
that care should taken while enabling IGMP at the VMI level.

vRouter also has to handle forwarding multicast data packets. For
release 5.1, MFIB table that is indexed by <S,G> is not implemented. Instead
multicast data packets are forwarded in the contrail by doing a lookup in
bridge table using multicast in the destination field of the packet.

Note: Future contrail releases might support a MFIB lookup in native VRF or
in the multicast VRF.

vRouter, in release 5.1, will receive packets from outside the contrail
on the fabric interface. The packets are encapsulated using VxLan. These
packets will be decapsulated and a bridge lookup is done using the inner
header multicast MAC. The resulting bridge entry will be used to forward
the packets appropriately.

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
