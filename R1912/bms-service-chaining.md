
# 1. Introduction

From 2.1, Contrail supports seamless integration with a Bare Metal Servers
(BMS) that sits outside the Contrail Cloud.  The BMSs and other virtual
instances can be configured to be part of any of the virtual networks
configured in the contrail cluster, facilitating communication between them and
the virtual instances running in the cluster. The solution was achieved by
using the Open vSwitch Database Management Protocol (OVSDB). Through ovsdb
management Juniper switches can exchange control and statistical information
with the SDN controllers, thereby enabling VM traffic from the entities in a
virtual-network (VN) to be forwarded to entities in a physical network and vice
versa. OVSDB, though has been deprecated and replaced with EVPN routing. Both
inter-VN and intra-VN modes are supported. EVPN is used in the control-plane
while VxLAN is the underlying dataplane encapsulation mechanism.  Contrail
introduced the concept of a Logical Router (LR) as part of the SNAT solution
wherein an LR is created to enable private VNs to talk to the public cloud. In
the context of VM-BMS communication, the same LR concept is extended to
facilitate EVPN based VxLAN routing. When enabled for a project, an internal
system VN (InternalVirtualNetwork) is created for every LR in the project. The
VNI for this internal VN can be either be configured as part of the LR
configuration or auto-generated if not specified.  All the routes in the VNs
connected to the LR will be exported to the RT-Int of VN-Int. The VN-Int RI
will have cumulative routes for all the VNs connected to the hosting LR. The
advantage of this approach is that the routes need not be re-populated for
every VN multiple times and instead only once for the VN-Int, thus, helping
scalability. The VN-Int (RT-Int) can be configured on the QFX switch so that
these routes can be leaked to the QFX IRB interface.

![](https://github.com/maheshmns/contrail-specs/blob/master/images/Figure%201.png?raw=true)

Contrail controller also supports chaining of various Layer 2 through Layer 7
services such as firewall, NAT, IDP, and so on. These services are offered by
instantiating service virtual machines to dynamically apply single or multiple
services to virtual machine (VM) traffic. It is also possible to chain physical
appliance-based services as well as to have multiple services to be chained
together. It should be noted, however, that a mix of VNF and PNF is not
supported today and there is no plan in the near future to do this.

Service-Chaining between a Source VN (SVN) and Destination VN (DVN) works by
re-originating interested DVN routes in to the SVN using a Service Instance
Interface (SVI) as the next-hop. This causes traffic to be steered through the
service chain as desired. These service-chain routes are classic inet[6] routes
that get replicated into bgp.l3vpn[6].0 table and then into remote inet[6]
tables using the bgp inet[6]-vpn address-family.

# 2. Problem statement

Today, Service-Chaining works between two VNs using L3VPN routes and MPLSoUDP/
MPLSoGRE encapsulations in the data plane. But this is limited only to traffic
between VMs. Traffic between VMs and a BMS cannot be steered through a service-
chain using EVPN as the underlying transport technology.

In this document, we propose a solution to enable service-chaining for traffic
flowing between BMS and VMs and vice-versa through one or more VNF SVIs. In
particular, we solve the problem of enabling inter-LR traffic from a BMS or VM
belonging to one VN (represented by LR1) and a BMS or VM belonging to another
VN (represented by LR2). In the context of VNs, this would mean that we will
facilitate traffic between two VN-Ints to traverse a chain of services. The
same solution can be extended, in theory, to support the case where
service-chaining needs to be enabled between an L3 VM not using LR and a BMS
using LR. The following cases will be considered

1. **VM-SI-BMS**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/VM-BMS-WithServiceChain.png?raw=true)

2. **BMS-SI-VM**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/BM-VM-WithServiceChain.png?raw=true)

3. **VM-BMS (no service-chain)**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/VM-BMS-NoServiceChain.png?raw=true)

4. **BMS - VM (no service-chain)**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/BMS-VM-NoServiceChain.png?raw=true)

5. **VM - VM** (inter-LR with and without service chain - this is technically
feasible since we are exposing the LR config to the user but we may choose to
disallow this mode)

![](https://github.com/maheshmns/contrail-specs/blob/master/images/VM-VM-NoServiceChain.png?raw=true)

![](https://github.com/maheshmns/contrail-specs/blob/master/images/VM-VM-WithServiceChain.png?raw=true)

# 3. Current Capabilities

# 3.1 Bare Metal Server Support

Contrail supports clusters having a BMS and other virtual instances connected
to a TOR switch via OVSDB in such a way that it can be configured to be part of
any of the VN configured in the cluster. This enables a BMS to communicate with
any of the virtual instances in the cluster. Further, the communication can be
controlled with Contrail policy configuration just like how one would control
the VM-VM communication. The solution was initially supported  by using OVSDB
configure the TOR switch and to import dynamically learned addresses from it.
Later, as indicated earlier, this solution has been replaced by EVPN (R5.01)
except for one customer in R5.x. Both inter-VN and intra-VN modes are
supported.

# 3.1.1 Control Plane

Support for EVPN Type-5 routing was added to be used for DCI and/or inter-subet
routing in R5.01. EVPN Route Type 5 messages are defined in the IETF
specification IP Prefix Advertisement in EVPN. EVPN Route Type 5 is an
extension of EVPN Route Type 2, which carries MAC addresses along with their
associated IP addresses. EVPN Route Type 5 facilitates in inter-subnet routing.

Control-node does route serving to type-5 EVPN routes similar to how it does
for all other routes. Type-5 routes when received from agent are processed and
always added to L3VRF.evpn.0 table with protocol XMPP. From here on-wards, this
route is replicated into bgp.evpn.0 table so that it is advertised to all BGP
peers as well.

When Type-5 EVPN prefix is received from a BGP peer, it is first installed into
bgp.evpn.0 like all other routes. From here, based on matching route targets,
the route gets replicated into all *.evpn.0 tables as applicable. From there,
the routes are advertised over Extensible messaging and presence protocol
(XMPP) to all interested agents.

# 3.1.2 Data Plane

Agent will rely on VXLAN routing (enabled on per project basis) to identify if
EVPN type-5 has to be enabled. This knob creates an internal VRF (here after
denoted as L3VRF) for each logical-router which will be used for routing across
all VN. The VN associated with L3VRF will have fixed VNI and not allow user to
configure the same. Encapsulation supported for type-5 will only be VXLAN for
unicast.  VXLAN encapsulation will be used in the data plane communication with
the TOR switch. The TOR switch acts as the Virtual tunnel endpoint (VTEP) for
the BMS Unicast traffic from the BMS is VXLAN encapsulated by the TOR switch
and forwarded, if the destination MAC is known within the virtual switch.
Unicast traffic from the VMs in the OpenContrail cluster are forwarded to the
TOR switch, where VXLAN is terminated and the packet is forwarded to the BMS.
Broadcast traffic from BMS is received by the TSN node and it uses the
replication tree to flood in the VN. Broadcast traffic from the VMs in the
cluster are sent to the TSN node, which replicates the packets to the TORs.

# 3.2 Service Chaining Basics

# 3.2.1 Definition and Capabilities

Services are offered by instantiating service virtual machines to dynamically
apply single or multiple services to virtual machine (VM) traffic. It is also
possible to chain physical appliance-based services.  Figure 1 shows the basic
service chain schema, with a single service. The service VM spawns the service,
using the convention of left interface (Left-IF) and right interface
(Right-IF).  Multiple services can also be chained together.

**Figure 2: Service Chaining**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/ServiceChaining.gif?raw=true)

Each Service VM is associated with two VRFs(VNs) with its Left-IF belonging to
the VM1’s VN and its Right-IF belonging to VM2’s VN. Traffic from VM1 arrives
at the Service VM on the Left-IF and the traffic from the Service VM is
forwarded out on the Right-IF towards VM2.  When you create a service chain,
the Contrail software creates tunnels across the underlay network that span
through all services in the chain. Figure 2 shows two end points and two
compute nodes, each with one service instance and traffic going to and from one
end point to the other. The service-chains are by default bi-directional in
that they provide the ability to steer traffic through a service-chain in both
directions (VM1 ⇒ VM2 and VM2 ⇒ VM1 in the above example)

**Figure 3: Contrail Service Chain**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/Contrail-Service-Chaining.gif?raw=true)


The following are the modes of services that can be configured.  Transparent or
bridge mode Used for services that do not modify the packet. Also known as
bump-in-the-wire or Layer 2 mode. Examples include Layer 2 firewall, IDP, and
so on.  In-network or routed mode Provides a gateway service where packets are
routed between the service instance interfaces. Examples include NAT, Layer 3
firewall, load balancer, HTTP proxy, and so on.  In-network-nat mode Similar to
in-network mode, however, return traffic does not need to be routed to the
source network. In-network-nat mode is particularly useful for NAT service.
The key to enabling a service chain out of heterogeneous network functions
(where some of them are virtual and others physical) is the abstraction
provided by the Virtual Machine Interface (VMI) and the port tuple object which
represents an ordered list of network interfaces connected to the same VM, or
container, or physical appliance. With its integration to routing policies, it
is also possible to filter and modify routes that are leaked through a Service
Chain.  Today, we only support service-chaining for communication between VMs
even though the  network functions themselves can be virtual or physical.

# 3.2.3 Service-Topology and Route Re-origination

The service-topology needed for enabling service-chaining between the VMs is
constructed using one or more service-chain routes from the service host, one
for each service VM attached to it. Based on the number of service-chain-routes
from each service host, we can construct multiple service-topologies for a
given VPN.

The service hosts themselves are each assigned an IP address and as mentioned
earlier are associated with a Left-IF belonging to the SVN and a Right-IF
belonging to the DVN. The IP reachability to the service hosts are advertised
using BGP L3VPN mechanisms.

The service-chain routes will have the Route-Target (RT) configured for the SVN
along with a label that maps to a VMI associated with the VN to which the
Left-IF belongs (SVN) to help direct the flow of traffic from other hosts into
the service host in the configured service-chain. On the other hand, routes
needed to reach the different service hosts in the service-chain are imported
and installed in the VN to which the Right-IF belongs (DVN).

Service-chaining works by originating routes (INET or INET6) from the DVN all
the way to the SVN through all the intermediate service-hosts. Among other
things, at each service-host, the service-chain routes are constructed by
modifying the route Next-Hop(NH) attribute which is made to point to its
Left-IF. Finally, the route is installed in the SVN with its NH pointing to the
first service-host in the service-chain.  Re-origination of routes is done by a
service-chain module which resides in each service-host.  This module is
responsible for re-originating interested routes from the next node in the
service-chain. The service-chain module in the last service-host (the one that
is connected to the compute hosting the destination VM) of the service-chain
will listen to routes from the DVN and re-originate them into its routing
table.  The previous service-host will steer these routes into its own routing
tables and so on. Finally the service-chain routes will be installed in the
first service-host (the one that is connected to the compute hosting the source
VM) of the service-chain.

# 3.2.4 Route Replication

The service-chain module, as mentioned, is responsible for re-originating
routes from the DVN into the routing tables of each service-host in the
service-chain.  In order to install these routes in the source VM, the route
replication module is used. This module is responsible for the following.
Replicating service-chain-routes from the first service-host in the
service-chain into the SVN routing table of the service-host to which the
source VM is attached.  Replicating routes from the SVN routing table into BGP
L3VPN routing table.  At the destination VM, it replicates routes from BGP
L3VPN routing table into the DVN routing table of the service-host to which the
destination VM is attached

# 3.2.5 Traffic Forwarding

When a VM in VN-Red wants to communicate to another VM belonging to VN-Green
via a service-chain, it originates data traffic towards the vRouter which acts
as the ingress Provider Edge (PE) router. The vRouter will in turn forward the
traffic towards the first service-instance in the service-chain. The first
service-instance could either be attached to the same compute where the vRouter
is hosted or be multiple hops away on a different compute node. As a result,
the vRouter will need to encapsulate the packets and send them over a tunnel
towards the compute node that hosts the service-instance.

The first service-instance in the service-chain path, on receipt of the
encapsulated data packets on the left-IF (which belongs to VN-Red),
decapsulates them, and forwards them to the Service-VM attached to it. Once the
Service-VM has serviced the packets, they are tunneled out of the node on the
right-IF(belonging to VN-Green) on to the compute node hosting the next
service-instance. The last service-instance (after servicing the packet at its
attached service-node) will forward the packet to the compute where the
destination is located (which can be on either the same compute or a different
one).

# 4. Proposed solution

# 4.1 Service-Topology and Route Re-origination
As explained in Section 3, service-chaining is currently enabled only for IPv4
and IPv6 address-families. In other words, we only re-originate prefixes in the
INET or INET6 address-families from DVN to the SVN. In essence, this only
enables support for allowing traffic to be steered through a service-chain
shown in Figure 4.

**Figure 4: Inter-VM Service Chain**

**VM <==> VM**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/Inter-VM-ServiceChain.png?raw=true)


With the proposed design, we intend to enable support for service-chaining for
the following cases as well (as shown in Figure 5 below). Note that any time a
BMS is involved, as explained earlier, an LR is created with an associated
internal VN (VN-Int). Route re-origination happens between the two internal VNs
(corresponding to the two LRs). In other words, SVN and DVN are the two
internal LR VNs.


**Figure 5: VM/BMS ←→  BMS/VM Service Chain**


**VM <==> BMS**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/VM-BMS-ServiceChaining.png?raw=true)

**BMS <==> BMS**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/BMS-VM-ServiceChaining.png?raw=true)

In order to achieve (1), the service-chain re-origination module will be
enhanced to do the following: VM → BMS: At the tail-end of the service-chain on
the BMS side (vRouter hosting the last service host in the service-chain):
Along with INET and INET6 address-families, the service-chain module will
re-originate DVN routes from the BMS as well. These routes manifest as EVPN
Type-5 routes in the control node (BMS → TOR →  TOR agent → Control Node) Note
that, there may be other EVPN route types that may be present in the DVN but it
is sufficient to only re-originate the Type-5 routes At the head-end of the
service-chain, on the VM side (vRouter hosting the first service host in the
service chain): When the EVPN Type-5 routes from the DVN are received, these
are installed in the SVN EVPN routing table.  In addition, these will also be
installed in the SVN INET or INET6 table depending on whether the payload
carried in the Type-5 route is INET or INET6 respectively.  At the intermediate
service hosts in the service-chain (vRouter attached to the service VMs): The
service-host is intended to be an IP-fabric and does not support EVPN.  Hence,
there is no change in functionality here in that they still are assigned an
INET or INET6 prefix and these routes are advertised through L3VPN.

BMS → VM: At the tail-end of the service-chain on the VM side (vRouter hosting
the last service host in the service chain): The service-chain module’s
existing functionality is sufficient here since we will need to re-originate
routes belonging to INET or INET6 address-family At the head-end of the
service-chain, on the BMS side (vRouter hosting the first service host in the
service-chain): When the INET/INET6 routes from the DVN are received, these are
installed in the SVN INET[6] routing table.  In addition, these will also be
installed in the SVN EVPN table depending as an EVPN Type-5 route.  At the
intermediate service hosts in the service-chain (vRouter attached to the
service VMs): The service-host is intended to be an IP-fabric and does not
support EVPN.  Hence, there is no change in functionality here in that they
still are assigned an INET or INET6 prefix and these routes are advertised
through L3VPN.


In order to achieve (2), the service-chain re-origination module will be
enhanced to do the following: At the tail-end of the service-chain on the BMS
side (vRouter hosting the last service host in the service-chain): Along with
INET and INET6 address-families, the service-chain module will re-originate DVN
routes from the BMS as well. These routes manifest as EVPN Type-5 routes in the
control node (BMS → TOR →  TOR agent → Control Node) Note that, there may be
other EVPN route types that may be present in the DVN but it is sufficient to
only re-originate the Type-5 routes At the head-end of the service-chain, on
the BMS side (vRouter hosting the first service host in the service-chain):
When the EVPN Type-5 routes from the DVN are received, these are installed in
the SVN EVPN routing table.  At the intermediate service hosts in the
service-chain (vRouter attached to the service VMs): The service-host is
intended to be an IP-fabric and does not support EVPN.  Hence, there is no
change in functionality here in that they still are assigned an INET or INET6
prefix and these routes are advertised through L3VPN.

The service-chain module: The service-chain module will need to adapt to two
different address-families: At the service-host connected to the destination
VM, it may need to re-originate Type-5 routes along with INET and INET6 routes
from the DVN into its routing table At the service-host connected to the source
VM, it will also track connected routes (route to the left IF) in order to
manipulate the NH  to point to its Left IF.

# 4.2 Route Replication

There is no enhancement needed in the route-replication module in order to
support the new requirements. It will continue to replicate service-chain
routes into the SVN routing table at the source and BGP L3VPN routes into the
DVN at the destination and is independent of the address-family that is
configured/enabled.

# 4.3 Traffic forwarding

In the proposed design, at the head-end, service-chain routes are re-originated
not only to the VN-RI.inet.0 table, but also into VN-RI.evpn.0 table as Type-5
route. While doing so, next-hop is set as the associated compute that hosts the
SI (left) interface and VXLAN VNI as the L3 label associated with the Logical
Router’s InternalVirtualNetwork (IVN). This route is installed into the table
VN-RI.evpn.0 and then gets replicated into bgp.evpn.0 table. From here onwards,
they are advertised to remote bgp evpn peers including TORs.

**Figure 6: Traffic flow from BMS → VM via Service Chain**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/TrafficFlow-BMS-VM.png?raw=true)


Figure 6 shows the flow of traffic from a BMS to VM via a service-chain. TOR
VTEP routers which connect to BMSes are to be configured with appropriate
route-target in order to attract the Type-5 Service-Chain routes, either using
the auto-generated route-target (8000000+) or with an explicit route-target
configured for the associated InternalVirtualNetwork. Once these routes get
imported into appropriate inet[6] tables in the TOR routes, all BMS to VN
traffic that hits this route get sent to the compute that hosts the left
interface of the SVI with the VXLAN that points to the L3-VNI of the
InternalVirtualNetwork associated with the left LR.

From here, it should be business as-usual and the packets traverse from left
SVI through the VNF to right SVI, eventually to VM using MPLSoUDP or MPLSoGRE
encapsulation setup using L3VPN routes. Note that ECMP will work here as well
(where the SIs are horizontally scaled with multiple SIs available for the same
service). This is because, each ECMP route is carried as a separate EVPN type-5
prefix (with different RD).

In the reverse direction (shown in Figure 6), when traffic originates in VMs
and are destined to BMSes, there won’t be any route towards BMS as it is. Hence
control-node service-chain module, upon receiving EVPN Type-5 route from the
remote TORs in red.evpn.0 should re-originate a new service-chain path into
red.inet.0 table with remote TOR as the next-hop and VXLAN as the encapsulation
type with associated L3 VNI. This route should then get replicated by
service-chain module using normal service-chain mechanisms. (Using MPLS over
UDP as the tunnel encapsulation)

**Figure 7: Traffic flow from VM → BMS via Service Chain**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/TrafficFlow-VM-BMS.png?raw=true)


When traffic is originated from VM towards BMS, traffic should first get
steered to the left interface of the SI. When the traffic eventually comes out
of the last SI’s right interface at the egress, vRouter does the lookup for the
destination BMS address. Next-hop should point to TOR that is connected to the
BMS with appropriate VXLAN (L3 VNI) and hence packets should be sent to the TOR
using associated L3 VNI as VXLAN encapsulation id.

This strategy of re-originating evpn routes into l3vpn (and vice-versa) should
also work when traffic between BMS (no VM) needs to be steered through the
service chain of VNFs. In this case, in both directions, BMS are learnt as EVPN
type-5 routes. These routes are re-originated into l3vpn[6] address family so
that VNFs are chained using l3vpn routes (mpls over udp) using existing
chaining mechanisms. Traffic from BMS is first sent the compute node that hosts
the first VNF in the service chain. Here, packet lookup happens in the VXLAN
table (based on L3 VNI VXLAN id) first. The NH on the route should point to the
left interface of the VNF (as re-originated by service chain l3vpn routes).
From here onwards, packets should go through the chain and eventually come out
of the right interface of the last VNF in the service chain. At the egress
node, table lookup should match the evpn type-5 routes.

# 4.4 Configuration (schema-transformer)

Whenever a Logical-Router (LR) is created, schema-transformer automatically
creates a first-class virtual-network object called “InternalVirtualNetwork”
(IVN) which is associated with this LR. This virtual-network is what is used to
carry all EVPN Type-5 prefixes. In order to connect two LRs managed through
EVPN through a service-chain, one needs to be able to configure a
network-policy inter-connecting the corresponding IVNs. This implies that this
hidden auto-created IVN needs to be exposed to users for further “selective”
configuration. Since there are many variations of LR use cases, we should allow
only those features for user configuration that are fully understood and well
tested.

# 4.5 Contrail vrouter-agent/vRouter

In the current Agent-vRouter EVPN type-5 implementation when VXLAN routing is
enabled, VN-Int (VRF-Int) is created for every LR in the project with VNI
associated with the LR configuration.

The VRF-Int inet table has the all of the inet/inet6 routes of the VRFs
associated with the LR. Default route is installed in the regular VRFs and
pointing to VRF-Int. EVPN Type-5 routes are advertised to the Control-node from
the VRF-Int’s EVPN table. The Agent will also copy the inet prefix in the
VRF-Int EVPN table to its inet /inet6 tables.

**Traffic forwarding**

**VM to BMS**: At the tail-end of the service-chain on the BMS side (vRouter
hosting the last service host in the service-chain): Control node sets the
nexthop to VXLAN tunnel in the last service host in the service-chain
INET/INET6 table At the head-end of the service-chain, on the VM side (vRouter
hosting the first service host in the service chain): Agent VRF-Int will have
the originated DVN Type-5 routes in the EVPN table and INET prefix copied to
respective INET/INET6 tables.  When the VM sends packet with vhost MAC and BMS
destination IP, VRF is looked in. The VRF default route NH is pointing to
VRF-Int., VRF-Int INET table will have route to the head of the service-chain
host and encap NH found. Rest is similar to encap NH processing.  At the
tail-end of the service-chain on the BMS side, during the re-origination of DVN
to SVN routes, its VRF Inet table will set to tunnel NH. When lookup is done in
the tail-end of the service-chain VRF, it will point to tunnel NH and packet is
sent with VXLAN encapsulation.

**BMS to VM**: At the head of the service-chain on the BMS side (vRouter
hosting the first service host in the service-chain): Control node sets the
nexthop to encap in the INET/INET6 table

BMS sends the tunnel NH packet with VXLAN encapsulation. Since the VNID carried
is of VN-Int, lookup is done in the VRF-Int. VRF-Int Inet table NH points to
encap and rest is similar to encap NH processing.

**VM to VM**: At the head of the service-chain (vRouter hosting the first
service host in the service-chain): Control node sets the nexthop to encap in
the INET/INET6 table

At the tail-end of the service-chain (vRouter hosting the last service host in
the service-chain): Control node sets the nexthop to VXLAN tunnel in the last
service host in the service-chain INET/INET6 table

VM sends the tunnel NH packet with VXLAN encapsulation. Since the VNID carried
is of VN-Int, lookup is done in the VRF-Int. VRF-Int Inet table NH points to
encap and rest is similar to encap NH processing.

At the tail-end of the service-chain, during the re-origination of DVN to SVN
routes, its VRF Inet table will set to tunnel NH. When lookup is done in the
tail-end of the service-chain VRF, it will point to tunnel NH and packet is
sent with VXLAN encapsulation.

For the inter-VN traffic forwarding vRouter Agent lookups are done in the
VRF-Int Inet table. From the initial scoping effort the existing EVPN type-5
implementation in vRouter agent should support EVPN service-chaining proposed
in the document.

Coverage and effort in vRoute agent unit testing might be required for this
feature.

Introspect and debugging Following debug tools provide forwarding debug: a)
vxlan --dump b)   vif --list c)    rt --dump d)   Agent (port 8085) introspect

Note: During testing and debug phase might warrant for additional debug
information and logs.

# 4.6 UI (GUI, Provisioning, etc.)

Web GUI needs to expose IVN of the LRs so that they can be configured for
service-chaining via network-policies Service-policy configuration should be
seam-less in order to configure route-targets correctly in the TORs

User Workflow: The user will create Network Policy using a service instance to
interconnect (1) MUST Logical Router LR1 to LR2 via Service Instance SI1 (2)
SHOULD : Virtual Network VNI1 to Virtual network VNI2 via Service Instance SI1

**Figure 8: UI Workflow**

![](https://github.com/maheshmns/contrail-specs/blob/master/images/UI-Workflow.png?raw=true)


Changes to the existing screen:

UI will follow the existing logic for policies. Screenshot above is up to date.
Other changes needed:
- Logical Routers with logical_router_type = snat-routing should be filtered
  out.  Only those LR with logical_router_type = vxlan-routing should be
displayed.
- If an LR, say LR1 is selected as source, LR1 should not be allowed to select
  in Destination.
- We can use the same LRs for a different chains.
- Should we allow to interconnect LR1 to LR2 and LR2 to LR1 to the same
  service?  Yes, if the flow is unidirectional. Just follow the existing
workflow.
- LR should be grayed out in a dropdown if we for example select a Protocol or
  any other field which is forbidden for LRs.
- We allow to update a policy. Just follow the existing logic.
- We should change the order of fields : Suggested order -  Source Type,
  Source, Source Port, Direction, Destination Type, Destination, Destination
Ports, Protocol, Action, Advanced

# 4.7 Deliverables

The first phase of this feature is targeted to be released in R1911. As part of
this deliverable the following will be implemented:

- EVPN service-chaining support for inter-LR routing
  -  LR will be exposed to UI to be added as the src/dest of the network
     policies that are used to configure service-chains
  -  LR to non-LR MUST NOT be allowed
  -  Inter-LR communication MUST be via service-chains.
- The supported inter-LR service-chaining scenarios are
  -  VM -> SC -> BMS
  -  BMS -> SC -> VM
  -  VM -> SC -> VM
  -  BMS -> SC -> BMS
- Intra-LR service chaining WILL NOT be supported


# 5. Implementation

# 5.1 Control-node changes

In this section, we provide details on the implementation of a service-chain in
the control-node and then describe the enhancements required to support the new
cases.

# 5.1.1 Inter-VN and Service-Chaining Implementation

Within a VN, all endpoints are allowed to talk to each other (intra-VN).
However, if we wish to enable inter-VN communication, say between VM1
(represented by VMI1) belonging to VN Red and VM2 (represented by VMI2)
belonging to a VN Green, a network policy is instantiated to allow traffic to
flow from VM1 to VM2 and vice-versa as shown in Figure 9 below.

![](https://github.com/maheshmns/contrail-specs/blob/master/images/InterVNWithNetworkPolicy.png?raw=true)

**Figure 9: Inter-VN Communication using Network Policies**

As far as implementation is concerned, this can be accomplished by leaking
routes between the VN’s Routing Instance (RI) or routing table. To achieve
this, Schema Transformer (ST) creates a link between the RIs of the two VNs.
This, in turn, results in the RIs importing and exporting each other’s Route
Targets (RTs) and hence each other’s routes. Traffic can now flow between the
VN Red and VN Green.

Now, in order to allow communication between the VNs but require that traffic
between the VNs go through, for instance, a firewall, we’ll need to create a
service-chain by enabling a Service Instance (SI) (that hosts the firewall) and
directing all traffic between the VNs to traverse the service chain as shown in
Figure 10.

![](https://github.com/maheshmns/contrail-specs/blob/master/images/ServiceChainArchitecture.png?raw=true)

**Figure 10: Service Chain Implementation Details**

In this case, there is no link created between the VNs. Instead, as shown in
the Figure, in addition to the default RI, ST also creates one or more
Service-RIs (S-RIs) in each VN. The number of S-RIs in the VN will correspond
to the number of SIs in the service-chain. In the above example, since we only
have one SI (for enabling a firewall) in the service-chain there will only be
one S-RI (RIred’ for VN-Red and RIgreen’ for VN-Green) created for each VN. To
this S-RI, it also attaches a Service Chain Info (SCI) property which contains
a bunch of information including the Source RI, Destination RI and the SI
(next-hop IP address of the Left IF) to be traversed among other things. The
purpose of the SCI is to enable control-node to re-originate all routes from
the DVN into the SVN RI and steer traffic from the SVN to the DVN through the
service-chain.

Based on the above information from ST, control-node creates a service-chain
and sets up a watcher for routes in the destination RI. In addition, in order
to manipulate the NH for the leaked routes, it also watches for the connected
routes (routes advertised by the vRouter hosting the SI) to get the IP address
of the S-RI. Using the re-origination logic described in Section 3.2.3 routes
from  RIgreen (routes belonging to VMI2 in VN-green) are inserted in to the
S-RI corresponding to the of the SI (RIred’). Notice from the Figure that there
is a link enabled between RIred and RIred’. As explained in Section 3.2.4, this
enables route-replication in the control node and hence resulting in the routes
in RIred’ to be replicated to the source RI (RIred). The same logic works in
the reverse direction as well where traffic from VN-Green to VN-Red needs to be
steered through a firewall. If there is more than one service in the
service-chain, new SIs (and hence new S-RIs) will be created by ST and enabled
in control-node.

We can also enable auto-scaling of SIs (for instance having multiple firewalls
each one on a different VM, and load-balancing the traffic between VN-Red and
VN-Green across those firewalls). In this case, we’ll have multiple Left and
Right IFs for each SI. In order to do this, a logical IP is allocated by the
Service Monitor (SM) for the service-chain and this IP is added to the SCI. All
the VMs hosting the firewall will advertise two routes for its Left IF (both
physical and logical IP). For control-node, this will boil down to having ECMP
routes for the connected table with appropriate NHs. Traffic from the SVN will
utilize ECMP to be steered to one of the VMs hosting the firewall based on a
per-flow hash.

# 5.1.2 Data structures

The service chaining implementation follows a template model based on the
address-family for which it is enabled. All classes implementation the
functionality are templatized per address-family. Currently, only INET and
INET6 address-families are supported.

At the core of the implementation is the ServiceChainMgr class, which is
responsible for processing the ServiceChainConfig and create, manage and delete
service-chains (represented by the ServiceChain class). ServiceChainConfig is
added as a property of the RI (represented by the RoutingInstance class).  All
the individual service-chains that are part of the same logical service chain
in the configuration are grouped together in a ServiceChainGroup class. The set
of member chains is tracked using RoutingInstance pointers (as opposed to
ServiceChain pointers) so that the state of pending chains can also be tracked.
With the proposed design, a new template instantiation will be defined to
accommodate EVPN address-family.

    template class ServiceChainMgr<ServiceChainEvpn>;

    class ServiceChainEvpn : public ServiceChainBase< EvpnRoute, InetVpnRoute,
EvpnPrefix, IpAddress> { };

    template <typename T1, typename T2, typename T3, typename T4> struct
ServiceChainBase { typedef T1 RouteT; typedef T2 VpnRouteT; typedef T3 PrefixT;
typedef T4 AddressT; };



When a new ServiceChain is created, two match conditions are added; one on the
source table (for tracking connecting routes to the SIs in the chain) and
another on the destination table (for re-originating routes from the DVN). The
underlay is typically an IP fabric that can support only INET or INET6.
Currently, for INET and INET6 service-chains, this is not a problem and both
the connected table and the destination table would belong to the same
address-family. For enabling an EVPN based service-chain, the destination table
and the source table belong to EVPN address-family but the connected table
still remains INET or INET6 (depending on the payload carried in the EVPN
packet and indicated by the service_chain_addr in the ServiceChainConfig).
Changes will be done to accommodate the new behavior for EVPN service-chains.

    // Get the BGP Tables to add condition
    BgpTable *connected_table = NULL; if (GetFamily() == Address::EVPN) {
	// For EVPN, connected table will be INET/INET6 depending on whether
	// service_chain_addr is v4/v6.
	IpAddress sc_addr = chain_addr; if (sc_addr.is_v4()) { connected_table
= connected_ri->GetTable(Address::INET); } else { connected_table =
connected_ri->GetTable(Address::INET6); } }



# 5.2 Schema changes

During the creation of service info, if the schema transformer finds that any
of the connected virtual network is an InternalVirtualNetwork, then it will
need to create a new Service Info exclusively for Logical Router Service
Chaining.

This new Service info will be similar to the existing service info, but with
both the IPV4 and IPV6 information prepended with a “evpn” string. The schema
transformer will also do certain back end checks like verifying that both the
ends of the service chain are only a Logical Router.

The schema change for the new EVPN service chain config is shown below.

<xsd:element name='evpn-service-chain-information' type='ServiceChainInfo'/>
<!-- #IFMAP-SEMANTICS-IDL Property('evpn-service-chain-information',
'routing-instance', 'system-only', 'CRUD', 'Internal service chaining
information, should not be modified.') -->

Further, currently, schema only exposes a VN object as a possible choice for
being the source and target for a service-chain. Since this feature will be
enabling service-chaining for LRs too, the corresponding schema object needs to
be enhanced to include LR also as a choice for being the source/destination of
a service-chain as shown below.

<xsd:complexType name="AddressType"> <!-- Deprecated in favor of subnet-list
--> <xsd:element name="subnet" type="SubnetType"           required='exclusive'
description='Any address that belongs to this subnet'/> <xsd:element
name="virtual-network" type="xsd:string"  required='exclusive' description='Any
address that belongs to this virtual network '/> <xsd:element
name="logical-router" type="xsd:string"  required='exclusive' description='Any
address that belongs to this logical router virtual-network'/> <xsd:element
name="security-group"  type="xsd:string"  required='exclusive' description='Any
address that belongs to interface with this security-group'/> <xsd:element
name="network-policy"  type="xsd:string"  required='exclusive' description='Any
address that belongs to virtual network which has this policy attached'/>

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

Unite test cases will be added to check for different scenarios when a router
acts as a route reflector.

# 10. Documentation Impact

# 11. References

