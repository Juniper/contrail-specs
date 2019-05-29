# 1. Introduction

This document describes the idea behind Contrail Fabric Manager and VMware integration to manage VMware underlay networks and contains an overview of the design of Contrail vCenter Fabric Manager (CVFM).

# 2. Problem statement

The purpose of this project is to synchronize the configuration of VMware Distributed Portgroups with the configuration on TOR (Leaf) switch (like QFX) port connected to the host (ESXi) where this Distributed Port Groups have active instance of VM’s. This will allow customers to configure the VLAN on Distributed Port Group of the Distributed Virtual Switch and have that VLAN configuration reflect on the switch port connected to the host.

# 3. Proposed solution

vSphere API can be used to obtain events regarding any Distributed Virtual Portgroup configuration change in vCenter. This information can be read using pyVmomi (Python API library for vSphere) and used to update the VNC database with appropriate data (using python VNC API library). Then, the desired configuration can be propagated to the QFXs by the Device Manager.

# 4. Alternatives considered

# 5. API schema changes

# 6. UI changes

# 7. Notification impact

# 8. Provisioning changes

Contrail vCenter Fabric Manager runs in separate docker container. That container is deployed using Contrail Ansible Deployer (CAD) through vcenter_fabric_manager role on contrail config node. Container requires access to config api and vCenter.

# 9. Implementation

Contrail vCenter Fabric Manager (CVFM) running on the config node of the contrail will perform the necessary tasks to achieve the automatic configuration and deletion of the VLANs on Switch Port based on the VLANs configured on the Distributed Port Groups of a Distributed Virtual Switch.

Contrail vCenter Fabric Manager(CVFM) will have the following behavior:

1. Plugin communicates with vCenter to listen to all the events related to vCenter like Port Group create/delete, VM create/delete and DVS etc.
2. Plugin communicates with CFM to get the mapping between ESXi host and Switch Port (QFX).
3. Plugin communicates with CFM to configure the Switch Port with VLANs corresponding to Port Group VLANs configured on Distributed Virtual Switch.

# 10. Performance and scaling impact

# 11. Upgrade

# 12. Deprecations

# 13. Dependencies

# 14. Testing
## Unit tests
The project repository will contain a suite of unit tests prepared by developers to verify that each code component works as expected on mocked data.
## Functional tests
The project repository will contain a suite of functional tests prepared by developers and meant to be run against a working instance of Config API node. These tests will use mocked data as vSphere Events and verify that a proper configuration is being pushed to VNC database.

# 15. Documentation Impact

# 16. References
