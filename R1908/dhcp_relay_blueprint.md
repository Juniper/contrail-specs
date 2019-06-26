# 1. Introduction
Contrail TSN node currently acts as a DHCP server for Managed BMS.

# 2. Problem statement
Customers would liked to be given an option to use their own DHCP servers instead of being forced to use the 
s/TSN/CSN node.

#### Supported Device Models
QFX5k, QFX10k

#### Software Releases
Junos 18.4R2 and onwards

# 3. Proposed solution
Allow the user/customer to provide DHCP server(s) information while adding Logical Router (LR).
There are three scenerios that need to be handled depending on where the DHCP server is located:
DHCP server is-

1. Within the same LR/VN
2. Reachable via default RI a.k.a Internet
3. In a different LR (Not in the current scope. Will be part of LR-Interconnect feature)

##3.1 Current scope limitations
1. If the user is going with the approach of using external DHCP server, they should NOT provision TSN node. 
(hence no mix-mode : external-DHCP and TSN node or toggling between external-DHCP and TSN node is supported)
2. Once external DHCP server is used to manage BMS(s), if down the line user decides to remove the DHCP server and 
provision TSN node instead, CEM will NOT manage the IPs. Customer would have to re-provision the BMS(s)

## 3.1 Alternatives considered
#### N/A

## 3.2 API schema changes
Add optional DHCP-Relay list field in Logical Router object.
   ```
<xsd:element name="logical-router-dhcp-relay-server" type="IpAddressesType" required="optional"/>
<!--#IFMAP-SEMANTICS-IDL
     Property('logical-router-dhcp-relay-server', 'logical-router', 'optional', 'CRUD',
              'DHCP server IP(s) to serve managed BMS(s).') -->
   ```
   
## 3.3 User workflow impact
User will have to provide DHCP Server(s) info during LR create/update.

## 3.4 UI changes
On Overlay->Logical Router page,  optional additional field for the user to enter 'DHCP Server' IP.
User can add multiple servers.

## 3.5 Notification impact
#### N/A

# 4. Implementation
## 4.1
Device Manager gets an update for LR create/update. If DHCP relay info is found, we first identify

### 4.1.1
If it resides in the same subnet as the VN. Following config is pushed to the leaf/spine to which the LR is extended:
  ```
    set routing-instances <vrfname> forwarding-options dhcp-relay forward-only
    set routing-instances <vrfname> forwarding-options dhcp-relay forward-only-replies
    set routing-instances <vrfname> forwarding-options dhcp-relay server-group DHCP_SERVER_GROUP <dhcp-server-ip>
    set routing-instances <vrfname> forwarding-options dhcp-relay active-server-group DHCP_SERVER_GROUP
    set routing-instances <vrfname> forwarding-options dhcp-relay group RELAY_DHCP_SERVER_GROUP interface <irb>
    set routing-instances <vrfname> forwarding-options dhcp-relay overrides relay-source lo0
    set interfaces irb unit <irb-uint> family inet address <irb-ip> preferred
  ```
IF multiple DHCP server ips, they would be added under the same relay group:
  ```
    set routing-instances <vrfname> forwarding-options dhcp-relay server-group DHCP_SERVER_GROUP <dhcp-server-ip>
  ```

IF multiple VNs are part of the same LR, all irbs would be configured under the same VRF:
  ```
    set routing-instances <vrfname> forwarding-options dhcp-relay group RELAY_DHCP_SERVER_GROUP interface <irb(n)>
  ```

### 4.1.2
If not, we need to leak routes to inet.0. In which case besides the generic dhcp relay config (mentioned above) we push 
the below additional config:

Route from LR to inet.0 -
  Static Route add for every DHCP server:
  ```
    set routing-instances <vrfname> routing-options static route <dhcp-server-ip> next-table inet.0
  ```
Route from inet.0 to LR -
  Static route add for every irb ip:
  ```
    set routing-options static route <irb-ip> next-table <vrfname>.inet.0
  ```
Note: For this scenerio, user needs to make sure the underlay is configured. CFM is NOT responsible for this config.

# 5. Performance and scaling impact
## 5.1 API and control plane
#### N/A

## 5.2 Forwarding performance
#### N/A

# 6. Upgrade
#### Backward compatible changes

# 7. Deprecations
#### N/A

# 8. Dependencies


# 9. Testing
## 9.1 Unit tests
Unit tests for Device Manager Ansible abstract config validation will be covered.
Note:
Since CFM Ansible config push implementation does not have any unit tests or unit test infrastructure, they are not in 
the scope of this feature.

## 9.2 Dev tests
The acceptance tests that will be identified will be automated.

## 9.3 System tests

# 10. Documentation Impact
- The scenerio where DHCP server resides in a different LR not being supported needs to be documented.
- Configuring underlay for DHCP server if it doesnot reside in the same VN.
- The points mentioned under the section 3.1

# 11. References