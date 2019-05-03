# 1. Introduction
Contrail Fabric Manager (CFM) supports assigning physical roles and routing bridging (overlay) roles to a device. These roles together define the roles and responsibilities of the device in the DC. The configuration pushed to the device is also governed by these roles. A device can have 1 physical role and 1 or more routing bridging roles.
Currently we don't have a mechanism to configure a replicator to optimize the efficiency of replication in Network Virtualization Overlay (NVO) networks. There is a need to support the AR-Client (Assisted Replicator client) and AR-Replicator (Assisted Replicator) roles designated replicators are configured for replicating BUM (Broadcast, Unknown-unicast and Multicast) traffic.
Reference: IETF RFC draft for EVPN assisted replication solution
https://tools.ietf.org/html/draft-ietf-bess-evpn-optimized-ir-06

# 2. Problem statement
CFM needs support for assisted replicator functionalities. It should add support for the following overlay roles
### AR-Client
This role configures the device as an assisted replicator client/leaf.

### AR-Replicator
This role configures the device as an assisted replicator, responsible for replicating the BUM traffic.

#### Deployment model:
Supported Roles:
- AR-Client@spine
- AR-Replicator@spine for QFX10k

#### Supported Device Models
AR-Client for QFX5k, QFX10k and MX as spine
AR-Replicator for QFX10k as spine

#### Software Releases
Junos 18.4R2

# 3. Proposed solution
The proposed solution is to use the existing CFM architecture for role based config push to the DC devices and add the new roles required.
CFM already has playbooks defined with different roles (for features) to push the config to the DC devices.
The effective config pushed to the devices is governed by the abstract config from Device Manager (DM) and the roles (physical, routing bridging) assigned to the device.
We will add two new routing bridging roles `AR-Client` and `AR-Replicator` and the corresponding configuration template

## 3.1 Alternatives considered
#### N/A

## 3.2 API schema changes
Add a field on Physical Router object to store replicator IP.

## 3.3 User workflow impact
The existing user workflow to assign roles remains the same. The user will see additional choices (`AR-Client` and `AR-Replicator`) for the applicable devices.

## 3.4 UI changes
For brownfield fabric creation loopback subnet should be mandatory.

## 3.5 Notification impact
#### N/A

# 4. Implementation
## 4.1 Add roles to node profile
CFM uses the node profile object to determine the roles applicable for a type of device. We will add the `AR-Client` role for MX, QFX10k and QFX5k as `spine` and the `AR-Replicator` role for QFX10k as `spine`

## 4.2 Support for AR-Client role
- Add a feature with the jinja template for AR-Client configuration

  The jinja template adds explicit knobs to configure the device as AR leaf
  ```
  set groups AR_CLIENT protocols evpn assisted-replication leaf replicator-activation-delay 30
  ```
  The activation delay value should be a configurable property driven by the existing role config framework

## 4.3 Support for AR-Replicator role
- Every replicator needs a unique replicator IP to be configured. During role assignment allocate IP from the loopback subnet to every device with AR-Replicator role (AR_IP) in both Greenfield and Brownfield fabric.

- Device Manager (DM) should propagate the replicator IP (AR_IP) for the device in abstract config.

- Add a feature with the jinja template for AR-Replicator configuration

  The jinja template adds explicit knobs to configure the device as AR Replicator
  ```
  set groups AR_REPLICATOR assisted-replication replicator inet <AR_IP>
  set groups AR_REPLICATOR assisted-replication replicator vxlan-encapsulation-source-ip ingress-replication-ip
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
The current CFM & DM Ansible config push implementation does not have any unit tests or a unit test infrastructure.
The unit tests for this feature will be in the ansible layer but with the assumption that building the infrastructure won't be in the scope of this feature.

## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact
Currently when the physical and routing bridging roles are assigned to the device, there is no check done on the version number of the Device OS.
In the case of `AR-Client` and `AR-Replicator` the config is valid only when the Junos version on the devices is 18.4R2 or later.
This needs to be documented, since the role can still be assigned for older Junos versions but the config push will fail.

# 11. References
