
#1. Introduction
Optimize contrail-vrouter-dpdk performance by batch processing and caching

#2. Problem statement
We use vtune and perf to find some hotspots, vr_htable_find_hentry is the biggest hotspot, htable
implementation is not so efficient, we also find vrouter doesn't use batch processing, these are main
factors impacting on contrail-vrouter-dpdk performance.

#3. Proposed solution
We propose to use caching and batch processing to boost performance, we have implemented a prototype
for this, that can boost performance by about 30%.

#4. Implementation
##4.1 Use hash table to cache flow table lookup result
Use a big array to cache flow table lookup results, flow key hash value is used as hash index, flow
entry pointer is saved in this array to check if there is a hash collision, the old one will be evicted
if yes, the new one is saved, otherwise the flow is allowed to forward, so remove expensive flow table
lookup overhead, our prototype shows it can boost performance by about 20%.

##4.2 Use hash table to cache bridge table lookup result
Use a big array to cache bridge table lookup results, destination mac hash value is used as hash index,
bridge entry pointer is saved in this array, which has next hop information. If the flow hits this cache,
we can remove expensive bridge table overhead, our prototype shows it can boost performance by about 5%.

##4.3 Do batch processing for vrouter Rx path
Many projects have proved batch processing is the best way to boost performance because it can make full
use of cache locality and prefetch, our simple prototype has confirmed this can boost performance by about
10%.

#5. Performance and scaling impact
##5.1 API and control plane
No Impact

##5.2 Forwarding performance
We expect this solution can boost forwarding perfromance by about 30%

#6. Upgrade
No Impact

#7. Deprecations
None

#8. Dependencies
None

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
None

#11. References
None
