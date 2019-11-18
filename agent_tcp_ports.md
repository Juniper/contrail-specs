# 1. Introduction
This blueprint describes the requirements, design and implementation of Agent
TCP Ports to listen on specific address for Port IPC and Metadata Service
instead of the very generic __ANY__ (0.0.0.0) address.

# 2. Problem statement
The agent process runs two HTTP servers, one for Port IPC Service and for
Openstack Metadata Service.

The concern that this document addresses is that the agent process
listens to __ANY__ IPv4 address belonging to self for the Port IPC
and the metadata service instead of specific applicable address(es).
Listening on __ANY__ IPv4 address means that connection can be accepted
on any of the IPv4 address that the system has. By listening on specific
address, the intention is made clear as to what IP address and port is
used by the server for providing service.

# 3. Proposed solution
## 3.1 Port IPC Service
The Agent service Port IPC is used for NOVA plugin to allow add/delete and
enable/disable of the VMI ports in the agent.  The communication is based on
HTTP.

The Agent will require changes to its HTTP server so that the nova plugin
that is used to communicate with is on "127.0.0.1:9091".

Note that the connection listen is on "127.0.0.1:9091" instead of existing, more
generic "0.0.0.0:9091".

All clients, namely, NOVA plugin, MESOS plugin, K8S plugin and vCenter plugin
connects to the address "localhost:9091" to communicate with Agent for Port-IPC.

## 3.2 Metadata Service
Also, Agent Metadata Service is used for providing proxy service between VM and
Openstack.  This service operates on port 8097, meaning agent listens on
"0.0.0.0:8097" to receive metadata requests from VMs.  VMs request metadata
information from the Openstack using the address 169.254.169.254:80 which is
passed through NAT translation for the Agent that it receives request to
<Agent-Router-IP:8097>.  Agent proxies this to the openstack using another
HTTP connection to Openstack.  This connection to openstack is optionally
encrypted based on the configuration options chosen.

This feature will change metadata service to listen on the <Agent-Router-IP> and
port 8097 instead of above mentioned "0.0.0.0:8097".

## 3.3 Alternatives considered
None

## 3.4 API schema changes
Not Applicable

## 3.5 User workflow impact
None

## 3.6 UI changes
None

## 3.7 Notification impact
None

# 4. Implementation
## 4.1 Work items
1. Change Agent code for Port IPC REST HTTP Server to make it listen on
   "127.0.0.1:9091".
2. Change Agent code for Metadata Service HTTP Server to make it listen on
   "<agent-vhost0-ip>:8097".

# 5. Performance and scaling impact
None

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
Since there is no framework for UT, manual UT will be done. The following
are the details of the UT that will be performed:
1. Test cases for Port IPC with loopback address.
   a. Test VM Addition and Deletion (Openstack Environment)
   b. Test Container Addition and Deletion (K8S Environment)
2. Add new test cases for Metadata Proxy for vhost0 address.
   a. Test Openstack Metadata Service using metadata queries
      from VM. For example:
      $ curl http://169.254.169.254/openstack/2012-08-10/meta_data.json

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References

