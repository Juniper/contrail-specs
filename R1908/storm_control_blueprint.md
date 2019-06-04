# 1. Introduction
A traffic storm occurs when broadcast packets prompt receiving devices to
broadcast packets in response. This prompts further responses, creating a
snowball effect. The switch is flooded with packets, which creates unnecessary
traffic that leads to poor performance or even a complete loss of service by
some clients. Storm control causes a device to monitor traffic levels and
take a specified action when a specified traffic level—called the storm
control level—is exceeded, thus preventing packets from proliferating
and degrading service. You can configure devices to drop broadcast and
unknown unicast packets, shut down interfaces, or temporarily disable
interfaces when the storm control level is exceeded.

# 2. Problem statement
Once devices are discovered on Contrail Command, user should be able to 
configure storm control on the interfaces.

# 3. Proposed solution
A new overlay configuration option called 'Access port profile' will be added.
Access port profile will currently compose of the storm control profile
configuration. Later on access port profile can be leveraged as the docking 
point for configurations like port mirroring etc

The access port profile can be attached to a VPG. The options to create 
access profiles, view existing ones will be provided.

The access port will internally include the storm control profile.

The storm control profile will contain the following configuration params - 
profile name, action, bandwidth (level or percentage), options to disable
storm control for unicast, multicast, unknown unicast, registered multicast and 
unregistered multicast. 

# 4. API schema changes

Objects to represent the access port profile and the storm control are
added to the schema. The schema changes is captured below:

<xsd:simpleType name="StormControlActionType">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="shutdown"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="StormControlBandwidth">
    <xsd:choice>
        <xsd:element name="bandwidth-level" type="xsd:integer" required="optional"
             description="Bandwidth expressed in kbps"/>
        <xsd:element name="bandwidth-percentage" type="xsd:float" required="optional"
             description="Bandwidth expressed as a percentage"/>
    </xsd:choice>
</xsd:complexType>

<xsd:element name="storm-control-profile" type="ifmap:IdentityType"
    description="Storm control configuration definition"/>
<xsd:element name="global-system-config-storm-control-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('global-system-config-storm-control-profile', 'global-system-config', 'storm-control-profile', ['has'], 'optional', 'CRUD',
             'list of storm control profiles supported by the system.') -->
<xsd:element name="storm-control-bandwidth" type="StormControlBandwidth"
    description="Configure storm control bandwidth as a level in kbps or as a percentage"/>
<xsd:element name="no-broadcast" type="xsd:boolean" default='false'
                     description="if set to true, disable broadcast traffic storm control"/>
<xsd:element name="no-multicast" type="xsd:boolean" default='false'
                     description="if set to true, disable multicast traffic storm control"/>
<xsd:element name="no-unknown-unicast" type="xsd:boolean" default='false'
                     description="if set to true, disable unknown unicast traffic storm control"/>
<xsd:element name="no-registered-multicast" type="xsd:boolean" default='false'
                     description="if set to true, disable registered multicast storm control"/>
<xsd:element name="no-unregistered-multicast" type="xsd:boolean" default='false'
                     description="if set to true, disable unregistered multicast storm control"/>
<xsd:element name="action" type="StormControlActionType"
                     description="Default DROP or DISCARD action is implicit. In addition other actions can be specified here"/>

<xsd:element name="access-port-profile" type="ifmap:IdentityType"
    description="Encapsulates access port configurations like storm control etc"/>
<xsd:element name="global-system-config-access-port-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('global-system-config-access-port-profile', 'global-system-config', 'access-port-profile', ['has'], 'optional', 'CRUD',
             'list of access port profiles supported by the system.') -->
<xsd:element name="access-port-profile-storm-control-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('access-port-profile-storm-control-profile', 'access-port-profile', 'storm-control-profile', ['has'], 'optional', 'CRUD',
             'Storm control profile that this access port profile uses.') -->
<xsd:element name="access-port-profile-virtual-machine-interface"/>
<!--#IFMAP-SEMANTICS-IDL
         Link('access-port-profile-virtual-machine-interface', 'access-port-profile', 'virtual-machine-interface', ['ref'], 'optional', 'CRUD',
         'Virtual machine interface for the port for which the access profile belongs to') -->


# 5. Alternatives considered
Apart from the solution discussed above, the following options were considered:
Option1: Provide storm control profile configuration options in the node profile. 
The is not good approach since it will not allow reuse of the storm control 
profiles across the different node profiles.
Option2: Provide storm control profile configuration options during fabric creation.
This will not allow the reuse of the profiles across different fabrics.

# 6. UI changes / User workflow impact

#### - Select Access Port Profiles from the overlay features
User selects the 'Access port profile' option from the list of overlay features
![Overlay features list](images/storm_control_overlay_feature_list.png)

#### - Existing access port profiles list
The existing Access Port Profile objects are displayed here. User can create
new access port profiles from this page
![Access port profile list](images/storm_control_access_port_profile.png)

#### - Existing storm control profiles list
Upon clicking to edit an access profile or creating a new one, the user is
taken into this page. The storm control tab here displays all the existing
storm control profile objects. The user can just select an (single) existing
storm control profiles or just create a new one via 'create' option on the
right hand top corner.
![Storm control profile list](images/storm_control_profiles_list.png)

#### - Storm control creation page
User can create a new storm control profile by entering the bandwidth, action etc
![Create storm control profile](images/storm_control_config_page.png)

#### - Access port profile attachment to VPG
The access port profile can be assigned to the VPG here. The access port profiles
drop down will display the existing access port profiles.
![Storm control VPG assignment page](images/storm_control_vpg.png)

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
The main implementation changes include:
1) Create new objects to represent the access port profile and the storm control
profile in the schema. The access port profile will refer to the virtual machine
interface (VMI) object.

2) Device manager changes to cache these new objects

3) Device manager changes to generate abstract config changes upon creation 
updation and deletion of the storm control objects

4) Jinja templates to generate the storm control configuration

The configurations for QFX10K and QFX5K vary slightly and the sample configs
are captured below:

####QFX10K
######Attaching profile:
set interfaces ae11 unit 0 family ethernet-switching storm-control stm-ctrl
######Configuring Profile:
set forwarding-options storm-control-profiles stm-ctrl all bandwidth-level 1000000
set forwarding-options storm-control-profiles stm-ctrl action-shutdown

####QFX5K:
######Attaching profile:
set interfaces ae11 unit 0 family ethernet-switching storm-control stm-ctrl

######Configuring Profile:
set forwarding-options storm-control-profiles stm-ctrl all bandwidth-level 1000000


# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Testing
#### Unit tests
#### Dev tests
#### System tests

# 15. Documentation Impact
The feature and workflow will have to be documented

# 16. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-4179
Junos documentation: https://www.juniper.net/documentation/en_US/junos/topics/concept/rate-limiting-storm-control-qfx-series-understanding.html 
