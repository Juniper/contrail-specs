1. Introduction
This blueprint describes the requirements, design and implementation for automating un-managed PNFor3rd party device and LR inter-connect using Contrail.

2. Problem statement
Currently, there is no way for user to specify unique individual IRB IP address for ERB/CRB gateway device. As of today, Contrail pushes only anycast ip on irb interface in ERB gateways.

Also, user cannot specify any routing protocol like eBGP or static routes to be configured on the irb interface. User cannot control import/export routing policies for routes which are learned via eBGP protocol.

3. Proposed solution
Users should have an option to configure individual ips in ERB/CRB gateways. There should be an option to also configure static routes or eBGP routing protocol session on those IRBs. Those constructs can be applied on L2 VPGs (access ifls) or Routed VPGs (fabric ports).

4. User workflow impact
1) Create a routed virtual Network and identify the CIDR. Attach the Virtual network to a Logical router from Overlay-> Logical Router screen. User will be required to input the IRB IP address on a per gateway device (ERB or CRB), to which this logical router is extended to, i.e if there are say 4 CRB GW spine nodes, then the user should be prompted to provide IP address for each of the CRB gateway and similarly for the ERB gateway. Also when a new CRB-GW or ERB-GW is added to the fabric (by adding a new device or by assigning the CRB-GW or ERB-GW role to a existing device), user should be prompted to assign the IRB IP addresses for the newly added nodes.
2) Also, user should be given an option to configure the below on the IRB unit for that routed VN
    - eBGP inet routing protocol session.
    - Static routes (User needs to select prefix name from existing interface static route table defined under Overlay->Routing. User can select multiple such route table n
      ames. User also need to provide next-hop IP address.
    - For eBGP - user must have the following UI options
      1) Peer IP for eBGP session. This is a mandatory input
      2) ASN number of local/peer endpoints. Local ASN is optional, however rpeer ASN is mandatory
      3) Authentication method (only MD5 is supported as of now) and authentication key. This is optional and user can choose None authentication method.
      NOTE: eBGP Session must use the unique IP address of irb unit and not the anycast IP address (which is the same across all QFXs).
3) If user has selected eBGP routing protocol for the VN subnet, user can apply import/export routing policies (one or more) on this inet routing protocol session. These routing policies are created using Overlay->Routing UI screen
4) User has the option to specify BFD protocol session on the IRB unit of this VN. CEM will configure BFD on the IRB unit when attaching the VN to the Logical Router (or to the master inet routing table). User has the option to select rx/tx time interval and detection time multiplier for this BFD session.

2) Logical Routers:
By default , two Logical Routers will be created at cluster bring-up. They represent the master inet.0 and inet.6 routing table. These default logical routers cannot be deleted, but the user can still add/delete routed VNs to/from them. When a routed VN is added to these logical routers, irb unit will be configured in the master routing tableinstead of VRF on the device.

5. API Schema changes
1) New property is added in existing virtual-port-group object to indicate whether its L2 VPG or Routed VPG
<xsd:element name="virtual-port-group-type" type='VpgType' default="access"/>
<!--#IFMAP-SEMANTICS-IDL
               Property('virtual-port-group-type', 'virtual-port-group', 'optional', 'CRUD',
                   'Type of Virtual port group. It can be either access i.e L2 connectivity or routed.') -->

<xsd:simpleType name="VpgType">
     <xsd:restriction base='xsd:string'>
         <xsd:enumeration value='access'/>
         <xsd:enumeration value='routed'/>
     </xsd:restriction>
</xsd:simpleType>

2) New property is added in existing virtual-network object to indicate whether its Routed VN or Tenant VN
<xsd:simpleType name="VirtualNetworkCategory">
    <xsd:restriction base="xsd:string">
        <xsd:enumeration value='infra'/>
        <xsd:enumeration value='tenant'/>
        <xsd:enumeration value='internal'/>
        <xsd:enumeration value='routed'/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:element name="virtual-network-category" type="VirtualNetworkCategory" default="tenant"/>
<!--#IFMAP-SEMANTICS-IDL
     Property('virtual-network-category', 'virtual-network', 'optional', 'CRUD',
              'This attribute is to differentiate the infrastructure networks from the tenant and routed networks. Infra-networks could be in-band network for management and control traffic') -->

3) New properties are added in existing virtual-network object, in case virtual network is of type Routed
<xsd:element name="virtual-network-routed-properties" type="VirtualNetworkRoutedPropertiesType"/>
<!--#IFMAP-SEMANTICS-IDL
     Property('virtual-network-routed-properties', 'virtual-network', 'optional', 'CRUD',
               'Attributes for routed virtual networks.') -->

<xsd:complexType name='VirtualNetworkRoutedPropertiesType'>
    <xsd:all>
        <xsd:element name="routed-properties" type="RoutedProperties" maxOccurs="unbounded"
             description='List of routed properties for virtual network.'/>
    </xsd:all>
</xsd:complexType>

<xsd:simpleType name="RoutingProtocolType">
    <xsd:restriction base='xsd:string'>
        <xsd:enumeration value='static-routes'/>
        <xsd:enumeration value='bgp'/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="RoutingPolicyParamaters">
    <xsd:all>
        <xsd:element name='import-routing-policy-uuid' type='xsd:string' required='optional' maxOccurs="unbounded"
             description='list of routing policy uuids used as import policies.'/>
        <xsd:element name='export-routing-policy-uuid' type='xsd:string' required='optional' maxOccurs="unbounded"
             description='list of routing policy uuids used as export policies.'/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="StaticRouteParamaters">
    <xsd:all>
        <xsd:element name='interface-route-table-uuid' type='xsd:string' required='true' maxOccurs="unbounded"
             description='list of interface route table uuids used to build list of prefixes.'/>
        <xsd:element name='next-hop-ip-address' type='smi:IpAddress' required='true' maxOccurs="unbounded"
             description='List of next-hop IP addresses.'/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="BfdParameters">
    <xsd:all>
        <xsd:element name='time-interval' type='xsd:integer' required='optional'
             description='Rx/Tx time interval for BFD session.'/>
        <xsd:element name='detection-time-multiplier' type='xsd:integer' required='optional'
             description='Detection time multiplier for BFD session.'/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="BgpParamaters">
    <xsd:all>
        <xsd:element name='peer-autonomous-system' type='xsd:integer' required='true'
         description='Peer autonomous system number for this eBGP session.'/>
    <xsd:element name='peer-ip-address' type='smi:IpAddress' required='true'
         description='Peer ip address used for this eBGP session.'/>
    <xsd:element name='hold-time' type='BgpHoldTime' default='0' required='optional'
         description='BGP hold time in seconds [0-65535], Max time to detect liveliness to peer. Value 0 will result in default value of 90 seconds'/>
    <xsd:element name='auth-data' type='AuthenticationData' required='optional'
         description='Authentication related configuration like type, keys etc.'/>
    <xsd:element name='local-autonomous-system' type='xsd:integer' required='optional'
         description='BgpRouter specific Autonomous System number if different from global AS number.'/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name='RoutedProperties'>
    <xsd:all>
        <xsd:element name='physical-router-uuid' type='xsd:string' required='true'
             description='Routed properties for particular physical router for this virtual-network.'/>
        <xsd:element name='routed-interface-ip-address' type='IpAddressType' required='true'
             description='IP address of the routed interface from the VN subnet.'/>
        <xsd:element name='routing-protocol' type='RoutingProtocolType' required='true'
             desription='Protocol used for exchanging route information.'/>
        <xsd:element name='bgp-params' type='BgpParamaters' required='optional'
             description='BGP parameters such as ASN, peer IP address, authentication method/key.'/>
        <xsd:element name='static-route-params' type='StaticRouteParamaters' required='optional'
             description='Static route parameters such as interface route table uuid, next hop IP address.'/>
        <xsd:element name='bfd-params' type='BdfParamaters' required='optional'
             description='BFD parameters such as time interval, detection time multiplier.'/>
        <xsd:element name='routing-policy-params' type='RoutingPolicyParamaters' required='optional'
             description='List of import/export routing policy uuids.'/>
    </xsd:all>
</xsd:complexType>

4) New property is added in existing logical-router object to indicate if it represents master/global routing table on device, instead of VRF
<xsd:element name="logical-router-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
     Property('logical-router-is-default', 'logical-router', 'optional', 'CRUD',
              'This attribute indicates whether it is a default logical router or not. This is used to represent global routing table and user can add/delete routed virtual-networks to this logical-router which needs to be a part of global routing table. Default logical routers cannot be deleted.') -->

6. Performance and scaling impact
None

7. Deprecations
None

8. Dependencies
None

9. Testing

9.2 Dev tests
9.3 System tests
10. Documentation Impact

11. References
https://contrail-jws.atlassian.net/browse/CEM-4455
https://contrail-jws.atlassian.net/browse/CEM-5405
https://contrail-jws.atlassian.net/browse/CEM-6974
