# 1. Introduction
Tungsten Fabric Manager (TFM) supports discovery and Zero Touch Provisioning (ZTP) of
network devices. To properly configure underlay for the infrastructure servers
(controller/computes), TFM needs to know about the connection details of
servers and switch ports. Also for baremetal workload, details about the
baremetal hardware needs to be added manually as of now. In the case of ZTP,
discovery of servers and auto-adding them to Tungsten Fabric database should be
provided.

# 2. Problem Statement
Adding Servers for provisioning or for adding baremetal nodes to
Tungsten Fabric database is a manual task as of now. With 100s of infrastructure
nodes/baremetal nodes to add, it becomes cumbersome to use UI. Also for
baremetal nodes, all hardware detail needs to be known and has to be entered.

So there is a requirement to automate this portion of the ZTP and discover all
possible servers.

### Use Case:
Server Discovery will enhance the capabilities of the Tungsten Fabric Manager
to auto-add the servers and their hardware details, network connections and
capabilities to Tungsten Fabric Manager. Also with possible integration of
Tungsten Fabric Manager with other tools, this may be used for re-imaging the
servers and bringing the servers with needed OS to provision controller and
computes.
Also this will provide connection details of baremetals to Fabric Manager,
which will configure the switches for providing underlay connectivity.

# 3. Proposed Solution
In order to address the above problems, we propose the following solution for
discovering baremetals and infra-servers.

Develop the ansible playbook that will integrate with thirdparty software e.g
Ironic. This ansible playbook may work in two modes, import the data from
thirdparty software or discovery based.

Discovery Based: Based on user provided IPMI subnets, IPMI credentials and
IPMI-Port, Ansible playbook will:
1. Check for active ip-addresses
2. Check for valid ipmi-address based on IPMI credentials and IPMI Port using
ipmitool
3. Check if valid ipmi-addresses-port are already available with Tungsten
Fabric Manager,
if yes, ignore the already available ipmi-addresses-port.
4. Push the ipmi-addresses list to thirdparty-software. e.g create ironic
nodes (in case of Ironic)
5. Trigger the Introspection, if available
6. Now wait till introspection completes
7. Authenticate with thirdparty software
8. Read all nodes and Interface details from thirdparty software,
9. Add them to the Tungsten Fabric Manager.

Import-Only: Baremetal and Infra-Servers may be imported from file or it may
be imported from thirdparty software e.g Ironic.
1. Authenticate with thirdparty software
2. Read all nodes and Interface details from thirdparty software,
3. Add them to the Tungsten Fabric Manager.

## 3.1 Alternatives considered
1. Use of integrated openstack and have reimaging and discovery as part of ZTP
Server.
2. Make use of standalone ironic instead of openstack integration

## 3.2 API schema changes
Add node and node-port schema to API Server. This is needed for the
creation discovered node, interfaces and creating reference b/w port and
discovered network-devices.

1. node: This resource will contain only hostname and will be referenced by
ports.

node {
- *hostname*: string
- *children* : port, port_group
- *ref*: node_profile
}
2. port: This resouce will contain mac_address and ref to parent object
port {
- *mac* : string
- *ref* : node_object
- *parent* : node 
}
3. port_group object will contain port objects and parent node ref
port_group {
- *port* : object
- *parent* : node
}

## 3.3 User workflow impact
1. Network device discovery needs to be made optional to allow repeat of
Server Discovery/Import
2. During ZTP, User will be able to provide additional information about
Server Discovery/Import.

## 3.4 UI Changes
UI need to allow user-input for following:
1. IPMI Subnet[s],
2. List of IPMI Credential
3. IPMI Ports or Ranges
4. Plugin Details (e.g ironic ):
  a. Auth URL
  b. Username/Password
  c. User Domain Name
  d. Project Name
  e. Project Domain Name


# 4. Implementation
For implementing the above requirements, it is proposed to make use of the
existing job template infra and implement an ansible-playbook for
server-discovery/import.

By using existing infra of job template, it will be easier to  provide
integrated ZTP workflow and user experience.

All user-input will passed to job_ctx.job_input. Ansible playbook will have
two tasks,
1. Server Discovery: This task may call filter_plugins or discovery module to
   do ping sweep, ipmi auth check, filter nodes from Tungsten Fabric Manager, then
   add nodes to plugin , and trigger plugin-based disocvery

2. Server Import: This task may call filter_plugins or import module to read
   from plugin based database or read from file. This task may need to wait
   for the introspection to complete, if introspection is in-progress. Once
   instrospection completes, this task will read data from plugin database and
   add node, ports and connection details to Tungsten Fabric Manager

# 5. Performance and scaling impact
No Performance and Scaling impact will be there as we are mostly reading from
thirdparty software.

# 6. Upgrade
Uprade from previously should not have any impact as there will be new addition to
schema (vnc schema and Tungsten Fabric schema).

# 7. Deprecations
N/A
# 8. Dependencies

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests
# 10. Documentation Impact
1. ZTP workflow needs to be updated for RHOSP and for centos
2. Server Discovery workflow needs to be updated.

# 11. References

# 12. Not Supported features for current release

