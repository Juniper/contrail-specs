# 1. Introduction
SmartNIC hardware offload infrastructure is present in branches R3.1 and R4.1
of vRouter, and adoption into master is imminent (
https://review.opencontrail.org/45600 ). This spec proposes enabling dynamic
registration of offloads instead of the current static implementation proposed.

# 2. Problem statement
Blueprint:
https://blueprints.launchpad.net/opencontrail/+spec/pluggable-offloads

Without a registration interface, hardware offloads are limited to static,
built-in implementations. Currently, the only proposed user of the offload
interface is the RTEflow implementation proposed in
https://review.opencontrail.org/45601 ; no in-tree Linux kernel implementation
exists at this point in time. The offload interface registration in
https://review.opencontrail.org/45600 occurs at module init for the kernel
vrouter and application init for the userspace DPDK vrouter. This is not
sufficient for dynamic configuration.

# 3. Proposed solution
The solution proposed in this spec builds on the design implemented in
https://review.opencontrail.org/45600 and assumes that it has been merged.

This spec proposes implementing a dynamic registration feature for the offload
hooks. Instead of having a set of hard-coded offload options selectable at
initialization, the proposal is to allow offloads to be registered at runtime.

On the kernel side, dynamic plugging and unplugging of offload modules requires
no extra code. For DPDK, using dynamic libraries would likely require
implementing or extending a dynamic loading framework. DPDK already has EAL
facilities to load PMDs (poll-mode-drivers) as dynamic modules, but this does
not extend to an arbitrary plug-in model.

Since this model has a supported implementation with full vrouter datapath
offload (in contrast to partial offloads currently proposed), this would
additionally allow the interfaces in upstream OpenStack to be eligible for
submission.

# 4. Alternatives considered
* Static offloads (as implemented in https://review.opencontrail.org/45600)
  means that there is no possibility for extending the vrouter without a full
  QA cycle, essentially making releases monolithic.
* A vendor might support this interface downstream, leading to fragmentation.
* Developing the offloads in-tree would be possible, but in-tree full offload
  solutions are not mature enough yet.

# 5. API schema changes
None.

# 6. UI changes
None.

# 7. Notification impact
None.

# 8. Provisioning changes
The provisioner would be responsible for deploying and loading the required
kernel modules on the compute nodes with hardware support. Specific
implementation of an offload module is beyond the scope of this spec.

# 9. Implementation

1. Rebase https://review.opencontrail.org/47035 onto
   https://review.opencontrail.org/45600.
2. Extend the review to cover test cases.
3. A related review ( https://review.opencontrail.org/42850 ) is required to
   support full offloads.

# 10. Performance and scaling impact
Minimal. RCU protection will be added around offload callbacks. The offload
callbacks are called during datapath modification events, not per packet.

# 11. Upgrade
None.

# 12. Deprecations
None.

# 13. Dependencies
https://review.opencontrail.org/45600

# 14. Testing
The plugin interface will be tested with unit and functional tests, being
consistent as far as possible with https://review.opencontrail.org/45600.
Third-party plugins will require a third-party CI to report to Tungsten Fabric
in a non-voting capacity only. No testing load or extra tests will be required
or supported by the Tungsten Fabric community.

# 15. Documentation Impact
This is expected to be an unstable interface, where the Tungsten Fabric project
may modify it unilaterally. As such, documentation will be made available on a
per-offload implementation, as they are merged into the Tungsten Fabric core.

# 16. References
1. Static offload hooks: https://review.opencontrail.org/45600
2. Static offload implementation for DPDK:
   https://review.opencontrail.org/45601
3. WIP review for dynamic registration: https://review.opencontrail.org/47035
