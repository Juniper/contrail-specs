# 1. Introduction
This blueprint describes the requirements, design and implementation of Agent
TCP Ports to listen on specific address for Metadata Service instead of the
very generic __ANY__ (0.0.0.0) address.

# 2. Problem statement
The agent process runs a HTTP server for Openstack Metadata Service.

The concern that this document addresses is that the agent process
listens to __ANY__ IPv4 address belonging to self for the
metadata service instead of specific applicable address(es).
Listening on __ANY__ IPv4 address means that connection can be accepted
on any of the IPv4 address that the system has. By listening on specific
address, the intention is made clear as to what IP address and port is
used by the server for providing service.

# 3. Proposed solution
## 3.1 Metadata Service
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

## 3.2 Alternatives considered
None

## 3.3 API schema changes
Not Applicable

## 3.4 User workflow impact
None

## 3.5 UI changes
None

## 3.6 Notification impact
None

# 4. Implementation
## 4.1 Work items
1. Change Agent code for Metadata Service HTTP Server to make it listen on
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
1. Test cases for Metadata Proxy for vhost0 address.
   a. Test Openstack Metadata Service using metadata queries
      from VM. For example:
      $ curl http://169.254.169.254/openstack/2012-08-10/meta_data.json

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References

