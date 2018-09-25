
# 1. Introduction
Many of the key features of vRouter are dependent on flows, but vRouter
performance is tightly constrained by its rate of flow creation. Problems will
occur if the rate of new flow creation at the physical host level exceeds
vRouter capacity and these problems may impact VMs on that host regardless of
whether a specific VM is involved in the high rate condition.

When the rate of flow creation at the physical server level is exceeded, all VMs
on that server will be impacted. Although it is at least somewhat acceptable
that a VM which is receiving or generating excessive traffic will be impacted,
it is not acceptable for uninvolved VMs to be impacted just because they happen
to be on the same server. A mechanism is required to protect uninvolved VMs.
#To support provisioning of flows which involves control of maximum number of
#flows at virtual network and virtual machine interface level.

# 2. Problem statement
The current control of maximum flow rate is inadequate.
#Support control of maximum number of flows at VMI level by providing
#configuration knobs at virtual machine interface(VMI) and virtual network level
#for all VMIs in virtual network, preference is given to max-flows configured at
#VMI level.

# 3. Proposed solution
The current flow rate provisioning supports control of maximum number of flows
at VM level. The proposal is to extend this by providing the following option:
a) maximum number of flows (max-flows in schema) at virtual network level.
b) maximum number of flows (max-flows in schema) at virtual machine interface
level.
The knob max-flows is provided at virtual network level and applied to each
VMI in the virtual network. It can also be given at VMI level.
Preference: When max-flows knob is configured both at virtual network and for
VMI then max-flows configured for VMI will be used to control maximum flows for
that VMI.

## 3.1 Alternatives considered None.

## 3.2 API schema changes
a) Virtual Network Change:
Schema has virtual-network-properties element of type VirtualNetworkType
associated with virtual-network object. The type VirtualNetworkType will be
extended further to include max-flows attribute. This will allow max-flows to be
configured at VN level for all virtual-machine-interfaces linked to the
virtual-network.

b) Virtual Machine Interface Change:
Schema has virtual-machine-interface-properties element of type
VirtualMachineInterfacePropertiesType associated with virtual-machine-interface
object, max-flows attribute will be added to
VirtualMachineInterfacePropertiesType allowing configuration of max-flows at
virtual-machine-interface level.

## 3.3 User workflow impact
User is expected to configure max-flows at VN level for flow provisioning. Even
though the configuration is done at VN level, the config values are applied at
VMI level. Aditionally max-flows can be configured at virtual-machine-interface
level to control maximum flows for that specific VMI.

### 3.3.1 max-flows interpretation
When we get flow setup request on a VMI which already has max-flows on it,
vrouter-agent programs flag on the VMI to inform vrouter to not trap new flow
requests to it on that VMI. The flag will remain on the VMI until the flow count
on the VMI reaches 90% of max-flows. (90% limit is a performance constraint)

## 3.4 UI changes
a) Option to configure max-flows at VN level has to be provided under
Configure->Networking->Networks.

b) Option to configure max-flows at VMI level has to be provided under
Configure->Networking->Ports->Advanced Options.

## 3.5 Notification impact

# 4. Implementation

## 4.1 Work items

### 4.1.1 Vrouter Agent changes
Contrail-vrouter-agent will parse the new configuration (done at VN level and
VMI level if max-flows is configured at VMI then this value is used), and store
the values in VN and VMI operational objects.  During flow setup, the value of
max-flows available on the VMI is consulted. If we have matched or exceeded the
value, then the new flow being setup will be made as SHORT flow. The preference
is given to max-flows first, when flow control is configured at VM level also.

### 4.1.2 Vrouter changes
Flows dropped due to max-flows limit is tracked in vrouter dropstats utility
with counter: New Flow Drops Changes: None.

## 4.2 Limitations
It is possible that some additional flows may get created (as short flows with
Action as DROP) on VMI even when max-flows has exceeded during transient
scenarios. Before vrouter-agent could program vrouter to drop new trap requests,
it could have trapped few requests to vrouter-agent.

# 5. Performance and scaling impact
## 5.1 API and control plane
#### Scaling and performance for API and control plane

## 5.2 Forwarding performance
#### Scaling and performance for API and forwarding

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None.

# 9. Testing
## 9.1 Unit tests
a) Create VN and configure max-flows on it. Verify that VMI and VN oper objects
have it. Also verify additional flows are not created on this VMI after
max-flows have been created.
b) When max-flows is configured both for VN and VMI validate max-flows from VMI
object is used for flow setup limit.
c) Verify that when we reach 90% (or lower) of max-flows, new flows get created
on the VMI.
d) Verify that additional flows (if any) created on VMI after max-flows
has exceeded are Short flows.
e) Configure both max-flows and max_vm_flows in conf file and verify that
max-flows is given preference over max_vm_flows configured at VM level.

## 9.2 Dev tests
a) Verify the flag on VMI to drop new flows is set in vrouter using vif utility
when we have exceeded the limit.
b) Verify the above flag is reset when flows reach 90% (or lower) of max-flows.

## 9.3 System tests

# 10. Documentation Impact

# 11. References
