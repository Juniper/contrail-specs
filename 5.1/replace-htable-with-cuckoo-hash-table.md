
#1. Introduction
Replace htable with Cuckoo hash table

#2. Problem statement
Contrail uses htable for many tables, such as flow table and bridge table, but it isn't a very
efficient hash implementation, we found it is a major performance bottle neck because it is called
very frequently.

#3. Proposed solution
We propose to use Cuckoo hash table to replace htable, Cuckoo hash table (membership library in DPDK
release) is a very highly-efficient hash table implementation, it leveraged AVX instructions to
optimize performance, our experiment has proved it can boost performance. Once htable
is re-implemented by Cuckoo hash table, flow table and bridge table lookup can get huge benefits
from it.

#4. Implementation
##4.1 Use Cuckoo hash table to replace htable
Change is very straight, we try our best to keep the old htable APIs so that changes are as few as
possible, contrail-vrouter, contrail-vrouter-dpdk and contrail-vrouter-agent all use it, so we
need to do changes very carefully, especially contrail-vrouter-agent shares flow tables with
contrail-vrouter/contrail-vrouter-dpdk by mmap, this is a big challenge we need to fix.

This implementation will work for both contrail-vrouter(kernel module) and contrail-vrouter-dpdk
because contrail-vrouter-agent is common for them and used the same table by mmap.

#5. Performance and scaling impact
##5.1 API and control plane
No Impact

##5.2 Forwarding performance
We expect this solution can boost forwarding performance

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
