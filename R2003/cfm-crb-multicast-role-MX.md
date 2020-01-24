# 1. Introduction
Contrail Fabric Manager (CFM) supports multiple physical roles and routing bridging (overlay) roles on QFX devices currently. For MX devices, CRB gateway and ERB gateway are currently already supported. This spec covers the design and implementation for the support of CRB-MCAST (Centrally Routed Bridging for Multicast) role to a MX device. In CRB the routing for a tenant occurs remotely from the place where a workload is attached. So in terms of leaf-spine architecture, the routing happens at spine.  This spec only covers the missing multicast routing.

### Supported Deployment model for MX:
Supported Roles:
- CRB-MCAST-Gateway@spine

### Supported Device Models for MX:
MX 240, MX 480, MX 960

# 2. Problem statement
CFM needs to support CRB-MCAST gateway role for MX devices. A device with this role can only perform Multicast Inter-VN routing
This role is similar to the existing CRB-Gateway role with the addition of explicit knobs for are added

-   When configuring Logical Router the Anycast Gateway should be configured for Multicast
-   When configuring Logical Router add explicit knobs for Multicast (including PIM-SM)

#### Software Releases
MX devices running with 18.1R3 or above are supported for this functionality.

#### Deployment model:
Supported Roles:
- CRB-MCAST-Gateway@spine

#### Supported Device Models
MX 240, MX 480, MX 960, MX 10003, MX 204

# 3. Proposed solution
The proposed solution is to use the existing CFM architecture for role based config push to the DC devices and add the role required for MX. CFM already supports the CRB mcast gateway role for QFX devices, we will leverage the existing configuration templates and make respective changes required for MX support and add any templates as needed.

# 4. API schema changes
N/A
# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
The existing user workflow to assign roles remains the same. The user will see additional choice (`CRB-MCAST-Gateway`) for the applicable devices.

# 7. Notification impact
N/A
# 8. Provisioning changes
N/A
# 9. Implementation
## 4.1 Add roles to node profile
CFM uses the node profile object to determine the roles applicable for a type of device. We will add the `CRB-MCAST-Gateway` role for MX  as `spine`.

## 4.2 Add Multicast feature templates
- Add a Multicast feature with the jinja template for configuration

   The jinja template adds explicit knob for multicast routing & enable IGM snooping.

```
set groups MULITCAST routing-instances _contrail-l2 protocols igmp-snooping vlan all proxy
set groups MULTICAST forwarding-options multicast-replication evpn irb local-remote
```

And PIM-SM config for IRBs in VRF

```
set groups MULTICAST routing-instances VRF-1 protocols pim interface all mode sparse-dense
set groups MULTICAST routing-instances VRF-1 protocols pim interface all family inet
```

# 10. Device specifics
MX devices running with 18.1R3 or above are supported for this functionality.

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
Below is the link to the test plan:
https://drive.google.com/open?id=1LKBVeC6VX5AJRRSz2Yw9hETNYUujK_BA

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. References
JIRA Story: https://contrail-jws.atlassian.net/browse/CEM-11256
