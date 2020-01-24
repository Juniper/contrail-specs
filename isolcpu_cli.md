# 1. Introduction

# 2. Problem statement
Jira story: https://contrail-jws.atlassian.net/browse/CEM-10093

It has been observed that there is a vrouter performance degradation of around
30% whenever compute node is configured with isolCPUs option on nova cores. After
analysing the situation in which performance was lower and iteratively tuning
different system parameters along with private changes in DPDK binary, performance
was observed to get better.

This document captures the design and changes required to introduce configurable
parameters in vrouter for the following:
1) To provide core mask for service lcores
2) To provide core mask for DPDK control threads
3) To configure forwarding lcores ring size
4) To provide an option to disable yield

# 3. Proposed solution
The solution is to provide command line arguments in dpdk command line interface to
configure above parameters and to make changes to assign CPUs to cores based on
core mask in service lcores and DPDK control threads.

# 4. Alternatives considered
None

# 5. API schema changes
None

# 6. UI changes
None

# 7. Notification Impact
None

# 8. Provisioning changes
This may require some provisioning changes to accommodate the new CLI commands and parameters.

# 9. Implementation
### 9.1 Agent
None

### 9.2 Vrouter
The following changes would be required in Vrouter:
1) Instead of using VR_DPDK_RX_RING_SZ and VR_DPDK_TX_RING_SZ, provide new parameters
with values defaulting to the above macros, and provide options in dpdk CLI to
configure them.

2) Yield is enabled by default for now, so CLI option must be provided to disable it.

3) As of now CPUs are assigned to service cores and dpdk cores automatically by running
through all available CPUs. An option must be provided to assign CPUs based on a core
mask through command line.

### 9.3 DPDK
1) There is no CLI option in DPDK(18.05) binary to provide core mask to DPDK control
threads, so a global array must be used to set values in vrouter and this array could be
used to assign CPUs to control threads.

# 10. Performance and scaling impact

# 11. Upgrade

# 12. Deprecations
None

# 13. Dependencies

# 14. Testing
To be added
