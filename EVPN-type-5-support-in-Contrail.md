# Introduction:

EVPN type 5 route that is proposed in [1] is a way to carry IP prefixes in BGP
EVPN based control plane. Regular type 2 EVPN carries MAC addresses along with
it is associated IP addresses. In use cases like Data-center Interconnect (DCI),
this would not be scalable. What is needed is a way to advertise just IP
prefixes without bothering about the MAC addresses. EVPN type 5 achieves this.
Another advantage of this is that it also facilitates inter-subnet routing
using EVPN when networks need it.

# Problem Statement:

Support and handle EVPN type 5 messages in contrail. Traditionally, contrail
controller used to support EVPN for L2 routes only through VXLAN encap. So it
cannot be used for use-cases like DCI and inter-subnet routing. Also, QFX
switches started supporting EVPN type 5 routes in junos 17.1 software. Hence we
need to support this feature in contrail.

# Type 5 NLRI

    +---------------------------------------+
    |      RD   (8 octets)                  |
    +---------------------------------------+
    |Ethernet Segment Identifier (10 octets)|
    +---------------------------------------+
    |  Ethernet Tag ID (4 octets)           |
    +---------------------------------------+
    |  IP Prefix Length (1 octet)           |
    +---------------------------------------+
    |  IP Prefix (4 or 16 octets)           |
    +---------------------------------------+
    |  GW IP Address (4 or 16 octets)       |
    +---------------------------------------+
    |  MPLS Label (3 octets)                |
    +---------------------------------------+

# Work items

## Agent

### Config
Agent will rely on vxlan-routing (enabled on per project basis) to identify if
EVPN type-5 has to be enabled. This knob creates an internal vrf (here after denoted as L3VRF) for each logical-router which will be used for routing across
all VN. The VN associated with L3VRF will have fixed VNI and not allow user to
configure the same. Encapsulation supported for type-5 will only be VXLAN for
unicast.

### Routes
L3VRF will be a regular VRF in oper, with following characteristics:

* All other VRF will export route(inet/inet6) routes to L3VRF evpn table with
  vhost mac address.
* Export of routes to CN from L3VRF will be done for evpn table as type 5 route.
* Other VRF will install default routes and host routes pointing to L3VRF.
* L3VRF will internally push the inet prefix from its evpn table to its own
  inet/inet6 table. Vrouter will use these inet tables for routing which is
  similar to L3vpn routing.

### ARP
This will continue to be similar to current behavior where every other vrf
(except L3VRF) will have IP-Mac binding to reply for arp. In L3VRF IP-MAC
binding will be absent as no arp reply is required from same and proxy will be
enabled.

### Nexthop
Non L3VRF will have all inet/inet6 routes pointing to L3VRF table nexthop. L3VRF
routes will use three types of nexthop:

1. Tunnel Nexthop - This will be regular tunnel nh with rewrite mac of
   destination vrouter mac.
2. EVPN type-5 encap NH - This is a new type of L3 encap nexthop which will be
   used to point to VMI with rewrite mac of VMI. The mac will be retrieved from
   IP-Mac binding present in source vrf inet route.
3. L3 VRF NH - This will make sure that packets routed via L3 VRF use its VNID
   and on destination lookup is done for inet tables in L3VRF instead of bridge
   table.

### Example
#### Subnet routing
Say there are two VM in same VN-1 - VM-A(1.1.1.2) and VM-B(2.2.2.2)
* VM-A to reach VM-B will request for arp of gateway.
* Vrouter responds with proxy vrouter mac as it has to be routed.
* VM-A sends packet with vhost mac and VM-B dest ip. VRF-1(mapping to VN-1) is
  looked in and route for 2.2.2.2 is found. It is pointing to table NH
  i.e. L3VRF. Lookup is again done in L3VRF inet table which will have route
  for 2.2.2.2 pointing to tunnel/encap NH.
* If its tunnel NH packet is sent with VXLAN encapsulation to destination
  compute and lookup is done again at the same. Since the VNI carried is of
  L3VRF, vrouter will see VNI pointing to L3 VRF NH. Lookup is done in inet
  table of L3VRF and encap NH is found. Rest is similar to encap NH processing.
* If its encap NH, destination mac of VM-B is retrieved from same and packet
  rewritten with it and delivered to VM-B.

#### Inter-VN routing
VM-A belongs to VN-1 and VM-B belongs to VN-2.
VN-1 and VN-2 export their inet/inet6 routes to L3VRF.
VM-A when trying to reach VM-B will hit gateway. After that it is similar to
subnet routing explained above. Scope of VN-1 and VN-2 ends at source. Rest all
is done in L3VRF.

#### Bridging
This continues to be same as before. VM-A and VM-B are in same subnet in VN-1.
VM-A will request for VM-B arp and it will be responded by vrouter from VN-1
inet table. Then the whole forwarding will happen via bridge table. Hence evpn
type-5 changes will not impact the same.

**Note: Configuring L3VPN and EVPN type-5 together will result in undefined
        behavior as this is a mis-configuration **

## vRouter

### VNID
Vrouter will have to distinguish L2 VNID vs L3 VNID as the look-ups are in
different table. This will be done using type of VRF NH, VNID will be pointing
to.

### Nexthop
Inner destination mac has to be retrieved from new encap NH (as mentioned in
agent changes). Its required as scope of mac address ends at compute and other
computes wont know about it. Only IP is exchanged across computes.

Tunnel NH rewrite information will be with vhost mac as dest mac.

## Control-Node
Control-node does route serving to type-5 evpn routes similar to how it does
for all other routes. Type-5 routes when received from agent are processed and
always added to L3VRF.evpn.0 table with protocol XMPP. From here on-wards, this
route is replicated into bgp.evpn.0 so that it is advertised to all bgp peers
as well.

When Type-5 evpn prefix is received from a bgp peer, it is first installed into
bgp.evpn.0 like all other routes. From here, based on matching route targets,
the route gets replicated into all L3VPN.evpn.0 tables as applicable. From there
the routes are advertised over xmpp to all interested agents.

Note: In Release 5.0, policy based route-leaking among different L3VRFs are not
      supported. Hence Service-Chaining for Type 5 L3VRFs is also not supported.

### Encoding

Today, evpn routes are always injected with Type 2 (MacAdvertisementRoute) as
the evpn route-type. These are for mac and host-ip /32 routes. We can use the
same encoding to send and receive type-5 routes as well. This can be indicated
using all-0s mac address. This ensures that there is no schema change and hence
simplifies backward compatibility.

autogen data structures can used as is (because there is no schema change)
using fields as marked below, in nlri and nexthop data structures.

ESI based overlay index is not used for forwarding. l3_label advertised by
the agent is expected to be the vxlan table label for the L3VPN.inet.0 or
L3VPN.inet6.0 as applicable. (???)


```
struct EnetAddressType : public AutogenProperty {
    EnetAddressType();
    virtual ~EnetAddressType();
    int af;              // BgpAf::L2Vpn
    int safi;            // BgpAf::Enet
    int ethernet_tag;    // 
    std::string mac;     // All 0s mac address
    std::string address; // Any IPv4/IPv6 prefix (need not be /32 host route)
};

struct EnetNextHopType : public AutogenProperty {
    EnetNextHopType();
    virtual ~EnetNextHopType();
    int af;
    std::string address;
    std::string mac;
    int label;
    int l3_label;
    EnetTunnelEncapsulationListType tunnel_encapsulation_list;
    EnetTagListType tag_list;
};
```

### Introspect and debugging

Since no additional data structures are introduced, all information necessary
should be available via already existing L3 tables related pages.

# Proposed solution
TBD

# API schema changes
TBD

# User workflow impact
TBD

# UI changes
TBD

# Testing
TBD

# References

1. [evpn draft] (https://tools.ietf.org/html/draft-ietf-bess-evpn-prefix-advertisement-08)
