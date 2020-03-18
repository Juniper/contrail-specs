1. Introduction
This blueprint describes the requirements, design and implementation of extending Contrail routing policy terms to support Juno devices and use this routing policies to leak routes selectively between two or more logical routers exists in the same contrail fabric.

2. Problem statement
Currently, there is no way for user to leak the routes between logical routers that are exists in the same Contrail fabric. Contrail Data center interconnect feature supports leaking routes between the logical routers exists in different fabrics.
Also, currently user cannot specify any routing policies and their policy terms to be configured for leaking routes between the logical routers. Contrail routing policy terms exist in the UI contains only the policy terms that vRouter compute device supports, this policy terms also requires extending to support Juniper junos devices.

3. Proposed solution
User should have an option to configure routing policy and its terms that applies to juniper junos physical device. Extend Routing policies terms to include Junos device supported terms.
There should be an option to specify routing policy to leak routes across different logical routers exists in the same fabric.
To leak routes, user should allow to select one logical router (LR) as a source and one or more logical routers as destination; along with user should allow to select either the list of routing policies or the list of tenant virtual networks existed in source logical router. Route will be leaks from source LR to all destination LR using either routing policies provided by user or user selected list of tenant virtual networks subnet which are already existed in source LR. 
On Junos device, It will import routing policies route-filter or tenant virtual network's subnets as prefix list which ever provided by user, this policies gets included in logical router's routing instances as a RIB-groups and this rib-groups will be assigned to routing-options of every destination LR's routing instances to leak the routes from source LR to destination LR(s).

4. User workflow impact
4-1) Create routing policy and its terms for the junos physical device from Overlay->Routing->Routing Policies screen. User will be required to select Routing Policy Type either Physical device or vRouter. On selection of Physical device type, Routing policy terms specific to junos devices will be provided to the user to configure. Details of this policy terms is specified in API Schema Changes section below. For inter-connect LR on same fabric, user should configure routing policy for physical device with terms having route-filter from clause must be defined.

4-2) Contrail UI will replace option of Overlay->DCI (Data Center Interconnect) with new UI option Overlay->Interconnects.
Overlay->Interconnects UI will have  Two-tab options; DCI and LR Interconnects.
- DCI UI screen will be exactly the same as previously existed under UI overlay->Data Center Interconnect screen.
- LR Interconnect UI screen will provide necessary details mentioned below in 4-3 steps to leak routes from source LR ro destination LR(s).

4-3) New UI screen Overlay->Interconnects->LR Interconnects will have following configuration options for user to provide to leak the routes from single source LR to multiple destination LR(s), all LR must exists in the same fabric.
       - Name, Description
       - Under Source table:
            - Select single Fabric
            - Select single Logical Router
            - Select one option from radio button: Routing Policy or virtual networks.
            - Depends on radio button selection, following details will be provided to user:
                - Select one or more Routing Policy. Or
                - Select one, or more Tenant virtual networks. These are the list of
                  tenant virtual networks which are already existed in selected source LR.
       - Under Destination table: User should allow to add one or more destination LR and their information.
            - Select single Logical router as destination LR
            - View only Fabric Name, this will display fabric name where this destination LR exists. thisn fabric name must be match to fabric name selected under Source table.
            - Select one or multiple physical routers existed in this selected destination LR. These are the list of physical router user will select on which leak routes configuration will be pushed to import routes from source LR.
            + Add link to add more destination LR.  
      
5. API Schema changes
1) In routing-policy object, New property is added in existing routing-policy object to indicate term-type as network-device or vrouter. 
<xsd:complexType name='PolicyStatementType'>
    <xsd:element name='term-type' type='TermType'/>
    <xsd:element name='term' type='PolicyTermType' maxOccurs='unbounded'/>
</xsd:complexType>

<xsd:simpleType name='TermType'>
    <xsd:restriction base="xsd:string" default="vrouter">
        <xsd:enumeration value="vrouter"/>
        <xsd:enumeration value="network-device"/>
    </xsd:restriction>
</xsd:simpleType>

2) In routing-policy object, list of new property and definition is added to extend policy terms to support junos device as a network-device term type.
<xsd:simpleType name='FamilyType'>
    <xsd:restriction base="xsd:string">
        <xsd:enumeration value="inet"/>
        <xsd:enumeration value="inet-vpn"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name='ExternalRouteType'>
    <xsd:restriction base="xsd:string">
        <xsd:enumeration value="ospf-type-1"/>
        <xsd:enumeration value="ospf-type-2"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="PrefixListMatchType">
    <xsd:element name="interface-route-table-uuid" type='xsd:string' required='true' maxOccurs="unbounded"
             description='list of interface route table uuids used to build list of prefixes.'/>
    <xsd:element name="prefix-type" type="PrefixType"/>
</xsd:complexType>

<xsd:complexType name="RouteFilterType">
    <xsd:element name="route"    type="xsd:string"  required='true'
         description='route filter IP address or host name.'/>
    <xsd:element name="route-type" type="RouteType"/>
    <xsd:element name="route-type-value" type="xsd:string" required='optional'
         description="Valid only for through, upto or prefix-length-range RouteType. Format: for prefix-length-range it stores '/min-length-/max-length'. through should be number, upto stores '/number'"/>
</xsd:complexType>

<xsd:simpleType name='RouteType'>
    <xsd:restriction base="xsd:string" default="exact">
        <xsd:enumeration value="exact"/>
        <xsd:enumeration value="longer"/>
        <xsd:enumeration value="orlonger"/>
        <xsd:enumeration value="prefix-length-range"/>
        <xsd:enumeration value="through"/>
        <xsd:enumeration value="upto"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name='PathSourceType'>
    <xsd:restriction base="xsd:string">
        <!-- Followings are valid for only termType vrouter -->
        <xsd:enumeration value="xmpp" description='valid for only vrouter termtype'/>
        <xsd:enumeration value="service-chain" description='valid for only vrouter termtype'/>
        <xsd:enumeration value="interface" description='valid for only vrouter termtype'/>
        <xsd:enumeration value="interface-static" description='valid for only vrouter termtype'/>
        <xsd:enumeration value="service-interface" description='valid for only vrouter termtype'/>
        <xsd:enumeration value="bgpaas" description='valid for only vrouter termtype'/>
        <!-- Followings are valid for both termType vrouter and network-device -->
        <xsd:enumeration value="bgp" description='valid for both termtype'/>
        <xsd:enumeration value="static" description='valid for both termtype'/>
        <xsd:enumeration value="aggregate" description='valid for both termtype'/>
        <!-- Followings are valid for only termType network-device -->
        <xsd:enumeration value="direct"/>
        <xsd:enumeration value="pim"/>
        <xsd:enumeration value="evpn"/>
        <xsd:enumeration value="ospf"/>
        <xsd:enumeration value="ospf3"/>
    </xsd:restriction>
</xsd:simpleType>

3) In routing-policy object, Policy Term is extended to support junos devices from and to term clauses
<xsd:complexType name="TermMatchConditionType">
    <xsd:element name="protocol" type="PathSourceType" maxOccurs="unbounded"/>
    <xsd:element name="prefix" type="PrefixMatchType" maxOccurs="unbounded"/>
    <xsd:element name="community" type="xsd:string"/> <!-- DEPRECATED, USE IN NETWORK-DEVICE -->
    <xsd:element name="community-list" type="xsd:string" maxOccurs="unbounded"/> <!-- USE IN NETWORK-DEVICE -->
    <xsd:element name="community-match-all" type="xsd:boolean"/>
    <xsd:element name="extcommunity-list" type="xsd:string" maxOccurs="unbounded"/>
    <xsd:element name="extcommunity-match-all" type="xsd:boolean"/>
    <!-- Followings are valid for only termType network-device -->
    <xsd:element name="family" type="FamilyType" required='optional'/>
    <xsd:element name="as-path" type='xsd:integer' required='optional' maxOccurs='unbounded'/>
    <xsd:element name="external" type='ExternalRouteType' required='optional'/>
    <xsd:element name="local-pref" type="xsd:integer" required='optional'/>
    <xsd:element name="nlri-route-type" type='xsd:integer' required='optional' maxOccurs='unbounded'
         description='list of integer values in range of 1 to 10 inclusive.'/>
    <xsd:element name="prefix-list" type='PrefixListMatchType' required='optional' maxOccurs="unbounded"
         description='list of prefixes from interface route table uuids with prefix types.'/>
    <xsd:element name="route-filter" type='RouteFilterType' required='optional'/>
</xsd:complexType>

<xsd:complexType name='TermActionListType'>
    <xsd:element name='update'  type='ActionUpdateType'/>
    <xsd:element name='action'  type='ActionType'/>
    <xsd:enumeration value="as-path-expand" type='xsd:string' required='optional'
         description="Valid only for network-device TermType. string shuld be in format of AS number(s) seperated by spaces"/>
    <xsd:enumeration value="as-path-prepend" type='xsd:string' required='optional'
         description="Valid only for network-device TermType. string shuld be in format of AS number(s) seperated by spaces"/>
</xsd:complexType>

4) Following new properties are added in existing data-center-interconnect object. type to indicate it represents interconnect logical-router in same fabric or across fabric and list of routing policies and list of virtual networks uuid to import route-filter prefixes in route leaks, list of physical routers (PR) existed in one or more destination logical routers user has selected to push route leak configration to this PR device(s).

<xsd:simpleType name="DataCenterInterConnectType" default='inter_fabric'>
    <xsd:restriction base='xsd:string'>
        <xsd:enumeration value='inter_fabric'/>
        <xsd:enumeration value='intra_fabric'/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:element name="data-center-interconnect-type" type='DataCenterInterConnectType'
             description='DCI Type. inter-fabric type is across the two fabrics. intra-fabric type is within a single fabric. For intra-fabric type, DCI-virtual-network-refs, DCI-routing-policy-uuid-list and DCI-physical-router-uuid-list properties should have valid values. It gets used to leak the routes from first entry defined in DCI-logical-router-refs as source LR to destination LR(s) defined in DCI-logical-router-refs second and on wards entries'/>
<!--#IFMAP-SEMANTICS-IDL
     Property('data-center-interconnect-type', 'data-center-interconnect', 'optional', 'CRUD',
              'Defines type of DCI, inter-fabric is across two fabric. intra-fabric is single fabric.') -->

<xsd:element name="data-center-interconnect-routing-policy-uuid-list" type='xsd:string' maxOccurs="unbounded"
             description='Used only if DCI-type is intra-fabric. it stores the List of routing policies uuid for this DCI to be used as import policies between logical routers. if any single or more uuid defined in this property then DCI-virtual-network-refs property value will be ignored for route leaks route-filter.'/>
<!--#IFMAP-SEMANTICS-IDL
     Property('data-center-interconnect-routing-policy-uuid-list', 'data-center-interconnect', 'optional', 'CRUD',
              'List of uuid of routing policies for this DCI to be used as import policies between logical routers.') -->

<xsd:element name="data-center-interconnect-physical-router-uuid-list" type='xsd:string' maxOccurs="unbounded"
             description='Used only if DCI-type is intra-fabric. it stores the List of physical router uuids. this physical router must be existed in one or more destination logical routers for intra-fabric DCI type. User selects this PR(s) per destination LR to leak routes from source LR to destination LR on this list of PRs only.'/>
<!--#IFMAP-SEMANTICS-IDL
     Property('data-center-interconnect-physical-router-uuid-list', 'data-center-interconnect', 'optional', 'CRUD',
              'holds List of physical router uuid of destination LR(s) in intra-fabric type DCI object') -->

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








