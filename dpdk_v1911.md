```
Upgrade DPDK library version to 19.11 LTS

Introduction:

Contrail vrouter currently uses DPDK version 18.05.1 which is a little less
than 2 years old. Lot of enhancements, bug fixes, support for new NIC firmware
etc. has gone in after that.

Problem statement:

Contrail vRouter should be updated to support the latest DPDK 19.LTS. The DPDK
19 series brings support for:

- new intel NIC’s and firmware
- enhancements around container networking DPDK
- many bug fixes
- virtio library fixes (required for upstreaming contrail vhost infrastructure)
- live migration fixes
- fixes in bonding/lacp
- bug fixes for mellanox NICs
- multiqueue enhancements (to support more queues than the forwarding cores)

Requirements:

1) Contrail code should work for both DPDK 18.05.1 and 19.11(LTS) for some time
2) Contrail code should give same/more performance
3) Contrail code should support live migration
4) Contrail code should move to use upstream vhost library

Changes needed:

DPDK library changes -

1) We need to port a few patches from DPDK 18.05.1 to 19.11 including LACP
   fast timer support
2) Need changes for proper way to handle rte_eth_dev_configure(). In DPDK
   18.05.1, on some config errors, the API was not returning, but instead
   proceeding to the next config change. But in the new version, it is
   returning back to the caller. This needs to be handled cleanly
3) Need a few changes in af_packet API

vrouter library changes -

1) Changes in API to get PCI address of bond
2) Some structure names like eth_header etc. have changed as rte_eth_header.
   Need to define a macro to handle this for 19.11 LTS
3) rx ethdev offloads have changed. Need to use proper macros.
4) kni implementation needs to be removed since its no longer used post R5.0
5) Some changes in SConscript needs to be done

UI changes -
None

Schema changes -
None

Feature testing -
TBD

Solution testing -
TBD

Upgrade impact -
None
```
