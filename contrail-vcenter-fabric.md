# 1. Introduction
#### VMware integration with contrail fabric manager.

# 2. Problem statement

#### Customers using VMware vSphere product suits lack the capability to configure the networking hardware in their data-center deployment. VMware product suits allows customers to deploy, configure and manage private cloud infrastructure, and also public cloud via AWS but customers need to find ways to automate the configuration of the network hardware to support the network configuration in overlays created using VMware suits.

#### Use cases: Link to use case google doc

https://urldefense.proofpoint.com/v2/url?u=http-3A__docs.google.com_document_d_1qqrRN9KP8zeTvT-2Do72D4lvi9oJK5QSvZRB3gcOzOSzE_edit-3Fusp-3Dsharing-5Feil-26ts-3D5c047a37&d=DwMFaQ&c=HAkYuh63rsuhr6Scbfh0UjBXeMK-ndb3voDTXcWzoCI&r=5cGBwnyUCiMK2Tm6UX_iIiml04qYJeDz4xQzoG2Hi8E&m=jrfRo-WzWoRkcVRGhXwAA3bwPcouFDvHYtXazucaIOM&s=E126WUnmh_AX1G8Xd0v3wo6tDTTIOsvb8CHy4dEbRM4&e=

# 3. Proposed solution
#### Configuration of VLANs which are configured on Port Group within Distributed Virtual Switch, on TOR switch (like QFX) port connected to the host (ESXi) where this Port Groups have active instance of VM's. This will allow customers to configure the VLAN's on Port Group of the Distributed Virtual Switch and have that VLAN configuration reflect on the switch port connected to the host.  
#### vCenter Plugin running on the config node of the contrail will perform the necessary tasks to achieve the automatic configuration and deletion of the VLANs on Switch Port based on the VLANs configured on the Port Groups of a Distributed Virtual Switch. vCenter Plugin will have the following behavior:
     1. Plugin communicates with vCenter to listen to all the events related to vCenter like Port Group create/delete, VM create/delete and DVS etc.
     2. Plugin communicates with CFM to get the mapping between ESXi host and Switch Port (QFX).
     3. Plugin communicates with CFM to configure the Switch Port with VLANs corresponding to Port Group VLANs configured on Distributed Virtual Switch. 
     4. CFM will support both Brownfield and Greenfield topologys for DC fabric.
#### open issue ( )
Should the user be allowed the choice to modify/add the VLAN, instead of dynamically programming it as is done in APSTRA demo? As per Jacopo, we aren't required to cover this use case.
The solutions also doesn't address the notification/event handling from Fabric devices to handle device error/fault use cases.

# 4. User workflow impact
#### Users will use this feature to configure the underlay network of their private or public cloud based on overlay configuration. This solution specifically addresses the automatic configuration of VLAN's from overlay to underlay for a very specific case of VLAN's configured on Port Group of a Distributed Virtual Switch. VMware has many other combination of VLAN configuration on Port Groups, and also different variations of Port Group configuration also which we are not handling.

# 5. References
#### TBD.



# 1. Introduction
#### Design of VMware integration with contrail fabric manager. 

# 2. Problem statement
#### Customers using VMware vSphere product suits lack the capability to configure the networking hardware in their data-center deployment. VMware product suits allows customers to deploy, configure and manage private cloud infrastructure, and also public cloud via AWS but customers need to find ways to automate the configuration of the network hardware to support the network configuration in overlays created using VMware suits.

# 3. Proposed solution
#### Configuration of VLANs which are configured on Port Group within Distributed Virtual Switch, on TOR switch (like QFX) port connected to the host (ESXi) where this Port Groups have active instance of VM’s. This will allow customers to configure the VLAN’s on Port Group of the Distributed Virtual Switch and have that VLAN configuration reflect on the switch port connected to the host. vCenter Plugin running on the config node of the contrail will perform the necessary tasks to achieve the automatic configuration and deletion of the VLANs on Switch Port based on the VLANs configured on the Port Groups of a Distributed Virtual Switch. vCenter Plugin will have the following behavior:
 1. Plugin communicates with vCenter to listen to all the events related to vCenter like Port Group create/delete, VM create/delete and DVS etc.
 2. Plugin communicates with CFM to get the mapping between ESXi host and Switch Port (QFX).
 3. Plugin communicates with CFM to configure the Switch Port with VLANs corresponding to Port Group VLANs configured on Distributed Virtual Switch. 

Limitations:
- Not handling events/notification from underlay.
- Not modifying the vSphere attributes if any required to handle the event/notifications from underlay.


# 4. Alternatives considered
#### N/A

# 5. API schema changes
#### Data model changes to support infra servers.
#### API server changes to support the CRUD calls for infra servers.
#### Following VNC API’s for vCenter plugin to use are required:
     1. API to get the QFX-port to host mapping. This API should provide all the QFX-port to host mapping not a specific QFX-port to host mapping.
     2. API to configure the VLAN on QFX-port.
     3. API to get all the VLAN's configured on a given port of the switch.
   

# 6. UI changes
#### Describe any UI change applicable to this feature
#### Modifications to existing UI work flow with appropriate justifications

# 7. Notification impact
#### Describe any log, UVE, alarm changes
#### Describe all new elements added as part of this feature

# 8. Provisioning changes
#### Describe how this feature will be automatically provisioned
#### Describe all applicable envs like Kubernetes, RHOSP, JuJu, openshift, etc.
#### Describe any change to notifications provided to client systems

# 9. Implementation
#### Describe changes in all affected components such as controller, agent, etc.
#### Describe all data model schema changes applicable to this feature
#### Describe changes to key data structures
#### Describe synchronous/asynchronous nature of any API calls being made if any
#### Describe the impact on the feature during process restart if any
#### Describe all logs and debug messages associated with this feature
#### Describe all high availability considerations

# 10. Performance and scaling impact
#### API and control plane
#### Scaling and performance for API and control plane
#### Scaling and performance for API and forwarding

# 11. Upgrade
#### Describe upgrade impact of the feature, including ISSU
#### Schema migration/transition
#### Upgrade and restart will have same behavior. We will require to read all the VLANs already configured on each switch Port to handle upgrade and restart. Contrail Fabric Manager should provide API which provide all the VLAN's configured per Port of the physical switch. 

# 12. Deprecations
#### N/A

# 13. Dependencies
#### Contrail Fabric Manager should support data model changes required for infra servers. API server (VNC API) should provide the following API's:
     1. API to get the QFX-port to host mapping. This API should provide all the QFX-port to host mapping not a specific QFX-port to host mapping.
     2. API to configure the VLAN on QFX-port.
     3. API to get all the VLAN's configured on a given port of the switch.
     4. CFM will support both Brownfield and GreenField topology's for Data Center Fabric.
    
# 14. Testing
#### Unit tests
#### Dev tests
#### System tests

# 15. Documentation Impact

# 16. Reference
