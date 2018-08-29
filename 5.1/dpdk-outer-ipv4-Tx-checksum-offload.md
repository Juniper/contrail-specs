# 1. Introduction
This document describes a new DPDK feature to do Tx IPv4 HW checksum offload for
the outer part of tunneled packets, aka MPLSoGRE, MPLSoUDP and VXLAN (in addition to inner).

# 2. Problem statement
Currently in the Vrouter, there are 2 modes for Tx checksum HW offload:
1. No HW Tx checksum offloads - all the checksums (outer ipv4, inner ipv4 and inner L4)
are calculated by the Vrouter SW.
2. Partial HW Tx checksum offload(VIF_FLAG_TX_CSUM_FULL_OFFLOAD) - The HW can
offload Tx checksum only for one part of the packet, either inner or outer.

This feature suggests to add mode to do full HW Tx checksum offload - all the
inner and outer Tx checksums can be offloaded by the HW.

# 3. Proposed solution
Add new VIF flag to indicates the capability for the feature (VIF_FLAG_TX_CSUM_FULL_OFFLOAD) and to
skip the SW calculation when it is needed.

## 3.1 Alternatives considered
None.

## 3.2 API schema changes
None.

## 3.3 User workflow impact
No user involvement.

## 3.4 UI changes
No UI changes.

## 3.5 Notification impact
No notifications.


# 4. Implementation
Affects only vRouter.
Implementation for new offload will only be enabled/used for DPDK vrouter.
Add new vif flag (VIF_FLAG_TX_CSUM_FULL_OFFLOAD) to indicates the new mode
for VIFs backed with PMD with capabilities for DEV_TX_OFFLOAD_IPV4_CKSUM, DEV_TX_OFFLOAD_UDP_CKSUM,
DEV_TX_OFFLOAD_TCP_CKSUM and PKT_TX_OUTER_IP_CKSUM.
When transmitting packets to enabled VIFs, arrange the dpdk Tx mbuf according to the dpdk API to enable the new HW offload.

# 5. Performance and scaling impact
## 5.1 API and control plane
No performance impact for control plane.

## 5.2 Forwarding performance
Should improve performance for VIFs capable of full offloads.
No performance impact for other VIFs.

# 6. Upgrade
None.

# 7. Deprecations
None.

# 8. Dependencies
None.

# 9. Testing
## 9.1 Unit tests
None.

## 9.2 Dev tests
None.

## 9.3 System tests
Using existing system CI tests
Verify no functional or performance impact for existing NICs with partial offloads.
Verify no functional impact and performance improvement with supported devices.

# 10. Documentation Impact
None.
# 11. References
None.


