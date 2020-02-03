```
1. Background
-------------
Performance for DPDK vrouter is measured in terms of PPS per core. From the time DPDK vrouter was conceptualized about 4 years ago till now, the performance numbers we have measured in our lab is between 1.2 and 1.6 MPPS per core in a centos or ubuntu environment. Compare this with around 4mpps per core for OVS-DPDK (Provider Vlan mode), contrail DPDK vRouter is heavily challenged in terms of performances when compared to vertical stacks (e.g. Cisco, Ericsson, Nokia, Huawei …). In parallel, ovs-dpdk has a complex multi-core support (queue pinning is indeed complex - round robin scheme being the least worse option according to RH compared to manual pinning -), on which solutions based on dynamic queue reassignment based on observed load are underway.

2. What has changed?
--------------------
From last year, the focus has shifted from American telcos like ATT to EMEA/APAC telcos like OBS and Telstra. These telcos use RHEL and adopt RedHat recommended performance tuning.

However, our performance testing methodology has not changed to include RHEL. We still use CentOS platforms. RedHat uses their homegrown performance tuning tool called “Tuned” for performance optimization. The tool essentially does “CPU partitioning” to various processes to achieve good performance. One such configuration that tuned uses is called “isolcpus (isolated_cores)”.

3. Problem statement
--------------------
Performance of vrouter in default (no isolcpus) is say ‘X’ PPS. When the compute node is configured with tuned’s “isolcpus” option on nova (application VM) cores, customers and SRE team are seeing a vrouter performance degradation of around 30% (0.7 X) in the PPS measurement.

4. Performance numbers with R5.1
-------------------------------
MPPS per core (upto 0.01% loss)
    Without Isolcpus - 1.5
    Isolcpus on nova - 1

5. Why is the performance lower?
---------------------------------
The configuration of the bad-case was isolcpus on nova cores and tuned (isolated_cores) enabled. The PROX VM had one forwarding core.

There were drops seen in the forwarding cores. The drops indicated that the cores -
1) Were doing more processing than what was expected
2) Were being context-switched

After more analysis, the following issues were found which caused low performance -

1) The forwarding cores were not isolated. They were just pinned using ‘taskset’ which will not guarantee context switches
2) The service lcores were not isolated or pinned. They could hijack the CPUs of forwarding cores.
3) The PROX VM itself was allocated just one forwarding core. Ideal combination is it would need two cores - one to receive packets and another to send packets
4) There were system calls sometimes in forwarding cores
5) As per RedHat, when isolcpus are enabled, Linux uses a different scheduling algorithm which has a larger latency. This could explain why turning off isolcpus did not have lower performance and when we turned on isolcpus, the performance was lower - https://www.rapitasystems.com/blog/cooperative-and-preemptive-scheduling-algorithms (This needs verification)
6) There was a management interface (linux based) in the VNF which was generating packets. This would cause VMExits on forwarding cores which would lower the performance. But all VNFs would have this, so we have to live with it

Note: The service lcores include - netlink lcore, pkt0 lcore, tapdev lcore, uvhost lcore and timer lcore. There are another three DPDK control threads which we don’t have any control on - rte_mp_handle, rte_mp_async and eal-intr-thread

6. The Golden Recipe
--------------------
Based on the above observations and analysis, the system parameters were iteratively tuned with private changes in DPDK binary. The performance became better. The final recipe is as follows -

1) Isolate and Pin forwarding and nova lcores - This makes sure no context switches are there in these lcores
2) Isolate and Pin the service lcores and DPDK control threads - This makes sure other processes don’t hijack forwarding lcores
3) Disable yield on forwarding lcores - This avoid system calls being made which result in context switches
4) Allocate two forwarding lcores to PROX VM - This is the preferred way
5) Increase the ring size between forwarding lcores - This will provide more resilience to handle micro-bursts

7. Changes for full CPU partitioning (Tracked via CEM-10093)
------------------------------------------------------------
1) Use kernel isolcpus to isolate all cores except a few for the host
2) Use tuned isolated_cores to isolate provide non-isolation cores for host
3) Have a configurable parameter in vrouter to provide coremask for service lcores
4) Have a configurable parameter in vrouter to provide coremask for DPDK control threads (This involves changing the DPDK library)
5) Have a configurable parameter in vrouter to provide the forwarding lcores ring size. (Default is 1024)
6) Have a configurable parameter in vrouter to disable yield
7) Update the juniper recommendation guide to reflect the above in case of full CPU partitioning

8. Results
----------
Test   Description                             PPS with drops     PPS without
                                               <0.01% per core    drops per core
----   -----------                             ----------------   --------------
1      Regular vRouter binary from 1910 with    0.5 to 1 Mpps     0.5 to 1 Mpps
       isolcpus enabled on nova only            (variable)        (variable)
2      Modified (private) 1908 vRouter binary   1.85Mpps          1.84Mpps
       with above recommendations applied       (stable)          (stable)

9. New CLI parameters to DPDK binary
------------------------------------

1) To address 7.3 and 7.4 (coremask for service lcores)
    --service_core_mask <hex coremask or string coremask>
    The acceptable values are hex number. Eg: 0x0060 or string: (5,6)
    This value is applied to both service lcores as well as DPDK control threads
2) To address 7.5 (forwarding lcores ring size)
    --vr_dpdk_rx_ring_sz <size>
    --vr_dpdk_tx_ring_sz <size>
    Default is 1024
3) To address 7.6 (disable yield)
    --yield_option <1/0>

10. Provisioning changes
------------------------
Provisioning changes are needed for TripleO, kolla and Juju.
A new variable SERVICE_CORE_MASK is provided in common_vrouter.env which is parsed by entrypoint.sh to populate the --service_core_mask CLI

11. DPDK (library) changes
--------------------------
There is no CLI option in DPDK(18.05) library to provide core mask to DPDK control threads, so a global array must be used to set values in vrouter and this array is used to assign CPUs to control threads. The changes are done in two steps:
1) Dependent patch https://github.com/DPDK/dpdk/commit/c3568ea376700df061abcbeabc40ddaed7841e1a is backported
2) Changes for this story are patched on this
3) We will upstream the changes to DPDK library

```
