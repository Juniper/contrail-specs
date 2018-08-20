
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

To solve the scaling problem given in the usecase, contiguous flows in the same subnet would be grouped into a fat flow having 
same protocol and port numbers configured. This would work along with the existing fat flow Ignore Src or Dst feature incase 
Src/Dst IP address is to be completedly ignored. The convention of Src and Dst will be same as existing convention implemented 
for Ignore address in R5.0. 

Depending on configuration, the configurable prefix length feature would be applicable only on Src, Dst or both the IP addresses 
of the flow. Fat flows would be created with the given fat flow prefix length only if the flow Src/Dst IP address is not part of 
exclude list of the VMI and there is a match for subnet configured with the Src and/or Dst IP address of the flow. 

For eg 1:

Subscriber pool = 10/8 
Fat flow configuration for VMIx:-
	protocol - 17
        port     - 53
        Ignore Address - Dst
        Src prefix aggregation - Subnet 10/8, fat flow prefix length 28
        
Lets say the original flows coming out from VMIx are:-
f1 = 10.0.0.1, 8.8.8.8, 17, 5000, 53 
f2 = 10.0.0.2, 27.28.2.1, 17, 27828, 53 
...
f15 = 10.0.0.15, 30.28.2.1, 17, 27828, 53 

then we would create one fat flow ff1 as given below representing all the flows from f1 to f15.

ff1 = 10.0.0.0/28, 0, 17, 0, 53

Similar mechanism will apply for IPv6 flows.

For eg 2:

Src pool = 10/8
Dst pool = 20/8
Fat flow configuration for VMIx:-
	protocol - 6
        port     - 0
        Ignore Address - None
        Src prefix aggregation - Subnet 10/8, fat flow prefix length 28
        Dst prefix aggregation - Subnet 20/8, fat flow prefix length 28
        
Lets say the original flows coming out from VMIx are:-
f1 = 10.0.0.16, 20.0.0.1, 6, 5000, 22 
f2 = 10.0.0.17, 20.0.0.2, 6, 5001, 22 
...
f14 = 10.0.0.31, 20.0.0.15, 6, 28473, 53 
f15 = 10.0.0.31, 20.0.0.17, 6, 26492, 22 

then we would create two fat flows ff1 and ff2 as given below -

ff1 = 10.0.0.16/28, 20.0.0.0/28, 6, 0, 0 --> corresponding to f1 to f14
ff2 = 10.0.0.16/28, 20.0.0.16/28, 6, 0, 0  --> corresponding to f15


## 3.1 Alternatives considered
None


## 3.2 API schema changes

Fat flow exclude prefix list:
-----------------------------

None.

Fat flow based on configurable prefix length:
---------------------------------------------

A new element "Src-Prefix" of type SubnetType would be added to ProtocolType to match Src IP (for flows exiting the VMI)
or Dst IP (for flow incoming to the VMI) of the flows. 
A new element "Src-Prefix-length" of type Integer would be added to ProtocolType to specific the Src prefix length to be used.
A new element "Dst-Prefix" of type SubnetType would be added to ProtocolType to match Dst IP (for flows exiting the VMI)
or Src IP (for flow incoming to the VMI) of the flows. 
A new element "Dst-Prefix-length" of type Integer would be added to ProtocolType to specific the Dst prefix length to be used.


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
1. A Src subnet in IPv4 or IPv6 format along with subnet mask which will be used to match the flow's subnet.
   Eg: 10.0.0.0/8 or 2000::1/64
2. An integer to indicate Src prefix length with which fat flows will be created.

Both 1 & 2 would be collectively known as Prefix-aggregation-Src.

3. A Dst subnet in IPv4 or IPv6 format along with subnet mask which will be used to match the flow's subnet.
   Eg: 10.0.0.0/8 or 2000::1/64
4. An integer to indicate Dst prefix length with which fat flows will be created.

Both 3 & 4 would be collectively known as Prefix-aggregation-Dst.

The new UI would interact with the existing fat flow UI in the following ways:-

a. The semantics of Src and Dst will be same as today's implementation.
b. When None is selected in the Ignore-address, then both Prefix-aggregation-Src and Dst will be applicable i.e items 1 to 4
   above would be active.
c. When Src is selected in the Ignore-address, then only Prefix-aggregation-Dst will be applicable i.e items 3 & 4 would be active.
d. When Dst is selected in the Ignore-address, then only Prefix-aggregation-Src will be applicable i.e items 1 & 2 would be active.
e. Prefix-aggregation-Src and Prefix-aggregation-Dst both are optional items and will be considered only if they are configured. 

PLM/UI team to add more if required.

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
6. Flow key changes to include source IP prefix length and destination IP prefix length.


### 4.2.3 Vrouter

At a high level following changes would be required on the vrouter side:-
1. Data structure changes to represent new fat flow config at vif.
2. Decode the new fat flow configuration received at VMI level and store it in the vif data structure.
3. Fat flow config lookup in IPv4 and IPv6 pkt path to see if there is a match and do the required processing
   viz. form the flow key based on fat flow config.
4. Store the src and destination IP address mask as part of flow key and changes related to that. 
5. Changes to vif utility to display new fat flow config.
6. Changes to flow utility to display flow source and destination IP with prefix length.
7. Changes to vrmemstats utility to display new stats related to mem alloc/free.

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

(a) Create fat-flow configuration at interface level and verify new fields of fat-flow config 
    are present in agent's oper table.
(b) Create fat-flow configuration at VN level and verify that fat-flow config is present in all the 
    interfaces of VN in agent's oper table.
(c) Verify behaviour when selecting Ignore-Address Src and specifying Prefix-aggregation-Dst for flows created in 
    VMI to fab and fab to VMI directions. Send traffic for multiple flow with same subnet and check if all the flows 
    map to the fat flow.
(d) Verify behaviour when selecting Ignore-Address Dst and specifying Prefix-aggregation-Src for flows created in 
    VMI to fab and fab to VMI directions. Send traffic for multiple flow with same subnet and check if all the flows 
    map to the fat flow.
(e) Verify behaviour when selecting Ignore-Address None and specifying both Prefix-aggregation-Src and Prefix-aggregation-Dst
    for flows created in VMI to fab and fab to VMI directions. Send traffic for multiple flow with same subnet and check if 
    all the flows map to the fat flow.
(f) Verify (c) to (e) with IPv6 addresses.
(g) Verify that flows matching exclude prefix list are not created as fat flows.


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

