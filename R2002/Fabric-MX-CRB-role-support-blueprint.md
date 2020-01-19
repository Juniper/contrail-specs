# 1. Introduction
Contrail Fabric Manager (CFM) supports multiple physical roles and routing bridging (overlay) roles on QFX devices currently. For MX devices, only DC gateway role is currently supported. This spec covers the design and implementation for the support of CRB (Centrally Routed Bridging) role to a MX device. In CRB the routing for a tenant occurs remotely from the place where a workload is attached. So in terms of leaf-spine architecture, the routing happens at spine. There is a need to support the ERB (Edge Routed Bridging) role where the routing for a tenant happens at leaf, but is not within the scope of this spec. This spec will only cover unicast routing, multicast routing is not within scope of this spec.

### Supported Deployment model for MX:
Supported Roles:
- CRB-Gateway@spine

### Supported Device Models for MX:
MX 240, MX 480, MX 960

# 2. Problem statement
CFM needs to support CRB gateway role for MX devices. 
A device with this role can only perform Unicast Inter-VN routing
* When a Virtual Network is configured on a port of a device provision the following (only for UNICAST traffic)
  * The Interface/IFL to VXLAN VNI mapping
  * The Bridge Domain
*  When a Logical Router across 2 or more Virtual Network is created, provision on all Devices where these VN exist.
  * The Anycast Gateway (irb) units and addresses
  * The L3 VRF
  
#### Software Releases
MX devices running with 18.1R3 or above are supported for this functionality.

#### Deployment model:
Supported Roles:
- CRB-Gateway@spine

#### Supported Device Models
MX 240, MX 480, MX 960, MX 10003, MX 204

# 3. Proposed solution
The proposed solution is to use the existing CFM architecture for role based config push to the DC devices and add the role required for MX. CFM already supports the CRB gateway role for QFX devices, we will leverage the existing configuration templates and make respective changes required for MX support and add any templates as needed.

# 4. API schema changes
N/A
# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
N/A
# 7. Notification impact
N/A
# 8. Provisioning changes
N/A
# 9. Implementation
Mentioned below are the current features being supported for CRB-gateway@spine overlay role. We will either reuse the existing template or have a new jinja template for the feature specific, which will be specific to MX as detailed below: 
 - basic
     * `enhanced-ip` has to be enabled at chassis configuration. This need to be added to a new template for MX along with the existing configuration available in basic.
     > set chassis network-services enhanced-ip
 - overlay_bgp
     - No change in the existing jinja template.
 - overlay_evpn
     -    `switch-options` configuration which includes the vtep config in the existing template would move directly under the routing instance configuration. 
     -    protocol configuration for evpn will also move under routing instance.
     -    `no-gateway-community` needs to be configured under protocol.
     
     Example config:
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn vtep-source-interface lo0.0
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn instance-type virtual-switch
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn route-distinguisher 77.77.77.77:7999
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn vrf-import __contrail_LR1_e7403f8f-ad10-4611-b9b2-b333c84c8d54-import
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn vrf-import _contrail_lvn-l2-11-import
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn vrf-import _contrail_rvn-l2-12-import
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn vrf-import import-evpn
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn vrf-target target:64512:7999999
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn protocols evpn encapsulation vxlan
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn protocols evpn extended-vni-list all
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn protocols evpn vni-options vni 11 vrf-target target:64512:8000010
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn protocols evpn vni-options vni 12 vrf-target target:64512:8000011
>     deactivate groups __contrail_overlay_evpn__ routing-instances user_evpn protocols evpn vni-options
>     set groups __contrail_overlay_evpn__ routing-instances user_evpn protocols evpn default-gateway no-gateway-community

- overlay_evpn_gateway
    Few major changes here from the same feature template for Qfx.
    * Bridge domains : `Bridge-domains` are used instead of `vlans` for defining the bridge domain and the member irb interfaces.
    * Bridge domain config is part of the `routing-instances` and not global.
    
Example config is listed below for reference:
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 description "Network - lvn"
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 domain-type bridge
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 vlan-id 100
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 interface xe-2/0/0.100
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 routing-interface irb.11
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 vxlan vni 11
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 vxlan encapsulate-inner-vlan
>     set groups __contrail_overlay_evpn_gateway__ routing-instances user_evpn bridge-domains bd-11 vxlan decapsulate-accept-inner-vlan

- overlay_evpn_type5
    * Type 5 configuration is not needed for MX. Enabling enhanced-ip and adding the `instance-type vrf` will allow the inter-vn routing to happen.
    
- overlay_networking
    * Logical router creation should trigger a creation of `instance-type vrf` which will stitch together the irb interfaces for the inter-vn routing.

Example config:
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF instance-type vrf
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF interface irb.11
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF interface irb.12
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF interface lo0.1
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF route-distinguisher 1:1110000
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF vrf-target target:1:111
> set groups __contrail_overlay_evpn_gateway__ routing-instances L3VRF routing-options auto-export


# 11. Performance and scaling impact
N/A

# 12. Upgrade
#### Backward compatible changes

# 13. Deprecations
N/A

# 14. Dependencies
- Current physical and routing bridging role lookup and assignment schema
- Current overlay abstract config generation in DM
- Current overlay config push architecture (Ansible way)
- 
# 15. Testing
## 15.1 Unit tests
The current CFM & DM Ansible config push implementation does not have any unit tests or a unit test infrastructure.
The unit tests for this feature will be in the ansible layer but with the assumption that building the infrastructure won't be in the scope of this feature.

#### Test #1 - Verify the unicast features are executed
- Verify the Unicast feature is executed for the CRB-UCAST-Gateway role

## 15.2 Feature tests
Below is the link to the feature test plan,
https://drive.google.com/open?id=1p4lpjiCbCFXFZvm2qWNHpX8Jc15L709_

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. References
https://contrail-jws.atlassian.net/browse/CEM-5407 
https://contrail-jws.atlassian.net/browse/CEM-11291
