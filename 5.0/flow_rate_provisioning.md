
# 1. Introduction
To support provisioning of flows which involves control of maximum flows and flow-rate at virtual-network level. 

# 2. Problem statement
Support control of maximum number of flows and setup rate of new flows in flows per second at VMI level by providing
configuration knobs at virtual-network level.

# 3. Proposed solution
The current flow rate provisioning supports control of maximum number of flows at VM level. The proposal is to
extend this by providing the following options
a) flow-rate configuration representing flow setup rate in flows-per-second. 
b) maximum number of flows.
Both the above knobs are provided at virtual network level and applied to each VMI in the virtual network.

## 3.1 Alternatives considered
None.

## 3.2 API schema changes
Schema has virtual-network-properties element of type VirtualNetworkType associated with virtual-network object.
The type VirtualNetworkType will be extended further to include max-flow-rate and max-flows attribute. This will allow 
max-flow-rate and max-flows to be configured at VN level.

## 3.3 User workflow impact
User is expected to configure max-flow-rate and max-flows at VN level for flow provisioning. Even though the configuration
is done at VN level, the config values are applied at VMI level.

### 3.3.1 max-flows interpretation
When we get flow setup request on a VMI which already has max-flows on it, vrouter-agent programsa flag on the VMI to
inform vrouter to not trap new flow requests to it on that VMI. The flag will remain on the VMI until the flow count on
the VMI reaches 90% of max-flows.

### 3.3.2 max-flow-rate interpretation
The max-flow-rate is an integer value representing maximum flows per second. When we get flow setup request on a VMI
whose current rate is greater than or equal to max-flow-rate, then new flows created on VMI will be made as SHORT flow
(which has action as DROP). Also vrouter will be programmed to NOT trap any additional flows on that VMI. The flag will
be reset every second, if it is set (because of exceeding max-flow-rate).

## 3.4 UI changes
Option to configure max-flow-rate and max-flows at VN level has to be provided under Configure->Networking->Networks

## 3.5 Notification impact

# 4. Implementation

## 4.1 Work items

### 4.1.1 Vrouter Agent changes
Contrail-vrouter-agent will parse the new configuration (done at VN level),
and store the values in VN and VMI operational objects. During flow setup, the
values of max-flows and max-flow-rate available on the VMI is consulted. If we
have matched or exceeded any of these values, then the new flow being setup will
be made as SHORT flow. The preference is given to max-flows first. When
max-flows permits flow to be setup, we check whether max-flow-rate also permits.
When max-flows does not permit flow to be setup, max-flow-rate will not be
consulted.

### 4.1.2 Vrouter changes
None.

## 4.2 Limitations
It is possible that some additional flows may get created (as short flows with Action as DROP) on VMI even when
max-flows/max-flow-rate has exceeded during transient scenarios. Before vrouter-agent could program vrouter to drop
new trap requests, it could have trapped few requests to vrouter-agent.
     
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
    (a) Create VN and configure max-flows on it. Verify that VMI and VN oper
        objects have it. Also verify additional flows are not created on this
        VMI after max-flows have been created.
    (b) Repeat the above test case with max-flow-rate.
    (c) Verify that when we reach 90% (or lower) of max-flows, new flows get
        created on the VMI.
    (d) Verify that additional flows (if any) created on VMI after max-flows/
        max-flow-rate has exceeded are Short flows.
    (e) Configure both max-flows and max-flow-rate and verify that max-flows is
        given preference over max-flow-rate.

## 9.2 Dev tests
    (a) Verify the flag on VMI to drop new flows is set in vrouter using vif
        utility when we have exceeded the limit.
    (b) Verify the above flag is reset when flows reach 90% (or lower) of
        max-flows.

## 9.3 System tests

# 10. Documentation Impact

# 11. References
