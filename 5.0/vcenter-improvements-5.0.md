# 1. Introduction

# 2. Problem statement
Following diagram shows OpenContrail integration with vCenter. 

 


 As part of the solution, single Distributed Virtual Switch (DVS) is created across the DataCenter (DC).  Each DVS Port Group (PG) is mapped to Virtual Network (VN) on Contrail. In order to ensure Virtual Machines always communicate via VirtualRouter, DVS PortGroup is configured with Isolated Private VLAN (PVLAN). 

 Use of PVLAN has following drawbacks:
 1) User/Operator needs to manager PVLAN allocation.
 2) Transparent service chaining is not supported due to limitation on double tagging (?)
 3) Since VLAN is allocated per PG ( i.e. Virtual-Network), vRouter uses vlan.src-mac to identify the VM interface on which packet was received. This restricts VRRP configuration on the Vmware VMs.

  Drawbacks of the plugin solution:
  1.  vcenter-Plugin is centrally managed. Failure of plugin can result in contrail control plane being offline.
  2.  Plugin currently manages single Data-Center.

  In addition to above, ContrailVM is spawned as a user VM. Drawback of this approach:
  1.  ESXi host schedules VMs, even when ContrailVM is down/deleted.
  2.  User can delete ContrailVM. It is not spawned again automatically.
  3.  ContrailVM is in data-path but it’s not the VM to be powered down neither is it the first VM to be powered on.

  # 3. Proposed solution
  The proposed solution is divided into 2 phases.
  1.  Phase I
  a.  vCenter Plugin Modifications
  b.  ESXi Agent Manager
  2.  Phase II
  a.  Distributed vCenter Plugin
  b.  vCenter Solution Manager

  Phase I will be delivered as part of 5.0. Phase II will be delivered in later release.

  ## 3.1 Phase I 

  ### 3.1.1  vCenter Plugin Modifications
  Here is the desired workflow change in vcenter-plugin.
  Network Creation:
  1.  Via Contrail WebUI, user creates a Virtual Network with required subnet information.
  2.  vCenterPlugin monitors for the new VN created and creates a DVSPortGroup with Same Name. DVSPortGRoups is configured as vlan type of none and also allow VLAN override.

  Virtual Machine Creation:
  3.  User creates VM and places the VM in Port Group representing Virtual Network.
  4.  Plugin receives VM creation event and overrides VM Port with VLAN that is unique per ESXi host. vCenter plugin will maintain per ESXi/vRouter VLAN allocation info. 
  5.  Plugin creates VM/VMI/InstanceIp objects on contrail api-server.
  6.  Plugin also does plug to vRouter Agent with this VLAN info. 

  Virtual Machine Deletion:
  1.  User deletes VM via vCenter UI.
  2.  Plugin receives VM delete event. It frees and VLAN used for override to the free pool.
  3.  It deletes VM/VMI/InstanceIp from contrail api-server.
  4.  Plugin also sends unplug message to contrail vRouter Agent.

  Virtual Network Deletion:
  1.  User deletes Virtual Network via Contrail WebUI.
  2.  Plugin knows about deleted network when it sync’s with vCenter PG, it deletes corresponding DVS PG from vCenter.
  3.  Either of the operations can fail, if interface/port is hanging around VN/PG.

  ### 3.1.2  ESXi Agent Manager

  ## 3.2 Alternatives considered

  ## 3.2 API schema changes
  .

  ## 3.3 User workflow impact

  ## 3.4 UI changes

  ## 3.5 Notification impact

  # 4. Implementation

  ## 4.1 Work items

  # 5. Performance and scaling impact

  ## 5.1 API and control plane

  ## 5.2 Forwarding performance

  # 6. Upgrade

  # 7. Deprecations
  This feature does not deprecate any older feature.

  # 8. Dependencies
  This feature depends on OpenContrail Java API.

  # 9. Testing
  Significant part of the final plugin code will be generated and should be mainly tested at the level of system tests.

  ## 9.1 Unit tests
  Development will include unit tests for non-generated code included in the plugin.

  ## 9.2 Dev tests
  Development will include code review. 

  ## 9.3 System tests

  #10. Documentation Impact
  TODO

  11. References

