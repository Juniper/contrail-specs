# 1. Introduction
This document describes the idea behind Contrail VMware fabric integration to manage VMware underlay networks

# 2. Problem statement

Currently VMware stack: vCenter + ESXi hosts lacks of good user experience automation for underlay DC infrastructure. The purpose of this project is to provide such automation.

## Sample usecase:

## Initial situation:
Customer with ESXi hosts under vCenter management where Distributed Virtual Switch (DVS)
and Distributed Portgrups (DPG) solutions are widely used.

## Expected situation:
Contrail in this case will be an automation tool to extend the management of ESXi hosts via
vCenter to the DC infrastructure. This will provide completely automated connectivity
for VMs running under ESXi hosts.

# 3. Proposed solution
As part of Contrail's VMware fabric installation Contrail vCenter Fabric Manager (further named CVFM) plugin will be deployed. The purpose of CVFM plugin is to synchronize the configuration of VMware Distributed Portgroups with the configuration on TOR (Leaf) switch (like QFX) port connected to the host (ESXi) where this Distributed Port Groups have active instance of VM’s. This will allow customers to configure the VLAN on Distributed Port Group of the Distributed Virtual Switch and have that VLAN configuration reflect on the switch port connected to the host.

# 4. User workflow impact

# 5. References
