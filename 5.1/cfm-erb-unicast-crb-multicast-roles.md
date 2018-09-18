# 1. Introduction
Contrail Fabric Manager (CFM) supports assigning physical roles and routing bridging (overlay) roles to a device. These roles together define the roles and responsibilities of the device in the DC. The configuration pushed to the device is also governed by these roles. A device can have 1 physical role and 1 or more routing bridging roles.
Currently the CRB (Centrally Routed Bridging) roles is already supported for DC devices. In CRB the routing for a tenant occurs remotely from the place where a workload is attached. There is a need to support the ERB (Edge Routed Bridging) role wherein the routing for a tenant occurs at the attachment point. This spec covers the design and implementation of the ERB role for DC devices.

# 2. Problem statement
CFM supports the following routing bridging roles on DC devices (in 5.0.2)
- Null
- CRB-Access
- CRB-Gateway
- DC-Gateway
These roles allow the DC devices to support CRB and DC-Gateway functionalities.

CFM needs support for ERB functionalities for DC devices. It should add support for the following overlay roles
### CRB-Gateway-MCAST
This role is similar to the existing CRB-Gateway role with the addition of explicit knobs for are added 
* When configuring Logical Router the Anycast Gateway should be configured for Multicast
* When configuring Logical Router add explicit knobs for Multicast (including PIM-SM)

### ERB-Access-UCAST
A device with this role can only perform Unicast Inter-VN routing
* When a Virtual Network is configured on a port of a device provision the following (only for UNICAST traffic)
  * The Interface/IFL to VXLAN VNI mapping
  * The Bridge Domain
* When a Logical Router across 2 or more Virtual Network is created, provision on all Devices where these VN exist
  * The Anycast Gateway (irb) units and addresses
  * The L3 VRF
  * The EXPLICIT KNOBs that indicates that Inter-VN Multicast is performed remotely, as well PIM-SM configuration

#### Deployment model: CRB with border spine
Supported Roles:
- CRB-Gateway-MCAST@spine
- ERB-Access-UCAST@leaf when the leaf is a QFX5110
- CRB-Access@leaf when the leaf is a QFX

#### Supported Device Models
Leaf is QFX5100, QFX5200, QFX 5110, QFX10K
Spine is QFX 10K, QFX 5110

#### Software Releases
Junos 18.1R3

# 3. Proposed solution
The proposed solution is to use the existing CFM architecture for role based config push to the DC devices and add the new roles required.
CFM already has playbooks defined with different roles (for features) to push the config to the DC devices.
The effective config pushed to the devices is governed by the abstract config from Device Manager (DM) and the roles (physical, routing bridging) assigned to the device.
We will add two new routing bridging roles `CRB-Gateway-MCAST` and `ERB-Access-UCAST` and the corresponding configuration template

## 3.1 Alternatives considered
#### N/A

## 3.2 API schema changes
#### N/A

## 3.3 User workflow impact
The existing user workflow to assign roles remains the same. The user will see additional choices (`CRB-Gateway-MCAST` and `ERB-Access-UCAST`) for the applicable devices.

## 3.4 UI changes
#### N/A

## 3.5 Notification impact
#### N/A

# 4. Implementation
## 4.1 Add roles to node profile
CFM uses the node profile object to determine the roles applicable for a type of device. We will add the `CRB-Gateway-MCAST` role for MX, QFX10k and QFX 5110 as `spine` and the `ERB-Access-UCAST` role for QFX5110 as `leaf`

## 4.2 Add Unicast and Multicast features and associate them with the corresponding roles
- Add a Unicast feature with the jinja template for configuration

  The jinja template adds explicit knob to indicate that inter-VN multicast is performed remotely
  ```
  set groups MULTICAST forwarding-options multicast-replication evpn irb remote
  ```

- Add a Multicast feature with the jinja template for configuration

  The jinja template adds explicit knob for multicast routing
  ```
  set groups MULTICAST forwarding-options multicast-replication evpn irb local
  ```
  And PIM-SM config for IRBs in VRF
  ```
  set groups MULTICAST routing-instances VRF-1 protocols pim interface irb.1 family inet
  set groups MULTICAST routing-instances VRF-1 protocols pim interface irb.1 mode sparse-dense
  ```

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
- Current physical and routing bridging role lookup and assginment schema
- Current overlay abstract config generation in DM
- Current overlay config push architecture (Ansible way)

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact
Currently when the physical and routing bridging roles are assigned to the device, there is no check done on the version number of the Device OS.
In the case of `ERB-Access-UCAST` and `CRB-Gateway-MCAST` the config is valid only when the Junos version on the devices is 18.1R3 or later.
This needs to be documented, since the role can still be assigned for older Junos versions but the config push will fail.

# 11. References
