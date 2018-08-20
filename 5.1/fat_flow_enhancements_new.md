
# 1. Introduction
The purpose of this document is to describe the fat flow enhancements done as part of R5.0.1 and R5.1.

# 2. Problem statement

Existing fat flow implementation (pre R5.0.1) resulted in some of the services like BGPaaS and SSH to metadata IP not work 
when fat flows were configured. This was because fat flows were being created for internal VN subnet .1, .2 IP addresses
and 169.254/16 subnets. Once fat flows were created for these internal IP addresses, depending on configuration, certain type of 
NAT wouldn't work, breaking the service. To address this problem, the concept of exclude prefix list is introduced in R5.0.1. 

Flow setup rate is still challenged in SP environments (esp. when dealing with failure scenarios or large IPV4/IPv6 pool 
-- there are millions of subscribers = SRC IPs). These were addressed through fat flow ignore Src or Dst IP feature in R5.0
to some extent, but there was loss of entropy as the Src or Dst IP addresses were masked out to 0 and flow scaling was limited 
in the usecase given below. An intermediate solution is required which can give reasonable entropy of flows as well as conserve 
flow table resources by creating fat flows for a group of flows. The degree to which the grouping is done would be configurable. 
The usecase assumes a typical internet access usecase for residential services (broadband or mobile).
In this case, most of the connectivity(flows) is setup from SP subscribers to the Internet.
 - SP subscribers have known IP address pool (say 10/8)
 - Internet has many
It is assumed that the first packet has SRC in the SP pool (here 10/8).
The requirement is to group flows from 10/8 into smaller groups of say 10/28 where /28 prefix length would be configurable.


# 3. Proposed solution

Fat flow exclude prefix list:
-----------------------------
This is an internal feature (not customer visible), where an exclude prefix list per VMI is sent along with the fat flow
configuration to the vrouter. The fat flow configuration is not applied to flows whose prefixes matched with the prefixes 
present in the exclude list. All the internal VN subnet .1, .2 and 169.254/16 prefixes are part of the exclude list. 
This results in normal flows being created for the internal IPs and they fall outside the purview of fat flow configuration.
This feature is supported for internal IPv6 addresses as well.

Fat flow based on configurable prefix length:
---------------------------------------------

As given in problem statement, contiguous flows in the same subnet would be grouped into a fat flow having same protocol 
and port numbers configured.

For eg:

Subscriber pool = 10/8 
Fat flow configured for protocol 17 and port 53, subnet 10/8, prefix length /28.

Lets say the original flows are:-
f1 = 10.0.0.1, 8.8.8.8, 17, 5000, 53 
f2 = 10.0.0.2, 27.28.2.1, 17, 27828, 53 
...
f15 = 10.0.0.15, 30.28.2.1, 17, 27828, 53 

then we would create one fat flow ff1 as given below representing all the flows from f1 to f15.

ff1 = 10.0.0.0/32, 0, 17, 0, 53

Similar mechanism will apply for IPv6 flows.
The fat flows would be created with the given fat flow prefix length only if there is a match for subnet configured in fat flow
with the source or destination IP address of the flow. 

## 3.1 Alternatives considered
None

## 3.2 API schema changes

Fat flow exclude prefix list:
-----------------------------
None.

Fat flow based on configurable prefix length:
---------------------------------------------
A new enumeration "Match Prefix" would be added to FatFlowIgnoreAddressType to indicate fat flow creation through prefix length method.
A new element of type SubnetType would be added to ProtocolType to give the subnet to be matched while creating fat flows.
A new element of type Integer would be added to ProtocolType to specific the prefix length to be used while creating fat flows.

## 3.3 User workflow impact

Fat flow exclude prefix list:
-----------------------------
None.


Fat flow based on configurable prefix length:
---------------------------------------------
The existing workflow of fat flow configuration at VMI or VN level would continue to be the same. The configuration would be 
extended to allow creation of fat flows based on configurable prefix length.

## 3.4 UI changes

Fat flow exclude prefix list:
-----------------------------
None.

Fat flow based on configurable prefix length:
---------------------------------------------
The existing fat flow UI under VMI and VN would be extended to allow taking following inputs for this feature:-
1. An enumeration to indicate that fat flows should be created using prefix length method. This can ideally go as a 
   new element "Match prefix" under the "Ignore Address" drop down.
2. A subnet in IPv4 or IPv6 format along with subnet mask which will be used to match the incoming flow's subnet.
   Eg: 10.0.0.0/8 or 2000::1/64
3. An integer to indicate prefix length with which fat flows will be created.

NOTE: PLM/UI team can add more details here as required.

## 3.5 Notification impact
None

# 4. Implementation

## 4.1 Fat flow exclude prefix list

Whenever a fat flow configuration is changed, the agent module is enhanced to send the fat flow exclude list per VMI 
via netlink messages to vrouter. The VN .1, .2, 169.254/16 and IPv6 link local IP addresses are part of the exclude list.
The vrouter module receives this information and stores it in its data structure which represents the vif interface.
Whenever a new flow is created through a VMI, the vrouter modules checks if the source or destination IP of the flow matches with
any of the IPs in the exclude list in the VMI. If there is a match, a normal flow is created instead of a fat flow.
The same flow key (6 tuple) is passed to agent which does further flow processing and establishes the flow.

## 4.2 Fat flow based on configurable prefix length

### 4.2.1 UI
To be added by UI team

### 4.2.2 Agent

At a high level following changes would be required on the agent side:-
1. Data structure changes to represent new fat flow config.
2. Parse the new fields in fat flow config and store it in VMI object.
3. Propogate new fat flow config information at VN level to VMI level (TBD if any changes are actually required here).
4. Encode and send the new fat flow config information at VMI level to vrouter via existing netlink message.
5. Handle trap from vrouter to agent when 1st pkt of the flow is received and install fat flow in vrouter.


### 4.2.3 Vrouter

At a high level following changes would be required on the vrouter side:-
1. Data structure changes to represent new fat flow config at vif.
2. Decode the new fat flow configuration received at VMI level and store it in the vif data structure.
3. Fat flow config lookup in IPv4 and IPv6 pkt path to see if there is a match and do the required processing
   viz. form the flow key based on fat flow config.
4. Store the src and destination IP address mask as part of flow key (TBD - is this really required?)
5. Changes to vif utility to display new fat flow config.
6. Changes to vrmemstats utility to display new stats related to mem alloc/free.

# 5. Performance and scaling impact
## 5.1 API and control plane

No impact.

## 5.2 Forwarding performance

There may be a negligible impact to forwarding performance due to additional if checks added. Maybe <= 1% impact.

# 6. Upgrade

No impact envisioned.

# 7. Deprecations

None.

# 8. Dependencies
Dependency on UI to be ready during UT/Integration testing.

# 9. Testing
## 9.1 Unit tests

(a) Create fat-flow configuration at interface level and verify new fields
    of fat-flow config are present in agent's oper table.
(b) Create fat-flow configuration at VN level and verify that fat-flow
    config is present in all the interfaces of VN in agent's oper table.
(c) Verify that selecting match prefix and specifying subnet address, mask and prefix length 
    results in creating fat flow with the given prefix when source ip of the flow falls in the configured
    subnet. Send traffic for multiple flow with same subnet and check if all the flows map to the fat flow.
(d) Verify that selecting match prefix and specifying subnet address, mask and prefix length 
    results in creating fat flow with the given prefix when destination ip of the flow falls in the configured
    subnet. Send traffic for multiple flow with same subnet and check if all the flows map to the fat flow.
(e) Verify (c) and (d) with IPv6 addresses.
(f) Verify flows matching exclude prefix list are not created as fat flows.


## 9.2 Dev tests

(a) Configure fat-flow and verify fat-flow configuration in vrouter using vif utility. 
    Make sure the matched subnet, mask and fat flow prefix length are displayed.
(b) Configure fat-flow and verify that both agent and vrouter compute same flow keys and 
    ensure there are no Hold flows.


## 9.3 System tests

(a) Create multiple fat-flow records on same interface. Send traffic matching different records and 
    verify that right fat-flow record is matched for each traffic.
(b) Try adding/updating/deleting the fat flow config and verify there are no memleaks using vrmemstats utility.


# 10. Documentation Impact

# 11. References

https://github.com/Juniper/contrail-specs/blob/master/5.0/fat_flow_enhancements.md

