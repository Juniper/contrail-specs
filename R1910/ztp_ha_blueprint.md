# 1. Introduction
Currently in CEM we onboard Fabric in one of the below mentioned ways.
a. Brownfield (underlay configuration of the fabric devices are managed by admin. CEM only handles the overlay)
b. Greenfield or Zero Touch Deployment (whole fabric management is done by CEM - underlay and overlay)

Both the above Fabric management schemes need be supported in two deployment scenerios that Contrail currently supports :
All In One (AIO) and High Availability (HA).

                         AIO          HA
Onboarding scheme

Brownfield               yes          yes
Greenfield               yes          NO

# 2. Problem statement
As stated in the introduction, we currently dont support ZTP in Contrail HA deployment.
Device manager (DM) and dnsmasq are tightly coupled. Since DM is active-backup, the active DM's Dnsmasq would have the active dhcp server running (lease file would be populated).
Incase the node crashes while dhcp server is serving, and another DM comes up active, its corresponding dnsmasq would be active now.
But the lease files are stored in the local filesystem, so the new active dnsmasq will not be aware of the previously/already served clients.

# 3. Proposed solution
Have dnsmasq run in a pseudo active backup mode, so that at any point in time only one dnsmasq is serving the clients.
 - Achieved by using the dnsmasq option - dhcp-reply-delay (seconds), which is set to different values on the different nodes of the cluster.

Use external storage system to populate dhcp lease info.
 - Achieved by using the the dnsmasq options - dhcp-script and leasfile-ro, which allows us to run a customized script to handle dhcp responses and
                                               leasefile-ro allows us to use external storage for lease data , in our case , its cassandra DB.

# 4. API schema changes
Add a new state 'dhcp' in physical-router object -
<xsd:simpleType name="ManagedStateType">
     <xsd:restriction base='xsd:string'>
         <xsd:enumeration value='dhcp'/>
         <xsd:enumeration value='rma'/>
         <xsd:enumeration value='activating'/>
         <xsd:enumeration value='active'/>
         <xsd:enumeration value='maintenance'/>
         <xsd:enumeration value='error'/>
     </xsd:restriction>
</xsd:simpleType>
<xsd:element name="physical-router-managed-state" type='ManagedStateType' default="active"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('physical-router-managed-state', 'physical-router', 'optional', 'CRUD',
              'Managed state for the physical router. Used to RMA devices') -->


Add a new filed to store dnsmasq lease info -
<xsd:complexType name="DnsmasqLeaseParameters">
    <xsd:all>
        <xsd:element name="lease_expiry_time" type='xsd:integer'/>
        <xsd:element name="client_id" type='xsd:string'/>
    </xsd:all>
</xsd:complexType>

<xsd:element name="physical-router-dhcp-parameters" type='DnsmasqLeaseParameters'/>
<!--#IFMAP-SEMANTICS-IDL
          Property('physical-router-dhcp-parameters', 'physical-router', 'optional', 'CRUD',
              'Dnsmasq lease parameters for the physical router.') -->

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
N/A. All the changes are infrastructure backend.

# 7. Notification impact
N/A

# 8. Provisioning changes
HA deployment for ZTP attached.

# 9. Implementation
Dnsmasq-
two scripts introduced.
dnsmasq_init_script.sh - Script gets triggered whenever there is init/add/del of lease parameters
dnsmasq_lease_processing.py - To handle router object creation in database (external lease storage). 
                              The router created will have the mac, dhcp ip, lease expiry info and will be put in 'dhcp' state.

configuration files-

base.conf
log-facility=/var/log/contrail/dnsmasq.log
bogus-priv
log-dhcp
dhcp-reply-delay=0  [value will be 2 or 4 on the other nodes]

<fabric_name>.conf
dhcp-vendorclass=set:jn,Juniper
dhcp-range=10.87.13.1,10.87.13.29,255.255.255.224,10.87.13.31,12h
dhcp-option=option:router,10.87.13.30
dhcp-option=66,10.87.13.3
dhcp-option=150,10.87.13.3
dhcp-option=tag:jn,encap:43,1,"<fabric_name>.sh"
dhcp-option=tag:jn,encap:43,3,"tftp"
dhcp-script=/etc/dnsmasq/dnsmasq_init_script.sh     -- new
leasefile-ro                                        -- new

Devicemanager (Fabric)-
Once we successfully onboard the device, we delete the corresponding 'dhcp' state object during device discovery.

# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
dhcp-reply-option is available from dnsmasq 2.8 onwards. We download the 2.8 version src and locally build it.

# 14. Testing
https://docs.google.com/spreadsheets/d/1z0JB-WGvAaTHifIp4qpDfjhmitDmVYF7j3oCZp1mO18

# 15. Documentation Impact
The feature will have to be documented

# 16. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-7585
Dnsmasq : http://www.thekelleys.org.uk/dnsmasq/doc.html