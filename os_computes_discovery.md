# 1. Introduction
Tungsten Fabric Manager (TFM) supports discovery of network devices. With
Networking Tungsten Fabric (NTF, aka ML2 plugin), TFM will be able to manage network devices
connected to OpenStack compute nodes. Compute can be OVS or SRiOV based.
NTF needs to know the compute node - switch connection details.

# 2. Problem Statement
To configure network devices, NTF needs to know compute node - switch connection:
physical interfaces for each node and how they are connected to network devices (port and switch names)
These details will be able to import via YAML file import. However, defining compute nodes connections
to the fabric devices in file could be error-prone, especially for larger data centers.
Hence, auto-discovery of the servers is desired.

# 3. Proposed Solution
In order to address the above problem, we propose the following solution for
compute nodes discovering:

Develop the computes autodiscovery job that will:
* Identify all leaf switches from fabric provided as input

Then, for each leaf switch:
* Collect leaf's LLDP neighbors, executing LLDP client (`show lldp neighbors` for QFX switches)
* Get detailed information about each LLDP neighbor,
  executing LLDP client for particular interface (`show lldp neighbors interface interface_name` for QFX switches)
* Parse LLDP client output in order to obtain link local information
* Create/Update proper Node objects in Config API for all neighbors identified as compute nodes
* Create/Update proper Port object for each compute node’s network interface in Config API
* For each modified Port object, find the Physical Interface object (representing a port on a switch)
  based on LLDP client output, and connect these objects

## 3.1 Requirements
* Fabric devices needs to be onboarded
* All compute nodes send LLDP packets to leaf switches
* Leaf switches have enable LLDP on all data plane ports
* LLDP packets from compute nodes have to have information about the name of the given compute name (host name)
  in Openstack. NTF needs to match host information attached to Neutron’s event with Node objects in Config API.
  Also, we want eliminate non compute machines out of autodiscovery consideration
* System Description field in LLDP packets has to contain "node_type: <OVS/SRiOV>"
  what allows them to identify it as compute node and its type. NTF handles OVS and SRiOV computes slightly differently.

## 3.2 API schema changes
Compute autodiscovery job requires that new node types (OVS, SRiOV) are added to Node object schema

## 3.3 User workflow impact

## 3.4 UI Changes
UI need to allow user trigger computes discovery for selected fabric.
Rest of needed parameters should be already in Contrail

# 4. Implementation
For implementing the proposed solution, it is proposed to make use of the existing job template infra and
implement an ansible-playbook for compute nodes discovery.

The job will log about:
* Currently processing leaf switch
* Found LLDP neighbors
* Identified compute node and its physical interfaces
* Links between discovered compute and processing leaf

# 5. Performance and scaling impact

# 6. Upgrade

# 7. Deprecations

# 8. Dependencies

# 9. Testing

* Initial servers discovery on freshly created fabric:
  * Node and Port should be created in Config API accordingly to existing computes.
  * Created links between Port and Physical Interfaces according to physical connections.

* New compute node connected to the fabric:
  * Node and Port objects should be created in Config API for newly added compute.
  * Created links between Port and Physical Interfaces according to new compute connections.

* Removed compute node previously connected to the fabric:
  * Node and Port objects related to removed compute should be removed from Config API

* One of compute's interfaces now connected to different device:
  * Port object related to updated interface now should be linked with proper Physical Interface, to
    represent current connection state.

# 10. Documentation Impact

# 11. References

# 12. Not Supported features for current release
