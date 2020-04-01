```
1. Background
-------------
Performance for DPDK vrouter is measured in terms of PPS per core. From the
time DPDK vrouter was conceptualized about 4 years ago till now, the
performance numbers we have measured in our lab is between 1.2 and 1.6 MPPS
per core in a centos or ubuntu environment. Compare this with around 4mpps per
core for OVS-DPDK (Provider Vlan mode), contrail DPDK vRouter is heavily
challenged in terms of performances when compared to vertical stacks
(e.g. Cisco, Ericsson, Nokia, Huawei …). In parallel, ovs-dpdk has a complex
multi-core support (queue pinning is indeed complex - round robin scheme being
the least worse option according to RH compared to manual pinning -), on which
solutions based on dynamic queue reassignment based on observed load are
underway.

2. What has changed?
--------------------
From last year, the focus has shifted from American telcos like ATT to
EMEA/APAC telcos like OBS and Telstra. These telcos use RHEL and adopt RedHat
recommended performance tuning.

However, our performance testing methodology has not changed to include RHEL.
We still use CentOS platforms. RedHat uses their homegrown performance tuning
tool called “Tuned” for performance optimization. The tool essentially does
“CPU partitioning” to various processes to achieve good performance.
One such CPU isolation mechanism that tuned uses is called “isolated_cores.
Linux Kernel is also providing another isolation method via isolcpus parameter.

3. Problem statement
--------------------
Performance of vrouter in default (no CPU isolation mechanism) is say ‘X’ PPS.
When the compute node is configured with some isolation mechanism option on
nova (application VM) cores, customers and SRE team are seeing a vrouter
performance degradation of around 30% (0.7 X) in the PPS measurement.

4. Performance numbers with R5.1
-------------------------------
MPPS per core (upto 0.01% loss)
    Without Isolcpus - 1.5
    Isolcpus on nova - 1

5. Why is the performance lower?
---------------------------------
The configuration of the bad-case was isolcpus on nova cores and tuned
(isolated_cores) enabled. The Virtual Machine (PROX VM) itself was allocated
just one forwarding core.

There were drops seen in the forwarding cores. The drops indicated that
the cores -
1) Were doing more processing than what was expected
2) Were being context-switched

After more analysis, the following issues were found which caused low
performance -

1) The forwarding cores were not isolated. They were just pinned using
‘taskset’ which will not guarantee that there will be no context switches.
2) The service lcores were not isolated or pinned. They could hijack the CPUs
of forwarding cores.
3) Allocate enough forwarding cores on DPDK VM (here for instance we’ve got
better result with a Virtual Machine – PROX VM – configured with two forwarding
lcores) - one to receive packets and another to send packets
4) There were system calls sometimes in forwarding cores
5) As per RedHat, when isolcpus are enabled, Linux uses a different scheduling
algorithm which has a larger latency. This could explain why turning off
isolcpus did not have lower performance and when we turned on isolcpus, the
performance was lower -
https://www.rapitasystems.com/blog/cooperative-and-preemptive-scheduling-algorithms
(This needs verification)
6) There was a management interface (linux based) in the VNF which was
generating packets. This would cause VMExits on forwarding cores which would
lower the performance. But all VNFs would have this, so we have to live with it

Note: The service lcores include - netlink lcore, pkt0 lcore, tapdev lcore,
uvhost lcore and timer lcore. There are another three DPDK control threads
which we don’t have any control on - rte_mp_handle, rte_mp_async and
eal-intr-thread

6. The Golden Recipe
--------------------
Based on the above observations and analysis, the system parameters were
iteratively tuned with private changes in DPDK binary. The performance became
better. The final recipe is as follows -

1) Isolate and Pin forwarding and nova lcores - This makes sure no context
switches are there in these lcores
2) Isolate and Pin the service lcores and DPDK control threads - This makes
sure other processes don’t hijack forwarding lcores
3) Disable yield on forwarding lcores - This avoid system calls being made
which result in context switches
4) Allocate two forwarding lcores to PROX VM - This is the preferred way
5) Increase the ring size between forwarding lcores - This will provide more
resilience to handle micro-bursts

7. Changes for full CPU partitioning (Tracked via CEM-10093)
------------------------------------------------------------
1) Use kernel isolcpus and tuned isolated_cores to isolate all cores except a
few for the host
2) Use tuned CPUAffinity to provide non-isolation cores for host
3) Have a configurable parameter in vrouter to provide coremask for service
lcores
4) Have a configurable parameter in vrouter to provide coremask for DPDK
control threads (This involves changing the DPDK library)
5) Have a configurable parameter in vrouter to provide the forwarding lcores
ring size. (Default is 1024)
6) Have a configurable parameter in vrouter to disable yield
7) Update the juniper recommendation guide to reflect the above in case of
full CPU partitioning

8. Results
----------
Test   Description                             PPS with drops    PPS without
                                               <0.01% per core   drops per core
----   -----------                             ---------------   --------------
1      Regular vRouter binary from 1910 with    0.5 to 1 Mpps     0.5 to 1 Mpps
       isolcpus enabled on nova only            (variable)        (variable)
2      Modified (private) 1908 vRouter binary   1.85Mpps          1.84Mpps
       with above recommendations applied       (stable)          (stable)

9. New CLI parameters to DPDK binary
------------------------------------

1) To address 7.3 (coremask for service lcores)
    --service_core_mask <hex coremask or string coremask>
    The acceptable values are hex number. Eg: 0x0060 or string: (5,6)
2) To address 7.4 (coremask for DPDK control threads)
    --dpdk_ctrl_thread_mask <hex coremask or string coremask>
    The acceptable values are hex number. Eg: 0x0060 or string: (5,6)
3) To address 7.5 (forwarding lcores ring size)
    --vr_dpdk_rx_ring_sz <size>
    --vr_dpdk_tx_ring_sz <size>
    Default is 1024
4) To address 7.6 (disable yield)
    --yield_option <1/0>

10. Provisioning changes
------------------------
1) Provisioning changes are needed for TripleO (and optionally kolla and Juju)
2) A new variable SERVICE_CORE_MASK is provided in common_vrouter.env which is
parsed by entrypoint.sh to populate the --service_core_mask CLI
3) A new variable DPDK_CTRL_THREAD_MASK is provided in common_vrouter.env which
is parsed by entrypoint.sh to populate the --dpdk_ctrl_thread_mask CLI

Note: For parameters in 9.3 and 9.4 above, we will not introduce any new
variables in provisioning. Instead, the customer needs to add these manually
as part of DPDK_COMMAND_ADDITIONAL_ARGS in entrypoint.sh file in the DPDK
container.

11. DPDK (library) changes
--------------------------
There is no CLI option in DPDK(18.05) library to provide core mask to DPDK
control threads, so we need changes in the DPDK library for this. We introduce
a new dpdk argument '-t'to assign CPUs to control threads.
The changes are done in two steps:

1) Dependent patch
https://github.com/DPDK/dpdk/commit/c3568ea376700df061abcbeabc40ddaed7841e1a
is backported
2) Changes for this story are patched on top of this
3) We will upstream the changes to DPDK library

12. Hardware Specifications
---------------------------
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                40
On-line CPU(s) list:   0-39
Thread(s) per core:    2
Core(s) per socket:    10
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 85
Model name:            Intel(R) Xeon(R) Silver 4210 CPU @ 2.20GHz
Stepping:              7
CPU MHz:               2201.000
CPU max MHz:           2201.0000
CPU min MHz:           1000.0000
BogoMIPS:              4400.00
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              1024K
L3 cache:              14080K
NUMA node0 CPU(s):     0-9,20-29
NUMA node1 CPU(s):     10-19,30-39

13. Template changes while Deploying
------------------------------------
The following parameters must be configured under ContrailDpdkParameters in
contrail-services.yaml file to specify the division of cores as per requirement
KernelArgs: “default_hugepagesz=1GB hugepagesz=1G hugepages=32 iommu=pt 
             intel_iommu=on isolcpus=1-9,11-19,21-29,31-39”
NovaVcpuPinSet: ‘6-9,11-19,25-29,31-39’
TunedProfileName: “cpu-partitioning”
IsolCpusList: “1-9,11-19,21-29,31-39”

14. Verification of deployment with IsolCPUs enabled
----------------------------------------------------
1) Check for /etc/tuned/cpu-partitioning-variables.conf file for system
   parameters
   # Examples:
   # isolated_cores=2,4-7
   # isolated_cores=2-23
   #
   # To disable the kernel load balancing in certain isolated CPUs:
   # no_balance_cores=5-10
2) Check for '/etc/systemd/system.conf' file for CPU Affinity
   CPUAffinity=0 10 20 30
3) Check for '/etc/sysconfig/network-scripts/ifcfg-vhost0' file for Isolated
   CPU list
   CPU_LIST=1,2,3,4,21,22,23,24
4) Check '/proc/cmdline' for IsolCPU output
   BOOT_IMAGE=/boot/vmlinuz-3.10.0-1062.9.1.el7.x86_64
   root=UUID=228c59ea-82f0-4ee8-9d03-5620e5f0fafb ro console=tty0
   console=ttyS0,115200n8 crashkernel=auto rhgb quiet iommu=pt intel_iommu=on
   isolcpus=1-9,11-19,21-29,31-39 default_hugepagesz=1GB hugepagesz=1G
   hugepages=128 hugepagesz=2M hugepages=2048 skew_tick=1 nohz=on
   nohz_full=1-9,11-19,21-29,31-39 rcu_nocbs=1-9,11-19,21-29,31-39
   tuned.non_isolcpus=40100401 intel_pstate=disable nosoftlockup

15. Performance Tests Results (MPPS per physical core)
------------------------------------------------------
----------------------------------------------------------
|           case           |     ixia      |     prox    |
----------------------------------------------------------
|   No CPU Partitioning    |     2.256     |     2.505   |
----------------------------------------------------------
| Partial CPU Partitioning |     0.165     |     0.0525  |
----------------------------------------------------------
|  Full CPU Partitioning   |     2.294     |     2.534   |
----------------------------------------------------------

16. Support deployments
-----------------------
RHEL (TripleO)

17. UI impact
-------------
None

18. Feature Testcases
---------------------
TBD
```
