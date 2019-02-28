
# 1. Introduction

Contrail vrouter with dpdk supports different NICs. Support for Mellanox
NIC requires additional build changes

# 2. Problem statement

To build Contrail vrouter dpdk application with Mellanox NIC support
dpdk library to be built with Mellanox configurations enabled. In addition,
the mellanox libraries need to be installed in the build system and as
well as the dpdk container. The kernel modules for mellanox needs to be
installed in the Host as well.

# 3. Proposed solution

For the dpdk build and install, the spec file for the vrouter-dpdk
will be modified to include Mellanox libraries.
For the host installation, Containers for Redhat and Ubuntu will be created
that will be launched during start up, which would install the necessary
kernel packages required.

## 3.1 Alternatives considered

## 3.2 API schema changes

## 3.3 User workflow impact

## 3.4 UI changes

## 3.5 Notification impact

# 4. Implementation
## 4.1 Build changes
The contrail-vrouter-dpdk.spec file will be modified to include the mellanox
libraries as dependencies. Both the buildRequires and Requires field needs to
be added so that the depenency packages will be installed in the build system
as well as in the vrouter-dpdk container.

## 4.2 Init Container changes
Two containers plugin/mellanox/init/ubuntu and plugin/mellanox/init/centos
would be created. Both the containers will run during startup and should mount
/lib/modules and /usr/src. The Centos container will install the kernel modules
and the Ubuntu container will compile and install the kernel module with the
dkms package.

## 4.3 Ansible/Helm changes
Ansible/Helm changes are required to start the init container to install
the mellanox packages in the host.

## 4.4 Configuration changes
To enable mellanox nic, the uio-pci-generic or the vfio-pci driver cannot be
used. The dpdk nic bind will not be done with any driver. So the
configuration option of DPDK_UIO_DRIVER under the vrouter configuration
should be set to empty value. Changes would be required in the
agent-functions.sh and the common-functions.sh to support this.

# 5. Performance and scaling impact

# 6. Upgrade

# 7. Deprecations

# 8. Dependencies

# 9. Testing

# 10. Documentation Impact
Documentation changes required for the DPDK_UIO_DRIVER configuration.
For Mellanox nics, the configuration should be set with an empty '' value.

# 11. References
