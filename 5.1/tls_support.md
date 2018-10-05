# 1. Introduction

There is currently no way to use TLS when deploying Contrail 5.x using 
contrail-ansible-deployer. This document describes the feature to enable the 
ability to configure TLS for Openstack and Contrail services using the
contrail-ansible-deployer provisioning scripts.

# 2. Problem statement

In previous versions of Contrail (<= 4.x), the following services supported 
TLS 1.2:

		VNC API (using haproxy)
		XMPP
		Introspect
		Sandesh
		WebUI
		Keystone
	
While moving the above packages to support the newer version of TLS (1.3), it 
is also required to add TLS support to the following external services in 
Contrail 5.1:
	
		Cassandra
		Zookeeper
		Kafka
		RabbitMQ
	
# 3. Proposed solution

## 3.1 VNC API
The VNC API in previous versions of Contrail did not support TLS natively. The
VNC API traffic was encrypted by having the client talk to the backend through 
a haproxy server and enabling SSL encryptions for the conversation between the 
client and the haproxy server. This was done in order to avoid the performance 
degradation that would be seen if encryption was enabled natively. A 
benchmarking exercise needs to be done to study the performance when TLS is 
enabled natively.

The other option is to use the haproxy server that comes as part of Openstack, 
but again, this may not be possible for non-openstack configurations. So it 
is proposed to add the native TLS support to VNC API as mentioned above.



## 3.2 XMPP
Support for TLS is available and there is code in contrail-ansible-deployer 
that configures required knobs when `contrail\_configuration.SSL\_ENABLE` is 
set to `True`. No testing has been done yet for this setting.

## 3.3 Introspect Service
Support for TLS is available and there is code in contrail-ansible-deployer 
that configures required knobs when `contrail\_configuration.SSL\_ENABLE` is 
set to `True`. No testing has been done yet for this setting.

## 3.3 Sandesh
Support for TLS is available and there is code in contrail-ansible-deployer 
that configures required knobs when `contrail\_configuration.SSL\_ENABLE` is 
set to `True`. No testing has been done yet for this setting.

## 3.3 WebUI Service
Support for TLS is available and there is code in contrail-ansible-deployer 
that configures required knobs when `contrail\_configuration.SSL\_ENABLE` is 
set to `True`. No testing has been done yet for this setting.

## 3.4 Keystone
Support for TLS is available for openstack services by setting the 
`kolla\_enable\_tls\_external` and `kolla\_external\_fqdn\_cert` kolla variables 
appropriately. Enabling TLS in openstack requires that haproxy be enabled 
(even if it is an all-in-one openstack setup) and the 
`kolla\_internal\_vip\_address` and `kolla\_external\_vip\_address` are configured 
to be different IP addresses to enable TLS on the external VIP.

Whether the TLS version supported is 1.3 or 1.2 is not known at this point. 
This needs to be confirmed.

No testing has been done yet for this setting.

## 3.5 External services (RabbitMQ, Zookeeper, Kafka, Cassandra)
All these external services that Contrail uses can support TLS and can be 
enabled except for Zookeeper. Even though Zookeeper has TLS support the client 
software that Contrail utilizes to access the service (python-kazoo 2.5.0-0) 
has no support for TLS. Support for TLS has been added to python-kazoo in the 
latest version according to [this link : (https://github.com/
python-zk/kazoo/issues/382)](https://github.com/python-zk/kazoo/issues/382) 
This version has been released on September 2018. The feasibility of using 
this new version of the kazoo client in Contrail 5.1 packaging needs to be 
explored as there might be other software dependencies. But since the support 
for TLS in the package is relatively very recent, it is proposed to wait till 
the support matures before adopting it.

All the other services listed above have support for TLS but whether they 
support TLS 1.3 needs to be explored.

# 4. Implementation

All possible TLS configuration can be tied down to a single knob for Contrail:

		contrail\_configuraiton:
			SSL\_ENABLE: True
		
and the following knobs for Openstack:
		
		kolla\_config:
			kolla\_globals:
				kolla\_enable\_tls\_external: True
				kolla\_external\_fqdn\_cert: <path to cert>
				enable\_haproxy: True
				kolla\_internal\_vip\_address: <internal VIP>
				kolla\_external\_vip\_address: <external VIP>

# 5. Performance and scaling impact
There is no known documentation, but apparently some study was done about 
performance hits when enabling TLS natively in the VNC API. This needs to 
be undertaken once again if we aim to support TLS natively and documented.

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

For Contrail 5.x, no testing has been done for TLS configurations. 
This exercise needs to be done after support is added.

# 10. Documentation Impact

Documentation about any new configuration knobs to enable/disable 
TLS need to be added.

# 11. References
