# 1. Introduction
This blueprint describes integrating Contrail and Appformix as a
single application for a unified telemetry user experience. With
this feature, Contrail can now provision network devices with telemetry
configurations and also have appformix related targets (eg. SNMP
targets, gRPC collectors and sflow collectors) under this telemetry
umbrella.

1. For 1910, we support only out-of-band(OOB) collector provisioning.
In-Band collector provisioning support will be added in the future release.
2. The collectors can be provisioned only once and before fabric onboarding.
3. No new collectors can be added to the cluster after initial provisioning is done.
4. Also, we support only sflow targets under telemetry at present. The
current framework can later be extended to include other configurations
like gRPC or SNMP.

# 2. Problem statement
Once devices are discovered on Contrail Command, user should be able to
attach a telemetry profile to the devices. This telemetry profile
would be an encapsulation of other configs such as sflow, gRPC.

# 3. Proposed solution
A new Infrastructure >  Fabrics > 'Telemetry Profiles' tab will be added.

The telemetry profile can be attached to a Physical Router (device).
There can be only one telemetry profile per device. The options to create
telemetry profiles, view existing ones will be provided.

The telemetry profile will include a side panel to display the existing
sflow profiles. There can also be only one sflow profile for a
telemetry profile. The sflow profiles can be created at the time a telemetry
profile is created and referred to this telemetry profile. Otherwise the user
can choose to attach an existing sflow profile after the telemetry profile is
created.

The sflow profile will contain the following configuration parameters -
sample rate, polling interval, agent id, adaptive sample rate, enabled
interface type and enabled interface params.

The following are the accepted values for some of the sflow parameters:
1. sample rate (number (one packet out of number)): Accepted Range: 1-16777215, default: 2000
2. polling interval (seconds): Accepted Range: 0-3600(seconds), default: 20
3. adaptive sample rate (number): Accepted Range: 300-900, default: 300

# 4. API schema changes

Each collector server is represenetd as a flow-node object in VNC database.
Objects to represent collector parameters such as virtual IP address
are added to the vnc schema. Telemetry subnets are modelled for supporting
in-band collector provisioning in future. The schema changes are captured below:

<xsd:element name="flow-node" type="ifmap:IdentityType"/>
<xsd:element name="global-system-config-flow-node"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('global-system-config-flow-node',
             'global-system-config', 'flow-node', ['has'], 'admin-only', 'CRUD',
             'Appformix flows node is object representing a logical node in system which serves xflow collectors.') -->

<xsd:element name="flow-node-ip-address" type="IpAddressType"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('flow-node-ip-address', 'flow-node', 'admin-only', 'CRUD',
              'Ip address of the appformix flow node, set while provisioning.') -->

<xsd:element name="flow-node-virtual-network"/>
<!--#IFMAP-SEMANTICS-IDL
         Link('flow-node-virtual-network',
             'flow-node', 'virtual-network', ['ref'], 'optional', 'CRUD',
             'Similar to using virtual-machine to model the bare metal server, we are using virtual-network to model telemetry underlay network. This would allow us to re-use the same IPAM data model and code base to manage the IP auto-assignments for the underlay telemetry networks.') -->

<xsd:element name="flow-node-load-balancer-ip" type="IpAddressType"/>
<!--#IFMAP-SEMANTICS-IDL
         Property('flow-node-load-balancer-ip', 'flow-node', 'required', 'CRUD',
              'IP address of the load balancer node for xflow collectors, set while provisioning.') -->


Objects to represent the telemetry profile and the sflow profile are
added to the fabric schema. The schema changes are captured below:

<xsd:simpleType name="SampleRate">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="1"/>
        <xsd:maxInclusive value="16777215"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name="PollingInterval">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="0"/>
        <xsd:maxInclusive value="3600"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name="AdativeSampleRate">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="300"/>
        <xsd:maxInclusive value="900"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="StatsCollectionFrequency">
    <xsd:all>
        <xsd:element name="sample-rate" type="SampleRate" required="optional" default="2000"/>
        <xsd:element name="polling-interval" type="PollingInterval" required="optional" default="20"/>
    </xsd:all>
</xsd:complexType>

<xsd:simpleType name="EnabledInterfaceType">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="all"/>
         <xsd:enumeration value="fabric"/>
         <xsd:enumeration value="service"/>
         <xsd:enumeration value="access"/>
         <xsd:enumeration value="custom"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="EnabledInterfaceParams">
    <xsd:all>
        <xsd:element name="name" type="xsd:string"/>
        <xsd:element name="stats-collection-frequency" type="StatsCollectionFrequency"/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="SflowParameters">
    <xsd:all>
       <xsd:element name="stats-collection-frequency" type="StatsCollectionFrequency"
            description= "Represents polling interval and sample rate either at the global level or at per interface level. Polling interval is specified in seconds that the device waits between port statistics update messages. Sample rate is specified as a number (one packet out of number)." />
       <xsd:element name="agent-id" type="smi:IpAddress"
            description= "IP address to be assigned as the agent ID for the sFlow agent." />
       <xsd:element name="adaptive-sample-rate" type="AdativeSampleRate" required="optional" default="300"
            description= "Represents  maximum number of samples that should be generated per line card." />
       <xsd:element name="enabled-interface-type" type="EnabledInterfaceType"
            description= "User can enable sflow either on all interfaces or all fabric ports or all access ports or custom list of interfaces." />
       <xsd:element name="enabled-interface-params" type="EnabledInterfaceParams" maxOccurs="unbounded"
            description= "If interface type is set to custom, this represents the list of physical interfaces to be enabled for sflow. User can specify polling interval and sample rate for each interface in this custom list." />
    </xsd:all>
</xsd:complexType>

<xsd:element name="telemetry-profile" type="ifmap:IdentityType"
    description="Encapsulates data related to telemetry from network devices like sflow, JTI, gRPC, SNMP etc"/>

<xsd:element name="project-telemetry-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-telemetry-profile', 'project', 'telemetry-profile', ['has'], 'optional', 'CRUD',
             'list of telemetry profiles supported under the project.') -->
<xsd:element name="telemetry-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('telemetry-profile-is-default', 'telemetry-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default telemetry profile or not. Default profiles are non-editable.') -->

<xsd:element name="telemetry-profile-sflow-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-sflow-profile', 'telemetry-profile', 'sflow-profile', ['ref'], 'optional', 'CRUD',
             'Sflow profile that this telemetry profile uses. Only one sflow profile can be associated to one telemetry profile.') -->
<xsd:element name="physical-router-telemetry-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('physical-router-telemetry-profile', 'physical-router', 'telemetry-profile', ['ref'], 'optional', 'CRUD',
             'Telemetry profile assigned to the physical router by user. Each physical router is associated with only one telemetry profile.') -->

<xsd:element name="sflow-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to sflow parameters"/>

<xsd:element name="project-sflow-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-sflow-profile', 'project', 'sflow-profile', ['has'], 'optional', 'CRUD',
             'list of sflow profiles supported under the project.') -->
<xsd:element name="sflow-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('sflow-profile-is-default', 'sflow-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default sflow profile or not. Default profiles are non-editable.') -->

<xsd:element name="sflow-parameters" type="SflowParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('sflow-parameters', 'sflow-profile', 'optional', 'CRUD',
                  'Parameters for each sflow profile, such as polling interval, sample rate, list of sflow enabled interfaces, sflow agent ID etc.') -->

# 5. UI changes / User workflow impact

#### - Select Telemetry Profiles from Fabrics (under Infrastructure)
User selects the 'Telemetry Profiles' option under the fabric option

#### - Existing telemetry profiles list
The existing telemetry profile objects are also displayed above. User can create
new telemetry profiles from this page

#### - Existing sflow profiles list
Upon clicking to edit a telemetry profile or creating a new one, the user is
taken into this page. The sflow Properties option here displays all the existing
sflow profile objects. The user can just select a single existing
sflow profile or just create a new one via 'Create New' option on the side.

#### - Sflow control creation page
User can create a new sflow profile by entering the profile name,
sampling rate, polling interval and adaptive sample rate. The user
must mandatorily select one of the following interface types
1. All Revenue Interfaces
2. All Fabric Interfaces
3. All Interfaces
4. Custom (user can choose multiple devices from multiple fabrics and multiple
interfaces from each of the devices)

#### - Telemetry profile attachment to Physical Router
The telemetry profile can be assigned to the Physical Router in this page.
Navigate to select Fabric and click on the fabric. This will list the devices
in the fabric. User can choose to group devices to be assigned same telemetry
profile or assign different telemetry profiles to different devices.

Upon selecting the devices to be assigned a telemetry porfile, an option 'Assign
Telemetry Profile' will be enabled. Clicking this button will then display a
pop-up which will list all existing telemetry profiles. Also hovering over a
particular telemetry profile will show a side panel displaying the sflow details
if attached to this telemetry profile.

# 7. Notification impact
N/A

# 8. Provisioning changes
During flow collector provisioning stage, the flow collector's details will be
registered to Contrail API Server. A POST request will be sent from flow collector
deployer to API Server with collector management and load-balancer IP address.

# 9. Implementation
The main implementation changes include:
1) Create new objects to represent the telemetry profile and the sflow
profile in the schema. The telemetry profile will refer to the Physical
Router (device) object.

2) Device manager changes to cache these new objects

3) Device manager changes to generate abstract config changes upon creation,
updation and deletion of the telemetry(sflow) objects

4) Jinja templates to generate the telemetry(sflow) configuration

The configurations for QFX10K and QFX5K are captured below:

#### Applicable for QFX - To attach the sflow profile

set protocols sflow interfaces xe-0/0/0
set protocols sflow interfaces xe-0/0/0 sample-rate ingress 1200
set protocols sflow interfaces xe-0/0/0 sample-rate egress 1200
set protocols sflow interfaces xe-0/0/0 polling-interval 900

###### Configuring Profile:
set protocols sflow adaptive-sample-rate 500 fallback
set protocols sflow collector 10.1.1.0 udp-port 6343
set protocols sflow agent-id 5.5.5.5
set protocols sflow sample-rate ingress 300
set protocols sflow sample-rate egress 500
set protocols sflow polling-interval 1000

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
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-4836
Junos documentation: https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/sflow-edit-protocols-qfx-series.html
