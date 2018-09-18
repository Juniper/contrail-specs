# 1. Introduction
Contrail support for Inter AS Option C (3 Label)
(https://github.com/Juniper/contrail-specs/blob/master/5.0/inter_as_opt_c.md)

# 2. Problem statement
The larger use case is to virtualize the CPE and offer services in the DC. The
connection between the DC and customer premise is over multiple AS'es, and they
have to rely on a transit provider. Service providers may have to use different
providers to reach their Residential and Commercial customers.
In short multiple vendors become involved and lining up the tunneling
technologies across these vendors becomes challenging.
MPLS however is deployed widely and MPLS's overhead is much lower compared to
other encapsulations.
Further, in order to decrease their reliance on the transit provider, providers
want to avoid having the transit provider participate in the control plane such
as in Inter-AS option A and B.
For these reasons, some providers prefer L3VPN Inter-AS Option C.

# 3. Proposed solution
The following diagram shows the network connections and roles of different
components in the solution.
<img src="images/inter_as_option_c_solution.png">
The controller maintains an eBGP(L3VPN AFI/SAFI enabled) session with the SDN-GW
router and iBGP(IPv4 LU AFI/SAFI enabled) session with the ASBR router.
Additionally, in order to avoid introducing MPLS in the fabric and supporting
MPLS tunnels on the vRouter we will continue to use IP(UDP/GRE) tunnels between
the vRouters and the ASBR(s).

The controller exchanges labeled routes with the vRouters over XMPP.

The vRouter uses MPLSoUDP or MPLSoGRE to reach the ASBR and encapsulates two
labels within it - the inner VPN label and outer BGP-LU label.
For the opposite direction, the vRouter will advertise a labeled unicast route
for its vhost address with a label 3 (implicit null), so effectively there will
be only a VPN label in the traffic from the ASBR delivered via the fabric to
the vRouter over an UDP/GRE tunnel.

The only drawback/compromise in this solution is that the MTU overhead due to
using IP tunnels in the fabric reduces the overall MTU end to end.

## 3.1 Alternatives considered
### 3.1.1 Implement BGP on vRouter
Implement BGP on vRouter. Peer the ToR with every vRouter (in that rack).
Operationally, the administrator will have to configure the ToR to peer with
every vRouter in that rack.
CLOS fabric will also do MPLS forwarding.
Every vRouter has to advertise BGP LU route for itself (label 3 - implicit null).
Run BGP LU only between ASBR and spine, spine and leaf, leaf and vR.
This option involves fair amount of new code for us as well as additional
operational tasks for customer (BGP peering on ToR with vRouter - configuring
and monitoring those sessions).

### 3.1.2 Run BGP from Controller to every ToR.
Run BGP from Controller to every ToR.
XMPP from Controller to vRouter (as usual).
Carry the BGP routes and advertise them into XMPP to the vRouter. vRouter learns
the BGP-LU routes using XMPP. Instead of running BGP-LU directly to the ToR.
Both ToRs will advertise different labels for the same BGP-LU destination. No
global coordination of labels.
Controller has to know which vRouter is behind which ToR and advertise only the
corresponding ToR's routes to the corresponding vRouters. Therefore, Controller
will have to maintain a RIB per ToR.
vRouters attached to a ToR subscribe to the Routing Table corresponding to that
ToR.
vRouter will have to somehow be informed or know who its ToR is
(during provisioning). And vRouter will subscribe to the corresponding Routing
Table on the Controller.
Operationally, this option is less work for the administrator however fairly
challenging for us in terms of implementation (maintaining per-ToR routing table
on the Controller, etc).

## 3.2 API schema changes
"inet-labeled" (iBGP-LU) as an address family needs to be supported in the UI
for adding BGP peers to the Controller. This is because the SDNGW as depicted
in the attached diagram will be an iBGP-LU peer for the Controller.
Additionally, we will need a per-neighbor, per address family  knob to control
default encap for vpn routes, in this configuration we will need it to be MPLS.
The knob will be a Default Tunnel Encapsulation List that will allow the user
to specify an ordered set of encapsulations, for received routes this list will
be used to assume the tunnel encapsulation to use if they are not specified in
the route, for routes advertised by the controller the list will be used to set
the tunnel encapsulation to be used for the route.
Provisioning of the BGP-LU peering, and the Default Tunnel Encapsulation List
with supported encapsulations per address family should be  exposed via the
corresponding RESTful API.

## 3.3 User workflow impact
TBD

## 3.4 UI changes
The BGP labeled inet address family should be configurable on the peer.
The Default Tunnel Encapsulation List, with supported encapsulations, should be
configurable per address family, preferrably only for VPN address families.
Visibility of the flows that are encapsulated by two tags should be provided in
the analytics engine.

## 3.5 Notification impact
TBD

# 4. Implementation
## 4.1 Work items
### 4.1.1 Controller
A new inet.3 table will be added to the master instance by default. This table
will hold the labeled unicast routes.
Schema changes will be made to support:
1) The BGP-LU address family.
2) A Default Tunnel Encapsulation List(ordered list of supported tunnel
encapsulation by desired preference)  knob per peer, per address family to
control the default vpn route encapsulation. For this use case we want to set
the the Default Tunnel Encapsulation List to contain, preferrably, only MPLS
under the vpn address family.

The Default Tunnel Encapsulation List will specify that we need to set the
tunnel encapsulation to when not specified in the VPN route received from the
SDN-GW. For VPN routes advertised by the vRouter the same Default Tunnel
Encapsulation List will be used to set the encapsulation (to MPLS in this use
case) when advertising the route.
Note, although the Default Tunnel Encapsulation List is meant to be used only
or vpn address families it can be easily supported for the labeled inet address
family.
The vRouter agents will be responsible for resolution of the VPN routes over the
IPv4 Labeled unicast routes. the IPv4 labeled unicast routes in turn would need
to have been resolved over the IP tunnels in the fabric.
Local vRouters' agents will learn each other's vhost routes in the new IPv4
labeled unicast address family. However, since the VPN route encapsulation
learnt over these will continue to have the default encapsulation they should be
directly resolved over the IP tunnels, the IPv4 labeled unicast routes will not
be used.
The controller will encode/decode the new address family/ sub address family to
and from BGP peers and Agents (XMPP).

### 4.1.2 Agent
Agent exports vhost route with label 3 (implicit NULL) so that received
packets processing is similar to current tunnelled packets as ASBR sends single
labelled packet.
Agent receives VPN routes in regular VN with nexthop as ASBR and label sent by
SDN-GW. Agent also receives IPv4 labeled route for ASBR in fabric VRF
(default-domain:default-project:ip-fabric:__default__).
A new inet.3 table is added to maintain labelled unicast routes in farbic VRF.
This table is used for resolving L3VPN routes with encapsulation type set to
MPLS and table entries are not programmed on vrouter.
A new nexthop class Labelled Tunnel NH is derived from Tunnel NH nexthop class
to maintain label information.
When agent receives l3vpn route with encapsulation type MPLS, it queries inet.3
table in fabric VRF to find the nexthop and which in turn relies on inet table
in farbic VRF to resolve the nexthop.

### 4.1.3 vRouter
vRouter should support stack of MPLS labels for sending tunnelled packets.
it retrieves inner label from route entry and outer lable from next hop entry.
### 4.1.4 UI
### 4.1.5 Analytics

# 5. Performance and scaling impact
## 5.1 API and control plane
#### Scaling and performance for API and control plane

## 5.2 Forwarding performance
#### Scaling and performance for API and forwarding

# 6. Upgrade/HA
The implementation accounts for ASBR redundancy. As well as for the SDNGW
redundancy. All regular Controller HA requirements are also supported.

# 7. Deprecations

# 8. Dependencies

# 9. Testing
## 9.1 Unit tests
### 9.1.1 Controller
End to end integration test for IPv4 labeled unicast routes
(AgentA-BgpRouterX-BgpRouterY-AgentB)
End to end integration test for IPv4 unicast vpn routes
(AgentA-BgpRouterX-BgpRouterY-AgentB) that verifies the behaviour of a Default
Tunnel Encapsulation List configured on BgpRouterX towards BGPRouterY.

## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References
TBD
