
# 1. Introduction
To enhance fat-flow feature to support ignore of both source and destination
ports and/or ignore or either source or destination IP address.
This document describes the design and implementation details of Contrail
components to achieve this.

# 2. Problem statement
Support aggregation of multiple flows into a single flow by ignoring both source
and destination ports and/or by ignoring either source or destination IP. Also
support fat-flow configuration at VN level.

# 3. Proposed solution
The current fat flow solution supports aggregation of flows by ignoring either
source or destination port for a given protocol. The proposal is to extend this
by providing the following options
a) Ignore both source and destination ports
b) Ignore either source or destination IP
c) Combination of both (a) and (b) above.

Also fat-flow in the current form is supported only at VMI level. This will be
extended to make it configurable at VN level. When it is configured at VN level,
it will be applied to all VMIs under that VN.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
Schema has fat-flow-protocol object which is of type ProtocolType. This
ProtocolType object will be extended further to ignore source or destination IP.
Also the port field of ProtocolType can now hold 0 as a valid value. The value 0
indicates that both source and destination ports need to be ignored. Also a new
property named virtual-network-fat-flow-protocols for virtual-network object
will be defined. This will allow fat-flow to be configured at VN level.

## 3.3 User workflow impact
User is expected to create fat-flow-protocol object and associate it at VMI or
VN level.

### 3.3.1 Ignore address interpretation
Each fat-flow configuration record will have a field named ignore-address which
can value of "none", "source" or "destination"

#### Source
For packets from local VM, “source” indicates the source-IP of the packet. For
packets from physical interface "source" indicates destination-IP of the packet.

#### Destination
For packets from local VM, “destination” indicates the destination-IP of the
packet. For packets from physical interface "destination" indicates source-IP of
the packet.

#### None
Neither source nor destination IP is ignored in this case.

### 3.3.2 Port
Each fat-flow configuration record already has port field. This field will now
take 0 as a possible value. Value 0 indicates both source and destination
ports will be ignored.

### 3.3.3 Configuration for sample use-case

                        -------------
    VN1 traffic--------|    SI       |-------------Internet
                        -------------

In the above figure, traffic from different sources of VN1 is going to Internet
via service instance SI. Since each source in VN1 can visit different sites in
Internet, the user might want to ignore all the Internet addresses in the flows.

To achieve this, we need the following configuration:

case 1:
source, SI and destination are in different computes.
Left Interface of SI -- Ignore source
Right Interface of SI -- Ignore destination

case 2:
source and SI are in same compute but destination is in different compute
source interface -- Ignore destination
Left Interface of SI -- Ignore source
Right Interface of SI -- Ignore destination

case 3:
source is in different compute but  SI and destination are in same compute
Left Interface of SI -- Ignore source
Right Interface of SI -- Ignore destination
destination interface -- Ignore source

## 3.4 UI changes
In Configure->Networking->Ports, when we edit any of the ports, a window for
that port opens. Here under Fat-Flow(s) section we can create and edit fat-flow
configuration. A new field for each fat-flow record named "Ignore Address" will
be added. This field can take values of "None", "Source" or "Destination".
The existing Port field of each fat-flow record will start taking 0 also as
value.

Similarly options to configure fat-flow at VN level will be provided under
Configure->Networking->Networks

## 3.5 Notification impact
None.

# 4. Implementation

## 4.1 Work items

### 4.1.1 Vrouter Agent changes
Contrail-vrouter-agent will send fat-flow configuration to vrouter via
Netlink messages at per interface level. Even when fat-flow is configured at VN
level, agent will map it to all the interfaces of that VN and send this per
interface fat-flow config to vrouter.
When vrouter-agent receives request to create flows from vrouter, it checks for
fat-flow configuration and maps it to existing flow or creates new flow. The
fat-flow configuration helps in determining the key for Flow.


### 4.1.2 Vrouter changes
Vrouter will receive fat-flow configuration from vrouter-agent. It uses this
configuration to determine the keys for flows. On receiving new packet, vrouter
creates flow entry in hold state and notifies agent about this. The agent on
processing this notification adds flow entry to vrouter and the entry moves out
of hold state.


## 4.2 Limitations
1. Fat-flow configuration to ignore source or destination address will not be
applied for NAT flows.
2. ECMP between VMIs of same compute and fat-flow configuration on these VMIs is
not a supported configuration. This may lead to some unexpected behavior.

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
This is an enhancement to existing fat-flow feature, so it is dependent on that.

# 9. Testing
## 9.1 Unit tests
    (a) Create fat-flow configuration at interface level and verify new fields
        of fat-flow config are present in agent's oper table
    (b) Create fat-flow configuration at VN level and verify that fat-flow
        config is present in all the interfaces of VN in agent's oper table.
    (c) Verify that port 0 of fat-flow config results in creation of flows
        with both source and destination ports as 0.
    (d) Verify source/destination/none values for ignore-address config of
        fat-flow results in creation of flows with proper keys.
    (e) Verify combination of both (c) and (d) in a single fat-flow record

## 9.2 Dev tests
    (a) Configure fat-flow and verify fat-flow configuration in vrouter using
        vif utility.
    (b) Configure fat-flow and verify that both agent and vrouter compute same
        flow keys and ensure there are no Hold flows.

## 9.3 System tests
    (a) Create fat-flow with ignore source/destination on interface which has
        Floating-IP configured. Verify that ignore source/destination is not
        applied to floating-IP traffic.
    (a) Create multiple fat-flow records on same interface. Send traffic
        matching different records and verify that right fat-flow record is
        matched for each traffic.

# 10. Documentation Impact

# 11. References
