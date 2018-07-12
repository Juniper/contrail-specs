
# 1. Introduction
This documents describes the proposed changes to intorduce genric
smart NIC offload support for Tungsten Fabric vRouter data-path.


# 2. Problem statement
There is a wide range of smart/intelligent NICs emerging on the market, 
implemented using various technologies. These NICs are capable of being
programed to perform advanced packet classification and modification
operations, allowing to implement flexible packet processing pipelines
implementing switching, routing, tunneling, etc.

The goal of this feature is to define a framework and initial implementation 
for accelerating vRouter packet processing/forwarding performance by utilizing 
these capabilities.

The requirement is to program the hardware NICs through existing standard
open source hardware offload APIs (e.g. DPDK rte_flow and/or Linux TC flower),
thereby enabling a single shared/generic vendor-agnostic offload implementation. 

# 3. Proposed solution

The proposed solution defines a generic offload module layer in the vRouter.

The generic offload layer logically resides between the vrouter datapath and
the smart NIC. It registers to receive notifications/callbacks on vrouter 
configuration changes from the agent (these are based on previous work in
contrail vrouter 3.1 branch, see https://github.com/Juniper/contrail-vrouter/
blob/R3.1/include/vr_offloads.h).

The generic offload layer is responsible for programming offload rules to the 
smart NIC through the existing standard open source hardware offload APIs
(e.g. DPDK rte_flow and/or Linux TC flower). The initial implementation will
add rules for classifiying tunnel packets received on the fabric interface 
matching established flows in the flow table, with an associated action to
mark/tag the packet with the flow id/index of the flow. 

Based on the programmed rules, packets received from the fabric interface are
checked to identify if they are marked/tagged. Packets identified with a 
mark/tag are known to have hit the flow rule matchign the specified index.
This information is then used to accelerate the vrouter data-path, bypassing
portions of the processing already handled in the NIC according to the
offload rules. The initial implementation proposed will bypass the majority
of the input classification process, including the input ip classification
the tunnel idenfifier classification (MPLS label or VXLAN VNI) and the flow 
table classification.

Any packet received without a mark/tag shall continue to be processed without
bypassing any of the existing vRouter implementation. 

Similarly, when offloads are not enabled, all packets will be considered
unmarked/untagged, and not subject to any offload/bypass, ensuring no 
functional impact/risk to the non-accelerated mode.    

The offload layer shall be designed to support both DPDK and kernel based
vrouters, allowing a shared logical implementation with an OS callback layer
abstracting teh OS specific functionality, in a manner similar to that 
currently done within vRouter. The initial version will only implement the
DPDK OS callback layer, deferring TC offload to a second step. 

It is expected that future releases would continue to enhance this initial
implementation to further accelerate/offload vRouter by utilizing additional
offload capbilites as those are made available through enhancements pushed to
the standard offload APIs.    


## 3.1 Alternatives considered
An alternative discussed in the community was to allow implementing the GOM 
ommunication to the hardware NICs through vendor-specific hooks, instead of
the proposed existing standard APIs (Linux TC flower and DPDK rte_flow).

The advantage of such an approach is that it would allow vendors to more 
easily implement offloads, bypassing the need to agree on the required
extensions required to the existing APIs and push thus changes into the
repsective communities (Linux/DPDK).

The disadvantage of such an approach is that it would inevitably lead to
multiple non-conformant vendor-specific offload implementations, instead of
a single shared/generic vendor-agnostic offload implementation.
 
## 3.2 API schema changes
No expected changes to APIs.

## 3.3 User workflow impact
The user will explicitly control enabling/disabling hardware offloads,
disabeld by default.
For DPDK vrouter - enabling the offloads will be done via the vrouter
command line (new enable_offloads flag).
For kernel vrouter - TBD when implemented in future releases.
A deployment option should also be added to automate this proccess
when deploying compute nodes (method may be deployment speciifc,
initial support will be added to contrail-ansible-deployer process.


## 3.4 UI changes
- No expected changes to UI

## 3.5 Notification impact
- Log if offload is enabled at startup
- Log any critical error while offloading rules

# 4. Implementation
## 4.1 Work items
  
  - vrouter: Implement interface for an offload module to receive notifications
    on vrouter configuration changes from the agent (based on previous work in 
    contrail vrouter 3.1 branch, see https://github.com/Juniper/
    contrail-vrouter/blob/R3.1/include/vr_offloads.h).
.
  - vrouter: Implement DPDK command line for enabling generic offload layer, 
    triggering registration of generic offload layer callbacks with above 
    interface to receive notifications. Implementation for kernel based vrouter
    layer may be done through kernel module config - deferred to future 
    releases implementing kernel offloads.  

 - vrouter: Implement geenric offload layer control plane logic (tracking vrouter 
   configurations through GOM callabck interfaces registeres and programming 
   hardware rules and actions).
   
 - vrouter: Implement detection of tagged packets on packets recieved from the
   fabric interface, and bypassing of input classification in vrouter datapath.   
 
 - vrouter: Update vrouter-ansible-deployer to allow enabling vrouter offload
   on a node.

 - vrouter: update vrouter to latest available stable DPDK version, targeting
   DPDK 18.05 or 18.08 (depending on release schedule). 

# 5. Performance and scaling impact
## 5.1 API and control plane
- None

## 5.2 Forwarding performance
- When offloading is disabled, there should be no performance impact.  
- When offloading is enabled forwarding performance is expected to
  increase for offloaded scenarios, and not have any singificant performance
  impact for scenarios not offloaded. 
   
# 6. Upgrade
- None

# 7. Deprecations
- None

# 8. Dependencies
- None

# 9. Testing
## 9.1 Unit tests
- None
## 9.2 Dev tests
- None
## 9.3 System tests
- Run all existing system tests in CI with offload disabled.
  - No functional impact
  - No performance impact 
- Run all existing system tests in CI with offload enabled.
  - No functional impact
  - Performance improvement or no performance impact 

# 10. Documentation Impact
- Document procedure for enabling smart NIC offloads.

# 11. References
- TF community paravirt offload design doc:
	Tungsten Fabric vRouter Smart NIC Offload for Para-virtualized VMs - Architecture and Design Proposal
    https://docs.google.com/document/d/1gxzC6xz-qnTv5r6Gt5KZgNHER6eFI29DZkpKt4RxhY4/edit#
