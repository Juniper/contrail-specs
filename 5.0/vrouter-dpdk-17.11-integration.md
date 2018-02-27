
#1. Introduction
Integrate OpenContrail vRouter with DPDK 17.11

#2. Problem statement
There have been a link status update delays in the DPDK bonding driver leading to link flapping and 
traffic loss, which has been fixed in DPDK 17.02. Also there has been some improvement in the bonding
driver in DPDK 17.11 that allows programing of the NIC through the rte_flow DPDK API to distinguish
data plane and control messages providing better performance by placing them in different queues. There
is also some work related to hashing of Tx datapath of the bonding driver that is expected to be pushed
into DPDK 17.11 to allow even better performance. Therefore we are proposing integrating OpenContrail 5.0
with DPDK 17.11.

#3. Proposed solution
Integrate with DPDK 17.11 and make the few necessary code updates in the vRouter

#4. Implementation
##4.1 Update the build system to pull DPDK 17.11
Changes need to be made in the vRouter build system in order to build with DPDK 17.11
##4.2 Update the vRouter to use the righ rte_flow API
Changes need to be made in the vRouter in order for it to setup the right flows and queues at init time
and allow better performance of the Data Plane.
##4.3 Any other updates due to possible DPDK API changes
Historically the changes of the DPDK API have been minor and moving from DPDK 17.02 to 17.11 we expect
single digit number API changes. From those APIs that changed the vRouter is expected to use a subset.
This subset will need to be refactored in the vRouter source.

#5. Performance and scaling impact
##5.1 API and control plane
No Impact

##5.2 Forwarding performance
With the above changes forwarding perfromance gain and stability should be achieved.

#6. Upgrade
No Impact

#7. Deprecations
None

#8. Dependencies
A new dependency to DPDK 17.11 will take place, replacing the existing dependencies to DPDK 17.02

#9. Testing
##9.1 Unit tests
None
##9.2 Dev tests
During Dev phase the developer should test the creation of the new queue for control messages and run
some basic benchmarks to confirm performance improvement.
##9.3 System tests
At system level a full setup with bonding driver should be put in place and verify the stability and 
performance gain of the system.

#10. Documentation Impact
Any documentation refering to DPDK's version should now refer to 17.11.

#11. References
<link to the flapping issue fix in bonding driver>
<link to the separation of the queues in the DPDK>
