# Introduction:

EVPN type 5 route that is proposed in [1] is a way to carry IP prefixes in BGP
EVPN based control plane. Regular type 2 EVPN carries MAC addresses along with
their associated IP addresses. In use cases like Data-center Interconnect (DCI),
this would not be scalable. What is needed is a way to advertise just IP
prefixes without bothering about the MAC addresses. EVPN type 5 achieves this.
Another advantage of this is that it also facilitates inter-subnet routing
using EVPN when networks need it.

# Problem Statement:

Support and handle EVPN type 5 messages in contrail. Traditionally, contrail
controller used to support EVPN for L2 routes only through VXLAN encap. So it
cannot be used for use-cases like DCI and inter-subnet routing. Also, some
vendors such as JUNOS (QFX) switches started supporting EVPN type 5 routes in
their recent releases. Hence we need to support this feature in contrail as
well.

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

## Config

### Schema Change [2]

In current schema, a link named "logical-router-gateway" already exists between
logical-router and virtual-network. To ensure backward compatibility, this link
is simply extended with an attribute with particular enumerations with values
"ExternalGateway" and "Layer3VirtualNetwork", with ExternalGateway as the
default value. Please refer to [2] for additional details regarding the changes
to the schema file.

Also, a new link virtual-network-logical-router is added to quickly find out the
parent logical-router from any virtual network.

To ensure backward compatibility, contrail-api-server is expected to update the
database with new links (from virtual-networks to logical-routers) as part of
the database migration process using contrail software upgrades process. Also,
all new virtual-networks should be correctly linked to the logical-router as
applicable.

## Control-Node
Control-node does route serving to type-5 EVPN routes similar to how it does
for all other routes. Type-5 routes when received from agent are processed and
always added to L3VRF.evpn.0 table with protocol XMPP. From here on-wards, this
route is replicated into bgp.evpn.0 table so that it is advertised to all BGP
peers as well.

When Type-5 EVPN prefix is received from a BGP peer, it is first installed into
bgp.evpn.0 like all other routes. From here, based on matching route targets,
the route gets replicated into all *.evpn.0 tables as applicable. From there
the routes are advertised over xmpp to all interested agents.

Note: In Release 5.0, policy based route-leaking among different L3VRFs is not
      supported. Hence Service-Chaining for Type 5 L3VRFs is also not supported.

### Encoding

Today, EVPN Routes are always injected with Type 2 (MacAdvertisementRoute) as
the EVPN Route-type. These are for MAC And host IP /32 routes. We can use the
same encoding to send and receive type-5 routes as well. This can be indicated
using all-0s MAC address. This ensures that there is no schema change and hence
simplifies backward compatibility.

autogen data structures can used as is (because there is no schema change)
using fields as marked below, in NLRI And nexthop data structures.

ESI based overlay index is not used for forwarding. Label advertised by the
agent is expected to be the VXLAN Table label for the L3VPN.inet.0 or
L3VPN.inet6.0 as applicable.


```
struct EnetAddressType : public AutogenProperty {
    EnetAddressType();
    virtual ~EnetAddressType();
    int af;              // BGPAf::L2Vpn
    int safi;            // BGPAf::Enet
    int ethernet_tag;    //
    std::string mac;     // All 0s MAC address
    std::string address; // Any IPv4/IPv6 prefix (need not be always /32 host route)
};

struct EnetNextHopType : public AutogenProperty {
    EnetNextHopType();
    virtual ~EnetNextHopType();
    int af;
    std::string address;
    int label;           // L3VRF Table Label value
    int l3_label;
    EnetTunnelEncapsulationListType tunnel_encapsulation_list;
    EnetTagListType tag_list;
};
```

### Introspect and debugging

Since no additional data structures are introduced, all information necessary
should be available via already existing L3 tables related pages.

## Agent

Agent will rely on VXLAN routing (enabled on per project basis) to identify if
EVPN type-5 has to be enabled. This knob creates an internal VRF (here after
denoted as L3VRF) for each logical-router which will be used for routing across
all VN. The VN associated with L3VRF will have fixed VNI and not allow user to
configure the same. Encapsulation supported for type-5 will only be VXLAN for
unicast.

### Routes
L3VRF will be a regular VRF, with following characteristics:

* All other VRF will export route(inet/inet6) routes to L3VRF EVPN table with
  vhost MAC address.
* Export of routes to CN from L3VRF will be done for EVPN table as type 5 route.
* Other VRF will install default routes and host routes pointing to L3VRF.
* L3VRF will internally push the inet prefix from its EVPN table to its own
  inet/inet6 table. Vrouter will use these inet tables for routing which is
  similar to L3vpn routing.

### ARP
This will continue to be similar to current behavior where every other VRF
(except L3VRF) will have IP-MAC binding to reply for ARP. In L3VRF IP-MAC
binding will be absent as no ARP reply is required from same and proxy will be
enabled.

### Nexthop
Non L3VRF will have all inet/inet6 routes pointing to L3VRF table nexthop. L3VRF
routes will use three types of nexthop:

1. Tunnel Nexthop - This will be regular tunnel nh with rewrite MAC of
   destination vrouter MAC.
2. EVPN type-5 encap NH - This is a new type of L3 encap nexthop which will be
   used to point to VMI with rewrite MAC of VMI. The MAC will be retrieved from
   IP-MAC binding present in source VRF inet route.
3. L3 VRF NH - This will make sure that packets routed via L3 VRF use its VNID
   and on destination lookup is done for inet tables in L3VRF instead of bridge
   table.

### Example
#### Subnet routing
Say there are two VM in same VN-1 - VM-A(1.1.1.2) and VM-B(2.2.2.2)
* VM-A to reach VM-B will request for ARP of gateway.
* Vrouter responds with proxy vrouter MAC as it has to be routed.
* VM-A sends packet with vhost MAC and VM-B dest ip. VRF-1(mapping to VN-1) is
  looked in and route for 2.2.2.2 is found. It is pointing to table NH
  i.e. L3VRF. Lookup is again done in L3VRF inet table which will have route
  for 2.2.2.2 pointing to tunnel/encap NH.
* If its tunnel NH packet is sent with VXLAN encapsulation to destination
  compute and lookup is done again at the same. Since the VNI carried is of
  L3VRF, vrouter will see VNI pointing to L3 VRF NH. Lookup is done in inet
  table of L3VRF and encap NH is found. Rest is similar to encap NH processing.
* If its encap NH, destination MAC of VM-B is retrieved from same and packet
  rewritten with it and delivered to VM-B.

#### Inter-VN routing
VM-A belongs to VN-1 and VM-B belongs to VN-2.
VN-1 and VN-2 export their inet/inet6 routes to L3VRF.
VM-A when trying to reach VM-B will hit gateway. After that it is similar to
subnet routing explained above. Scope of VN-1 and VN-2 ends at source. Rest all
is done in L3VRF.

#### Bridging
This continues to be same as before. VM-A and VM-B are in same subnet in VN-1.
VM-A will request for VM-B ARP and it will be responded by vrouter from VN-1
inet table. Then the whole forwarding will happen via bridge table. Hence EVPN
type-5 changes will not impact the same.

**Note: Configuring L3VPN and EVPN type-5 together will result in undefined
        behavior as this is a mis-configuration **

## vRouter

### VNID
Vrouter will have to distinguish L2 VNID vs L3 VNID as the look-ups are in
different table. This will be done using type of VRF NH, VNID will be pointing
to.

### Nexthop
Inner destination MAC has to be retrieved from new encap NH (as mentioned in
agent changes). Its required as scope of MAC address ends at compute and other
computes wont know about it. Only IP is exchanged across computes.

Tunnel NH rewrite information will be with vhost MAC as dest MAC.

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

1. [EVPN Type5 IETF draft](https://tools.ietf.org/html/draft-ietf-bess-evpn-prefix-advertisement-08)
2. [Schema Changes](https://review.opencontrail.org/#/c/39167/)
3. [Blue Print](https://blueprints.launchpad.net/opencontrail/+spec/evpn-type-5)
4. [Control-Node changes](https://review.opencontrail.org/#/c/39200/)
