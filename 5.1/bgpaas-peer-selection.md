# 1. Introduction
The BGPaaS has to support control-node-zone selection.
Configuration option would be given in BGPaaS to choose control-node-zones
to which it has to be peered.

# 2. Problem statement
Service Providers are relying on BGPaaS for routing from/to VNF.
VNFs are made up of several nodes spread on different computes.
VNFs will usually rely on two bgp peers for High Availablity.
All BGP peers from a VNF can be terminated onto the same control node.
If the control node fails, both primary and secondary peer get lost at
the same time. It will break the High Availablity.

# 3. Proposed solution
In the BGPaaS configuration, option would be given to choose primary and
secondary control-node-zones. Control-node-zones can have one or more
control-nodes. If control-node-zone has more than one control-nodes, one will
be choosen randomly as peer. Existing model would be followed if
control-node-zone is not configured(i.e if either primary/secondary or both
control-node-zone(s) are configured, existing model would not be followed)

## 3.1 Alternatives considered None.

## 3.2 API schema changes
New Ifmap identifier control-node-zone will be added as a child to
global-system-config.
    <xsd:element name="control-node-zone" type="ifmap:IdentityType"/>
    <xsd:element name="global-system-config-bgp-router"/>
    <!--#IFMAP-SEMANTICS-IDL
        Link('global-system-config-bgp-router',
                'global-system-config', 'bgp-router', ['ref'], 'required', 'R',
                'List of references to all bgp routers in systems.') -->

Link will be added between bgp-router and control-node-zone.
    <xsd:element name="bgp-router-control-node-zone"/>
    <!--#IFMAP-SEMANTICS-IDL
        Link('bgp-router-control-node-zone',
            'bgp-router', 'control-node-zone', ['ref'], 'optional', 'CRUD',
            'This bgp-router belongs to the referenced control-node-zone.') -->

Link will be added between bgpaas and control-node-zone with attribute
BGPaaSControlNodeZoneAttributes to select primary and secondary
control-node-zones.

    <xsd:element name="bgpaas-control-node-zone" type="BGPaaSControlNodeZoneAttributes"/>
    <!--#IFMAP-SEMANTICS-IDL
     Link('bgpaas-control-node-zone',
          'bgp-as-a-service', 'control-node-zone', ['ref'], 'optional', 'CRUD',
          'Reference to control-node-zone for bgp-peer selection') -->

    <xsd:complexType name="BGPaaSControlNodeZoneAttributes"/>
        <xsd:all>
            <xsd:element name='bgpaas-control-node-zone-type'
                type='BGPaaSControlNodeZoneType'
                required='optional' operations='CRUD'
                description="Specifies BGPaaSControlNodeZoneType. If bgpaas uses
                x.x.x.1 ip for peering, BGPaaSControlNodeZoneType should be set to
                primary. If it is x.x.x.2 ip for peering, BGPaaSControlNodeZoneType
                should be secondary"/>
        <xsd:all>
        </xsd:all>
    </xsd:complexType>

    <xsd:simpleType name="BGPaaSControlNodeZoneType">
        <xsd:restriction base="xsd:string">
            <xsd:enumeration value="primary"/>
            <xsd:enumeration value="secondary"/>
        </xsd:restriction>
    </xsd:simpleType>


## 3.3 User workflow impact
User is expected to configure control-node-zones and select primary and
secondary zone in BGPaaS
Any configuration change would be applied to new BGPaaS sessions. Existing
flows wont be affected.
If Existing flows are affected for any reasons, new flows would be set with
current BGPaaS configuration.

## 3.4 UI changes
Global-System-Config will have an option to add/modify/delete control-node-zones
Control-Node-Zone will have an option to add/modify/delete control-nodes
Bgp-Router will have an option to add/modify/delete control-node-zone.
BGPaaS will have an option to add/modify/delete primary and secondary
control-node-zone.

## 3.5 Notification impact

# 4. Implementation

## 4.1 Work items

### 4.1.1 Vrouter Agent changes
Contrail-vrouter-agent will get control-node-zone and bgp-router
configuration from respective ifmap nodes. It will also get primary
and secondary control-node-zone names from bgpaas-control-node-zone.
BGPaaS in the contrail-vrouter-agent will construct primary and secondary
bgp-peers(control-nodes) from control-node-zones. If VNF uses x.x.x.1,
bgp-peer will be selected from primary. If it uses x.x.x.2, bgp-peer
will be selected from secondary. [Selecting bgp-peer in primary/
secondary is random.]. If control-node-zone is not configured in BGPaaS,
it will follow the existing model(choose first xmpp server for x.x.x.1
and second xmpp server for x.x.x.2).
BgpAsAServiceSandeshResp page in the introspect will be enhanced to show
control-node-zones and bgp-peers for the BGPaaS.
BgpRouterSandeshResp and ControlNodeZoneSandeshResp pages will be added in the
introspect to get configured bgp-routers and control-node-zones.

#### 4.1.1.1 Vrouter Agent work flow
1. Contrail-vrouter-agent will construct BgpRouterConfig from bgp-router
   and control-node-zone ifmap nodes.
2. BgpRouterConfig will provide apis to get bgp-router for a control-node-zone
   or for an ip.
3. BGPaaS will get the bgp-router using BgpRouterConfig apis by passing
   control-node-zone or xmpp server ip during flow setup for BGPaaS.
4. BGPaaS will set the bgp-router ip and port for the BGPaaS flows if
   bgp-router is found. Otherwise flows would be dropped.

### 4.1.2 Vrouter changes
There are no vrouter changes.

## 4.2 Limitations

# 5. Performance and scaling impact
## 5.1 API and control plane
#### Scaling and performance for API and control plane

## 5.2 Forwarding performance
#### Scaling and performance for API and forwarding

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None.

# 9. Testing
## 9.1 Unit tests
a) Configure BGPaaS with primary/secondary control-node-zone and check the
bgp-peer
b) Configure BGPaaS without control-node-zone and check it follows the existing
model.
c) Stop all control-nodes in primary control-node-zone and see if bgpaas peer
does not re-establish
d) Stop all control-nodes in secondary control-node-zone and see if bgpaas peer
does not re-establish

## 9.2 System tests

# 10. Documentation Impact

# 11. References
