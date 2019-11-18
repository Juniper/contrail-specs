# 1. Introduction
This blueprint describes the requirements, design and implementation of Agent
TCP Ports TLS Support.

This document also talks about the need, and what, of the Agent process
listening on specific address for Port IPC and Metadata Service instead of
the very generic __ANY__ (0.0.0.0) address.

# 2. Problem statement
The agent process runs HTTP server for Port IPC Service. This service is not
capable of hanlding data using encryption and this can pose a security risk.

The other problem that this document and feature addresses is that the agent
process listens to __ANY__ IPv4 address belonging to self for the Port IPC
and the metadata service instead of specific applicable address(es).

# 3. Proposed solution
## 3.1 Port IPC Service
The Agent service Port IPC is used for NOVA plugin to allow add/delete and
enable/disable of the VMI ports in the agent.  The communication is based on
HTTP and is not encrypted.  This communication needs to be encrypted by making
use of TLSv1.2.

The will require changes to Agent and also to the plugin that is used to
communicate wth Agent via SSL on "127.0.0.1:9091".

Note that the connection listen is on "127.0.0.1:9091" instead of existing, more
generic "0.0.0.0:9091" and is incorporated as part of this feature itself.

All clients, namely, NOVA plugin, MESOS plugin, K8S plugin and vCenter plugin
use the following configuration information to decide on communication mechanism
with Port IPC server running in Agent.

The SSL parameters in the contrail-vrouter-agent.conf, which is the server side:
[PORT-IPC]
port_ipc_ssl_enable=True
port_ipc_key=/etc/contrail/ssl/private/server-privkey.pem
port_ipc_cert=/etc/contrail/ssl/certs/server.pem
port_ipc_ca_cert=/etc/contrail/ssl/certs/ca-cert.pem

The SSL parameters on the client side:
[PORT-IPC]
port_ipc_use_ssl=True
port_ipc_client_key=/etc/contrail/ssl/private/server-privkey.pem
port_ipc_client_cert=/etc/contrail/ssl/certs/server.pem
port_ipc_client_ca_cert=/etc/contrail/ssl/certs/ca-cert.pem

The above parameters should be present in /etc/contrail/port-ipc.conf
on the:
1. nova_compute container in the case of OpenStack
2. ?? container in the case of K8S
3. contrail-vcenter-manager in the case on vCenter

## 3.2 Metadata Service
Also, Agent Metadata Service is used for providing proxy service between VM and
Openstack.  This service operates on port 8097, meaning agent listens on
"0.0.0.0:8097" to receive metadata requests from VMs.  Agent proxies this to the
openstack using another HTTP connection to it.  This connection to openstack is
optionally encrypted based on the configuration options chosen.

This feature will allow metadata service to listen on the Agent router id and
port 8097 instead of above mentioned "0.0.0.0:8097".

The SSL configuration fields for metadata service are:
    metadata_use_ssl=True
    metadata_client_key=/etc/contrail/ssl/private/server-privkey.pem
    metadata_client_cert_type=PEM
    metadata_client_cert=/etc/contrail/ssl/certs/server.pem
    metadata_ca_cert=/etc/contrail/ssl/certs/ca-cert.pem

The above mentioned SSL configuration fields is handled already in the
current implementation.

## 3.3 Alternatives considered
None

## 3.4 API schema changes
Not Applicable

## 3.5 User workflow impact
Please see above for the required configuration to be done.

## 3.6 UI changes
None

## 3.7 Notification impact
None

# 4. Implementation
## 4.1 Work items
1. Agent changes for Port IPC server to make use of SSL parameters.

2. Plug-in changes for the following:
   - Openstack
     Python plugin changes to use either http or https based on configuration.
   - K8S
     GO plugin changes to use either http or https based on configuration.
   - MESOS
     ??
   - vCenter
     Python (CVM) plugin changes to use either http or https based on
     configuration. CVM plugin for reading vRouter ports creates own
     HTTP session to agent API. But, for creating and deleting ports
     contrail-vrouter-api is used. This package calls vrouter-port-control
     which connects to agent API. These two packages will be modified
     to enable TLS connection.

3. Deployer/Provisioning changes -
   Priority order for the deployers is the following one:
   - RHOSP /TripleO
   - Ansible-deployer
   - OpenShift
   - Juju
   - Helm

# 5. Performance and scaling impact
TODO

## 5.1 API and control plane
None

## 5.2 Forwarding performance
None

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
1. Add new test cases for Port IPC SSL with loopback address.
2. Add new test cases for Metadata Proxy for vhost0 address.
3. Negative tests.

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
