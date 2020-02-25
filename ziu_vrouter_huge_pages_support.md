
# 1. Introduction

One of the major challenges that Contrail customers face is the difficulty
and risks related to upgrading a deployed Cluster to a more recent release.
In order to enable customers to upgrade at a frequent pace, it is absolutely
required to make upgrade operation as seamless as possible.
Today, customers use In service software upgrade (ISSU) to upgrade their
clusters. This involves vacating the workloads from the compute being
upgraded to other computes, upgrading the compute and then moving
the workloads back to the upgraded compute. This is a tedious and time
consuming process which sometimes takes weeks/months to upgrade the
whole cluster (depending on cluster size).
It also requires additional compute resources to temporarily host the workloads.

To avoid the above problems, as part of Zero Impact Upgrade (ZIU) feature
an In place upgrade (IPU) procedure will be introduced. The objective of ZIU
features is to allow customers to upgrade from one release to a more recent
release while minimizing cost, time, risk, complexity, and unavailability impacts.
In case of IPU, the workloads will not be migrated and all the contrail compute
related components like vRouter Agent, vRouter datapath will be upgraded to
the newer version of software in place. This results in faster upgrades without
requiring any additional compute resources. There would be some amount of
downtime for user traffic which is unavoidable with IPU.

There are multiple stories associated with ZIU (please refer to
EPIC [CEM-4739](https://contrail-jws.atlassian.net/browse/CEM-4739) for more details).
This document focuses on adding huge page support for vRouter running
in kernel mode - [CEM-11529](https://contrail-jws.atlassian.net/browse/CEM-11529).

# 2. Problem statement

As part of IPU, one of the requirements is to be able to upgrade vRouter agent
 and datapath **without** requiring compute reboot. However, vRouter datapath
 does large memory allocations for bridge and flow tables requiring compute
 reboot to avoid memory fragmentation issues. This compute reboot is mainly
 required for vRouter running in kernel mode due to lack of huge page support.
 Without huge page support, it is possible that the required number of
 contiguous pages may not be available to satisfy large memory allocations.

# 3. Proposed solution

Add huge page support to vRouter running in kernel mode.
DPDK vRouter already uses huge pages. Huge pages are reserved
in the linux kernel at the time of provisioning and once the reservation
goes through successfully, the subsequent allocation of pages is
guaranteed to succeed.
Huge pages of size 2MB and 1GB will be supported.
The flow and bridge tables will be allocated using huge pages.
The basic vRouter changes to support huge pages in kernel mode was already
done as part of
[https://github.com/Juniper/contrail-specs/blob/master/vrouter_huge_memory.md](https://github.com/Juniper/contrail-specs/blob/master/vrouter_huge_memory.md). 
However, it was never integrated/qualified and it only supports 1 GB page size.
As part of this story, changes required in agent and vRouter to enable and use
huge pages in 2MB and 1GB form will be done.
Provisioning changes will also be done to activate the huge page support.

## 3.1 Alternatives considered

None

## 3.2 API schema changes

None

## 3.3 User workflow impact

There will be deployment specific configuration items to enable huge page
support in vRouter kernel mode. This will be similar to DPDK case where the
number of huge pages required is configured.
For eg: In case of ansible deployer, instance.yaml contains a parameter to
configure number of huge pages required.
[Provisioning team to add more details]

## 3.4 UI changes

None

## 3.5 Notification impact

None

# 4. Implementation

## 4.1 vRouter Agent

As part of provisioning, the agent configuration file is populated with the
names of the mounted hugetlbfs files for flow and bridge table viz.
/dev/hugepages/bridge and /dev/hugepages/flow. These files are used by
agent to memory map the tables into its address space.
They also indicate that huge pages is enabled for vRouter running in kernel mode.
The huge page size to be used (2MB or 1GB) is derived from the configuration,
since both sizes are configured separately.

contrail-vrouter-agent.conf

    [RESTART]

    #Huge pages, mounted at the files specified below, to be used by vrouter
    #running in kernel mode for flow table and brige table.
     huge_page_1G=<1G_huge_page_1> <1G_huge_page_2>
     huge_page_2M=<2M_huge_page_1> <2M_huge_page_2>

Once Agent has read this config, as part of init, it needs to calculate the flow
and bridge table size (by reading the values from vRouter) and then mmap the
flow and bridge table memory into its process address space.
It then programs the vRouter datapath of the virtual memory address of the
pages and huge page size information.
On success response from vRouter, further agent processing continues.
In case of failure from vRouter, an error message is logged in agent log file.

## 4.2 vRouter kernel module

The kernel module receives huge page config from agent containing the starting
virtual memory address of pages and the huge page memory size. Based on this
information, the kernel module pins these pages in memory. It further initializes
and allocates the flow and bridge tables using the same huge pages.
It returns success or failure code to agent.

## 4.3 Provisioning

The following changes are required (for all deployment types)-

1. set /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages for 2MB hugepages
    or set /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages for 1GB
    hugepages based on user provided input.
    This will result in kernel reserving the configured number of huge pages.

2. mount hugetblfs in /dev/hugepages inside the vRouter agent container

3. create the /dev/hugepages/flow and /dev/hugepages/bridge files

4. Populated huge_page_1G=<1G_huge_page_1> <1G_huge_page_2>
     and/or huge_page_2M=<2M_huge_page_1> <2M_huge_page_2> in
     contrail-vrouter-agent.conf file via entrypoint.sh

5. On vrouter/agent error, compute should be rebooted after informing user.

# 5. Performance and scaling impact

No impact.

## 5.1 API and control plane

No impact.

## 5.2 Forwarding performance

No impact.

# 6. Upgrade

When upgrading from old version which doesn't support huge pages in vRouter
kernel mode to a new version supporting it, a reboot may be required to make
huge page reservation successful.
Also, in cases where huge page reservation or allocation fails, a reboot may be
required.
Both these cases needs to be handled by the deployer at the time of provisioning.

# 7. Deprecations

None.

# 8. Dependencies

There is a dependency on provisioning changes to test this feature as a whole.
The actual provisioning changes are tracked using [CEM-12353](https://contrail-jws.atlassian.net/browse/CEM-12353)

# 9. Testing

## 9.1 Unit test
1. Verify if agent reads the agent config file correctly and mmaps the hugepage
     files correctly.
2. Verify if kernel module receives huge page information from agent and
    allocates flow and bridge memory using huge pages.
3. Verify all the permutation and combination of hupepage enabled/disabled
    and kernel compute/dpdk compute.
5. Repeat 1-3 for 2MB and 1GB huge page sizes.
Note: Most/All of the above test cases would be executed manually due to
lack of infra/HW dependency.

## 9.2 Feature test

1. Check if kernel mode vRouter works fine with hugepage setting of 2MB and 1GB
2. Check if dpdk mode vRouter works fine with hugepage setting of 2MB and 1GB
3. Check if kernel mode vRouter works fine without hugepage setting
4. Check if dpdk mode vRouter works fine without hugepage setting
5. Restart vRouter in kernel mode with hugepage setting multiple times (eg 10 times)
6. Restart vRouter in dpdk mode with hugepage setting multiple times (eg 10 times)
7. Restart vRouter agent alone and verify behaviour with kernel mode vRouter

## 9.3 Integration test (vRouter + Provisioning code integrated)

1. Test with Openstack ansible deployer
2. Test with RHOSP deployment

## 9.4 Solution test

1. Test RHOSP deployment
2. Test JuJu Openstack/Ubuntu deployment
[Test team to add more]

# 10. Documentation Impact

The new deployer parameters to configure/enable vRouter huge pages in kernel
mode needs to be documented.

# 11. References

1. ZIU PRD: https://docs.google.com/document/d/10cqRxmpZowrmfaP7Tzxac4mMqzH7_fHJDDhFqqWarG0
2. ZIU EPIC: https://contrail-jws.atlassian.net/browse/CEM-4739
3. Huge page support story: https://contrail-jws.atlassian.net/browse/CEM-11529
4. vRouter huge page changes: https://github.com/Juniper/contrail-specs/blob/master/vrouter_huge_memory.md
5. Provisioning: https://contrail-jws.atlassian.net/browse/CEM-12353

